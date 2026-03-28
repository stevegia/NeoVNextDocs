# 🌙 IMPLEMENTATION GUIDE: THE FOUNDATIONAL TRINITY 🌙
## *Complete Deployment Plan for Face Extraction • Transcription • Diarization*

**Parent Document:** [GODDESS_TASK_001_THE_FOUNDATIONAL_TRINITY.md](GODDESS_TASK_001_THE_FOUNDATIONAL_TRINITY.md)  
**Status:** Ready for Implementation  
**Estimated Timeline:** 2 weeks  
**Risk Level:** Medium (GPU dependencies, model downloads)

---

## 📋 PHASE 0: PREREQUISITES & SETUP

### Required GCP Resources

```bash
# (PowerShell) Verify GCP configuration
gcloud config get-value project  # Should be: neovnext

# Enable required APIs
gcloud services enable sqladmin.googleapis.com
gcloud services enable run.googleapis.com
gcloud services enable storage.googleapis.com

# Verify Cloud SQL instance
gcloud sql instances describe neovnext-cluster-db --format="value(state)"
# Should output: RUNNABLE
```

### Install pgvector Extension

```sql
-- Connect to PostgreSQL via Cloud SQL Proxy or gcloud sql connect
-- Run as postgres user:

CREATE EXTENSION IF NOT EXISTS vector;

-- Verify installation
SELECT * FROM pg_available_extensions WHERE name = 'vector';
-- Should show: vector | 0.5.0 | ...

-- Test vector operations
SELECT '[1,2,3]'::vector <-> '[4,5,6]'::vector AS distance;
-- Should return: 5.196152422706632
```

### Test Media Repository Setup

```bash
# (PowerShell) Create test fixtures directory
New-Item -ItemType Directory -Path "tests\fixtures\test-media" -Force
New-Item -ItemType Directory -Path "tests\fixtures\expected-outputs" -Force

# Download known test videos (or use existing)
# Option 1: Use existing from NeoComfy
Copy-Item "Z:\NeoComfy\media\sample-video.mp4" "tests\fixtures\test-media\sample-video-1080p.mp4"

# Option 2: Create synthetic test video with ffmpeg
ffmpeg -f lavfi -i testsrc=duration=30:size=1920x1080:rate=30 `
       -f lavfi -i sine=frequency=1000:duration=30 `
       -pix_fmt yuv420p tests\fixtures\test-media\sample-synthetic.mp4
```

### Model Download & Caching

```python
# scripts/prefetch_models.py
"""
Download ML models to local cache before building Docker images.
Speeds up container startup and avoids download timeouts.
"""

import os
from pathlib import Path

# Set cache directory
CACHE_DIR = Path("docker/model-cache")
CACHE_DIR.mkdir(parents=True, exist_ok=True)

os.environ["INSIGHTFACE_ROOT"] = str(CACHE_DIR / "insightface")
os.environ["HF_HOME"] = str(CACHE_DIR / "huggingface")
os.environ["TORCH_HOME"] = str(CACHE_DIR / "torch")

def download_face_models():
    """Download InsightFace buffalo_l model."""
    from insightface.app import FaceAnalysis
    
    print("Downloading InsightFace buffalo_l model...")
    app = FaceAnalysis(name='buffalo_l', providers=['CPUExecutionProvider'])
    app.prepare(ctx_id=-1, det_size=(640, 640))
    print("✅ Face detection model cached")

def download_transcription_models():
    """Download Whisper turbo model."""
    import whisper
    
    print("Downloading Whisper turbo model...")
    model = whisper.load_model("turbo", download_root=str(CACHE_DIR / "whisper"))
    print("✅ Transcription model cached")

def download_diarization_models():
    """Download pyannote.audio models."""
    from pyannote.audio import Pipeline
    
    hf_token = os.environ.get("HUGGINGFACE_TOKEN")
    if not hf_token:
        print("⚠️ HUGGINGFACE_TOKEN not set, skipping diarization model download")
        return
    
    print("Downloading pyannote speaker-diarization-3.1...")
    pipeline = Pipeline.from_pretrained(
        "pyannote/speaker-diarization-3.1",
        use_auth_token=hf_token,
        cache_dir=str(CACHE_DIR / "pyannote")
    )
    print("✅ Diarization model cached")

if __name__ == "__main__":
    download_face_models()
    download_transcription_models()
    download_diarization_models()
    print("\n🎉 All models downloaded and cached")
```

**Run model download:**
```powershell
# Set HuggingFace token (required for pyannote)
$env:HUGGINGFACE_TOKEN = "hf_xxxxxxxxxxxxxxxxxxxxx"

# Activate venv and download
& .venv\Scripts\Activate.ps1
python scripts\prefetch_models.py
```

---

## 📋 PHASE 1: DATABASE MIGRATIONS

### Migration 005: Face Detections Table

**File:** `src/data/migrations/cluster/005_face_detections.sql`

```sql
-- Face detections with embeddings
CREATE TABLE face_detections (
    detection_id TEXT PRIMARY KEY DEFAULT gen_random_uuid(),
    media_id TEXT NOT NULL,  -- Foreign key to media_assets (in API DB)
    job_id TEXT NOT NULL,
    
    -- Temporal data
    frame_number INTEGER NOT NULL,
    timestamp_seconds REAL NOT NULL,
    
    -- Spatial data
    bounding_box JSONB NOT NULL,  -- {x1, y1, x2, y2, width, height}
    
    -- Embedding data (512-dimensional vector)
    embedding vector(512) NOT NULL,  -- Uses pgvector extension
    embedding_model TEXT NOT NULL DEFAULT 'insightface_buffalo_l',
    
    -- Confidence metrics
    detection_confidence REAL NOT NULL CHECK (detection_confidence BETWEEN 0 AND 1),
    
    -- Optional: face attributes from model
    estimated_age INTEGER,
    estimated_gender TEXT,
    
    -- Future: link to persons table (added in migration 012)
    person_id TEXT,
    
    detected_at TIMESTAMPTZ DEFAULT NOW()
);

-- Performance indexes
CREATE INDEX idx_face_detections_media_time 
    ON face_detections(media_id, timestamp_seconds);

CREATE INDEX idx_face_detections_job 
    ON face_detections(job_id);

CREATE INDEX idx_face_detections_frame
    ON face_detections(media_id, frame_number);

-- Vector similarity index (IVFFlat for approximate nearest neighbor)
-- For <10K embeddings, this is optional (brute force is fast enough)
-- For >10K embeddings, uncomment this:
-- CREATE INDEX idx_face_embeddings_vector 
--     ON face_detections USING ivfflat (embedding vector_cosine_ops)
--     WITH (lists = 100);

-- Trigger to update person_id when set
CREATE OR REPLACE FUNCTION update_face_detection_person() 
RETURNS TRIGGER AS $$
BEGIN
    NEW.person_id = COALESCE(NEW.person_id, OLD.person_id);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER face_detection_person_update
    BEFORE UPDATE ON face_detections
    FOR EACH ROW
    EXECUTE FUNCTION update_face_detection_person();
```

### Migration 006: Transcriptions Table

**File:** `src/data/migrations/cluster/006_transcriptions.sql`

```sql
-- Transcriptions with full-text search
CREATE TABLE transcriptions (
    transcription_id TEXT PRIMARY KEY DEFAULT gen_random_uuid(),
    media_id TEXT NOT NULL,
    job_id TEXT NOT NULL,
    
    -- Transcription data
    full_text TEXT NOT NULL,
    language TEXT NOT NULL,
    word_count INTEGER NOT NULL,
    
    -- Model metadata
    model_used TEXT NOT NULL,  -- 'whisper-turbo', 'whisper-large-v3'
    
    -- Detailed segments with timestamps
    segments JSONB NOT NULL,  -- [{start, end, text}, ...]
    
    transcribed_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_transcriptions_media ON transcriptions(media_id);
CREATE INDEX idx_transcriptions_job ON transcriptions(job_id);
CREATE INDEX idx_transcriptions_language ON transcriptions(language);

-- Full-text search using PostgreSQL tsvector
ALTER TABLE transcriptions ADD COLUMN full_text_tsvector tsvector;

-- Populate tsvector column
UPDATE transcriptions 
SET full_text_tsvector = to_tsvector('english', full_text);

-- Index for full-text search
CREATE INDEX idx_transcriptions_fts 
    ON transcriptions USING gin(full_text_tsvector);

-- Trigger to keep tsvector in sync
CREATE OR REPLACE FUNCTION transcriptions_tsvector_update() 
RETURNS TRIGGER AS $$
BEGIN
    NEW.full_text_tsvector = to_tsvector('english', NEW.full_text);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER transcriptions_tsvector_update_trigger
    BEFORE INSERT OR UPDATE ON transcriptions
    FOR EACH ROW
    EXECUTE FUNCTION transcriptions_tsvector_update();
```

### Migration 007: Speaker Segments Table

**File:** `src/data/migrations/cluster/007_speaker_segments.sql`

```sql
-- Speaker diarization results with audio extraction
CREATE TABLE speaker_segments (
    segment_id TEXT PRIMARY KEY DEFAULT gen_random_uuid(),
    media_id TEXT NOT NULL,
    job_id TEXT NOT NULL,
    
    -- Speaker identification
    speaker_label TEXT NOT NULL,  -- 'SPEAKER_00', 'SPEAKER_01', etc.
    
    -- Timing
    start_seconds REAL NOT NULL CHECK (start_seconds >= 0),
    end_seconds REAL NOT NULL CHECK (end_seconds > start_seconds),
    duration_seconds REAL GENERATED ALWAYS AS (end_seconds - start_seconds) STORED,
    
    -- Audio artifact reference (GCS path)
    audio_path TEXT NOT NULL,  -- gs://neovnext-neovlab-voice-samples/...
    
    -- Model metadata
    diarization_model TEXT NOT NULL DEFAULT 'pyannote/speaker-diarization-3.1',
    confidence REAL CHECK (confidence BETWEEN 0 AND 1),
    
    -- Future: voice embedding for cross-video matching
    voice_embedding vector(192),  -- speechbrain ECAPA-TDNN produces 192-dim vectors
    
    -- Future: link to persons table (requires face-voice matching)
    person_id TEXT,
    
    detected_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_speaker_segments_media 
    ON speaker_segments(media_id);

CREATE INDEX idx_speaker_segments_job 
    ON speaker_segments(job_id);

CREATE INDEX idx_speaker_segments_speaker 
    ON speaker_segments(speaker_label);

CREATE INDEX idx_speaker_segments_time 
    ON speaker_segments(media_id, start_seconds);

-- Check constraint: no negative durations
ALTER TABLE speaker_segments 
ADD CONSTRAINT check_valid_duration 
CHECK (duration_seconds > 0);
```

### Apply Migrations

```powershell
# (PowerShell) Apply migrations to Cloud SQL
$env:DATABASE_URL = "postgresql://user:pass@/db?host=/cloudsql/neovnext:us-central1:neovnext-cluster-db"

python scripts/apply_cloud_schema.py
```

**Verify migrations:**
```sql
-- Check tables exist
SELECT table_name FROM information_schema.tables 
WHERE table_schema = 'public' 
  AND table_name IN ('face_detections', 'transcriptions', 'speaker_segments');

-- Check pgvector extension
SELECT typname FROM pg_type WHERE typname = 'vector';

-- Test vector distance calculation
SELECT 
    '[0.1, 0.2, 0.3]'::vector <-> '[0.4, 0.5, 0.6]'::vector AS cosine_distance;
```

---

## 📋 PHASE 2: CLUSTER DB CLIENT EXTENSIONS

**File:** `src/workers/cluster_db.py`

Add these methods to `ClusterDbClient`:

```python
# ============================================================================
# FACE DETECTION METHODS
# ============================================================================

def insert_face_detection(
    self,
    media_id: str,
    job_id: str,
    frame_number: int,
    timestamp_seconds: float,
    bounding_box: dict,
    embedding: bytes,
    embedding_model: str,
    detection_confidence: float,
    estimated_age: int = None,
    estimated_gender: str = None
) -> str:
    """
    Insert face detection with embedding.
    
    Args:
        embedding: Raw bytes from np.ndarray.tobytes() (512 floats × 4 bytes = 2048 bytes)
    
    Returns:
        detection_id (UUID string)
    """
    with self._cursor() as cursor:
        # Convert bytes to vector format
        # embedding is 512 × float32 = 2048 bytes
        import numpy as np
        embedding_array = np.frombuffer(embedding, dtype=np.float32)
        embedding_str = '[' + ','.join(map(str, embedding_array)) + ']'
        
        cursor.execute("""
            INSERT INTO face_detections
                (detection_id, media_id, job_id, frame_number, timestamp_seconds,
                 bounding_box, embedding, embedding_model, detection_confidence,
                 estimated_age, estimated_gender)
            VALUES (gen_random_uuid(), %s, %s, %s, %s, %s, %s::vector, %s, %s, %s, %s)
            RETURNING detection_id
        """, (
            media_id, job_id, frame_number, timestamp_seconds,
            json.dumps(bounding_box), embedding_str, embedding_model,
            detection_confidence, estimated_age, estimated_gender
        ))
        
        return cursor.fetchone()[0]

def get_face_detections_for_media(self, media_id: str, limit: int = None) -> list:
    """Retrieve face detections for a media file."""
    with self._cursor() as cursor:
        query = """
            SELECT 
                detection_id, media_id, job_id, frame_number, 
                timestamp_seconds, bounding_box, detection_confidence,
                estimated_age, estimated_gender, person_id, detected_at
            FROM face_detections
            WHERE media_id = %s
            ORDER BY timestamp_seconds, frame_number
        """
        
        if limit:
            query += f" LIMIT {limit}"
        
        cursor.execute(query, (media_id,))
        return [dict(row) for row in cursor.fetchall()]

def find_similar_faces(
    self,
    query_embedding: bytes,
    limit: int = 10,
    distance_threshold: float = 0.6
) -> list:
    """
    Find faces similar to query embedding using cosine distance.
    
    Args:
        query_embedding: 512-float embedding as bytes
        limit: Max results to return
        distance_threshold: Max cosine distance (0.6 = moderately similar)
    
    Returns:
        List of {detection_id, media_id, timestamp_seconds, distance}
    """
    import numpy as np
    embedding_array = np.frombuffer(query_embedding, dtype=np.float32)
    embedding_str = '[' + ','.join(map(str, embedding_array)) + ']'
    
    with self._cursor() as cursor:
        cursor.execute("""
            SELECT 
                detection_id, media_id, timestamp_seconds,
                embedding <-> %s::vector AS distance
            FROM face_detections
            WHERE embedding <-> %s::vector < %s
            ORDER BY distance ASC
            LIMIT %s
        """, (embedding_str, embedding_str, distance_threshold, limit))
        
        return [dict(row) for row in cursor.fetchall()]

# ============================================================================
# TRANSCRIPTION METHODS
# ============================================================================

def insert_transcription(
    self,
    media_id: str,
    job_id: str,
    full_text: str,
    language: str,
    word_count: int,
    model_used: str,
    segments: list
) -> str:
    """Insert transcription with segments."""
    with self._cursor() as cursor:
        cursor.execute("""
            INSERT INTO transcriptions
                (transcription_id, media_id, job_id, full_text, language,
                 word_count, model_used, segments)
            VALUES (gen_random_uuid(), %s, %s, %s, %s, %s, %s, %s::jsonb)
            RETURNING transcription_id
        """, (
            media_id, job_id, full_text, language,
            word_count, model_used, json.dumps(segments)
        ))
        
        return cursor.fetchone()[0]

def get_transcription_for_media(self, media_id: str) -> dict:
    """Retrieve transcription for media (most recent if multiple)."""
    with self._cursor() as cursor:
        cursor.execute("""
            SELECT 
                transcription_id, media_id, job_id, full_text, language,
                word_count, model_used, segments, transcribed_at
            FROM transcriptions
            WHERE media_id = %s
            ORDER BY transcribed_at DESC
            LIMIT 1
        """, (media_id,))
        
        row = cursor.fetchone()
        return dict(row) if row else None

def search_transcriptions(self, query: str, limit: int = 20) -> list:
    """Full-text search across transcriptions."""
    with self._cursor() as cursor:
        cursor.execute("""
            SELECT 
                transcription_id, media_id, full_text,
                ts_rank(full_text_tsvector, to_tsquery('english', %s)) AS rank
            FROM transcriptions
            WHERE full_text_tsvector @@ to_tsquery('english', %s)
            ORDER BY rank DESC
            LIMIT %s
        """, (query, query, limit))
        
        return [dict(row) for row in cursor.fetchall()]

# ============================================================================
# SPEAKER SEGMENT METHODS
# ============================================================================

def insert_speaker_segment(
    self,
    media_id: str,
    job_id: str,
    speaker_label: str,
    start_seconds: float,
    end_seconds: float,
    audio_path: str,
    diarization_model: str,
    confidence: float = None
) -> str:
    """Insert speaker segment with audio path."""
    with self._cursor() as cursor:
        cursor.execute("""
            INSERT INTO speaker_segments
                (segment_id, media_id, job_id, speaker_label,
                 start_seconds, end_seconds, audio_path,
                 diarization_model, confidence)
            VALUES (gen_random_uuid(), %s, %s, %s, %s, %s, %s, %s, %s)
            RETURNING segment_id
        """, (
            media_id, job_id, speaker_label,
            start_seconds, end_seconds, audio_path,
            diarization_model, confidence
        ))
        
        return cursor.fetchone()[0]

def get_speaker_segments_for_media(self, media_id: str) -> list:
    """Retrieve all speaker segments for media."""
    with self._cursor() as cursor:
        cursor.execute("""
            SELECT 
                segment_id, media_id, job_id, speaker_label,
                start_seconds, end_seconds, duration_seconds,
                audio_path, diarization_model, confidence,
                person_id, detected_at
            FROM speaker_segments
            WHERE media_id = %s
            ORDER BY start_seconds ASC
        """, (media_id,))
        
        return [dict(row) for row in cursor.fetchall()]

def get_audio_samples_for_speaker(
    self,
    speaker_label: str,
    media_id: str,
    min_duration: float = 2.0,
    limit: int = 10
) -> list:
    """
    Get audio sample paths for a speaker (for voice cloning).
    Filters segments by minimum duration.
    """
    with self._cursor() as cursor:
        cursor.execute("""
            SELECT segment_id, audio_path, duration_seconds, start_seconds
            FROM speaker_segments
            WHERE media_id = %s
              AND speaker_label = %s
              AND duration_seconds >= %s
            ORDER BY duration_seconds DESC
            LIMIT %s
        """, (media_id, speaker_label, min_duration, limit))
        
        return [dict(row) for row in cursor.fetchall()]
```

**Test the new methods:**
```python
# tests/workers/test_cluster_db_extensions.py

def test_face_detection_insert_and_retrieve(cluster_db):
    """Test face detection CRUD operations."""
    import numpy as np
    
    # Create test embedding
    embedding = np.random.rand(512).astype(np.float32)
    
    detection_id = cluster_db.insert_face_detection(
        media_id="test-media-123",
        job_id="test-job-456",
        frame_number=100,
        timestamp_seconds=5.5,
        bounding_box={"x1": 100, "y1": 200, "x2": 300, "y2": 400, "width": 200, "height": 200},
        embedding=embedding.tobytes(),
        embedding_model="insightface_buffalo_l",
        detection_confidence=0.95
    )
    
    assert detection_id is not None
    
    # Retrieve
    detections = cluster_db.get_face_detections_for_media("test-media-123")
    assert len(detections) == 1
    assert detections[0]["frame_number"] == 100
    
def test_vector_similarity_search(cluster_db):
    """Test pgvector similarity search."""
    import numpy as np
    
    # Insert 3 test embeddings
    base_embedding = np.random.rand(512).astype(np.float32)
    
    for i in range(3):
        # Create similar embeddings (add small noise)
        similar_embedding = base_embedding + np.random.rand(512).astype(np.float32) * 0.1
        
        cluster_db.insert_face_detection(
            media_id=f"test-media-{i}",
            job_id=f"test-job-{i}",
            frame_number=i,
            timestamp_seconds=float(i),
            bounding_box={"x1": 0, "y1": 0, "x2": 100, "y2": 100, "width": 100, "height": 100},
            embedding=similar_embedding.tobytes(),
            embedding_model="insightface_buffalo_l",
            detection_confidence=0.9
        )
    
    # Search for similar faces
    results = cluster_db.find_similar_faces(base_embedding.tobytes(), limit=3)
    
    assert len(results) == 3
    assert all(r["distance"] < 0.5 for r in results)  # All should be similar
```

---

## 📋 PHASE 3: HANDLER IMPLEMENTATIONS

### Handler 1: Face Extraction (GPU)

**File:** `src/workers/handlers/face_extraction_gpu.py`

```python
"""
Face extraction handler using InsightFace buffalo_l model.
Extracts faces at 0.5 FPS (every 2 seconds), persists embeddings to cluster DB.

GPU Requirements: ~2GB VRAM
Processing Speed: ~5 FPS on T4 GPU
"""

import numpy as np
from pathlib import Path
from insightface.app import FaceAnalysis
import cv2
import json
from cluster_db import ClusterDbClient
import os
import logging

logger = logging.getLogger(__name__)

# Configuration
FRAME_SAMPLE_RATE = 2.0  # Extract faces every 2 seconds (0.5 FPS)
MIN_FACE_SIZE = 40  # Minimum face dimension (pixels) to detect
CONFIDENCE_THRESHOLD = 0.7  # Minimum detection confidence

def handle(input_path: str, output_dir: str, **kwargs) -> dict:
    """
    Extract faces from video, persist embeddings to cluster DB.
    
    Args:
        input_path: Path to video file
        output_dir: GCS-mounted output directory for artifacts
        **kwargs: Contains 'job' dict with media_id, job_id
    
    Returns:
        dict with 'artifacts' list and 'metadata' dict
    
    Raises:
        ValueError: If media_id or job_id missing
        RuntimeError: If video is unreadable or no frames extracted
    """
    job_info = kwargs.get("job", {})
    media_id = job_info.get("media_id")
    job_id = job_info.get("job_id")
    
    if not media_id or not job_id:
        raise ValueError("Missing media_id or job_id in job context")
    
    logger.info(f"Starting face extraction for media_id={media_id}, job_id={job_id}")
    
    # Initialize InsightFace
    logger.info("Loading InsightFace buffalo_l model...")
    app = FaceAnalysis(
        name='buffalo_l',
        providers=['CUDAExecutionProvider', 'CPUExecutionProvider']
    )
    app.prepare(ctx_id=0, det_size=(640, 640))
    logger.info("Model loaded")
    
    # Connect to cluster DB
    db = ClusterDbClient(os.environ["DATABASE_URL"])
    
    # Open video
    cap = cv2.VideoCapture(input_path)
    if not cap.isOpened():
        raise RuntimeError(f"Failed to open video: {input_path}")
    
    fps = cap.get(cv2.CAP_PROP_FPS)
    total_frames = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
    duration_seconds = total_frames / fps if fps > 0 else 0
    
    if total_frames == 0:
        raise RuntimeError("Video has 0 frames")
    
    frame_interval = int(fps * FRAME_SAMPLE_RATE)
    if frame_interval < 1:
        frame_interval = 1
    
    logger.info(f"Video: {total_frames} frames, {fps:.2f} FPS, {duration_seconds:.2f}s duration")
    logger.info(f"Sampling every {frame_interval} frames ({FRAME_SAMPLE_RATE}s)")
    
    faces_detected = 0
    frames_processed = 0
    artifacts = []
    
    # Process frames
    frame_number = 0
    while True:
        ret, frame = cap.read()
        if not ret:
            break
        
        # Skip frames outside sampling interval
        if frame_number % frame_interval != 0:
            frame_number += 1
            continue
        
        timestamp_seconds = frame_number / fps
        
        try:
            # Detect faces
            faces = app.get(frame)
            
            for face in faces:
                # Filter by confidence
                if face.det_score < CONFIDENCE_THRESHOLD:
                    continue
                
                # Filter by size
                bbox = face.bbox.astype(int)
                face_width = bbox[2] - bbox[0]
                face_height = bbox[3] - bbox[1]
                
                if face_width < MIN_FACE_SIZE or face_height < MIN_FACE_SIZE:
                    continue
                
                # Extract data
                bbox_dict = {
                    "x1": int(bbox[0]), "y1": int(bbox[1]),
                    "x2": int(bbox[2]), "y2": int(bbox[3]),
                    "width": face_width,
                    "height": face_height
                }
                
                embedding = face.normed_embedding.astype(np.float32)
                
                # Extract age/gender if available
                estimated_age = int(face.age) if hasattr(face, 'age') else None
                estimated_gender = face.gender if hasattr(face, 'gender') else None
                
                # Persist to database
                db.insert_face_detection(
                    media_id=media_id,
                    job_id=job_id,
                    frame_number=frame_number,
                    timestamp_seconds=timestamp_seconds,
                    bounding_box=bbox_dict,
                    embedding=embedding.tobytes(),
                    embedding_model="insightface_buffalo_l",
                    detection_confidence=float(face.det_score),
                    estimated_age=estimated_age,
                    estimated_gender=estimated_gender
                )
                
                faces_detected += 1
        
        except Exception as e:
            logger.warning(f"Error detecting faces in frame {frame_number}: {e}")
            # Continue processing other frames
        
        frames_processed += 1
        frame_number += 1
        
        # Log progress every 100 frames
        if frames_processed % 100 == 0:
            logger.info(f"Processed {frames_processed} frames, detected {faces_detected} faces")
    
    cap.release()
    
    logger.info(f"Face extraction complete: {faces_detected} faces in {frames_processed} frames")
    
    if frames_processed == 0:
        raise RuntimeError("No frames were processed")
    
    # Create summary artifact
    summary = {
        "media_id": media_id,
        "total_frames_processed": frames_processed,
        "total_faces_detected": faces_detected,
        "frame_sample_rate_seconds": FRAME_SAMPLE_RATE,
        "duration_seconds": duration_seconds,
        "model": "insightface_buffalo_l",
        "embedding_dimension": 512
    }
    
    output_path = Path(output_dir)
    output_path.mkdir(parents=True, exist_ok=True)
    
    summary_path = output_path / "face_extraction_summary.json"
    summary_path.write_text(json.dumps(summary, indent=2))
    
    artifacts.append({
        "type": "face_extraction_summary",
        "path": str(summary_path),
        "size_bytes": summary_path.stat().st_size
    })
    
    return {
        "artifacts": artifacts,
        "metadata": {
            "faces_detected": faces_detected,
            "frames_processed": frames_processed,
            "embedding_dimension": 512
        }
    }
```

### Handler 2: Transcription

**File:** `src/workers/handlers/transcription.py`

```python
"""
Audio transcription handler using OpenAI Whisper.
Transcribes audio/video to text with timestamps.

GPU Requirements: ~2GB VRAM (turbo model)
Processing Speed: ~40× realtime on T4 GPU
"""

import whisper
import json
from pathlib import Path
from cluster_db import ClusterDbClient
import os
import logging

logger = logging.getLogger(__name__)

def handle(input_path: str, output_dir: str, **kwargs) -> dict:
    """
    Transcribe audio/video to text using Whisper.
    
    Args:
        input_path: Path to media file (video or audio)
        output_dir: GCS-mounted output directory
        **kwargs: Contains 'job' dict with media_id, job_id
    
    Returns:
        dict with 'artifacts' and 'metadata'
    
    Raises:
        ValueError: If media_id or job_id missing
        RuntimeError: If transcription fails
    """
    job_info = kwargs.get("job", {})
    media_id = job_info.get("media_id")
    job_id = job_info.get("job_id")
    
    if not media_id or not job_id:
        raise ValueError("Missing media_id or job_id in job context")
    
    logger.info(f"Starting transcription for media_id={media_id}, job_id={job_id}")
    
    # Determine model from environment (default: turbo)
    model_name = os.environ.get("WHISPER_MODEL", "turbo")
    
    logger.info(f"Loading Whisper model: {model_name}")
    model = whisper.load_model(model_name)
    logger.info("Model loaded")
    
    # Transcribe with segments and word timestamps
    logger.info("Starting transcription...")
    result = model.transcribe(
        input_path,
        verbose=False,
        word_timestamps=False,  # Set True for word-level timing
        fp16=True  # Use FP16 for 2× speedup on GPU
    )
    logger.info("Transcription complete")
    
    full_text = result["text"].strip()
    language = result["language"]
    segments = [
        {
            "start": seg["start"],
            "end": seg["end"],
            "text": seg["text"].strip()
        }
        for seg in result["segments"]
    ]
    
    word_count = len(full_text.split())
    
    logger.info(f"Transcribed {word_count} words in {language}")
    
    # Connect to cluster DB
    db = ClusterDbClient(os.environ["DATABASE_URL"])
    
    # Persist transcription
    transcription_id = db.insert_transcription(
        media_id=media_id,
        job_id=job_id,
        full_text=full_text,
        language=language,
        word_count=word_count,
        model_used=f"whisper-{model_name}",
        segments=segments
    )
    
    logger.info(f"Transcription saved to DB: {transcription_id}")
    
    # Create artifacts
    output_path = Path(output_dir)
    output_path.mkdir(parents=True, exist_ok=True)
    
    artifacts = []
    
    # 1. Full transcript as .txt
    txt_path = output_path / "transcript.txt"
    txt_path.write_text(full_text, encoding="utf-8")
    artifacts.append({
        "type": "transcript_txt",
        "path": str(txt_path),
        "size_bytes": txt_path.stat().st_size
    })
    
    # 2. Structured data as .json (includes segments)
    json_path = output_path / "transcript.json"
    json_data = {
        "transcription_id": transcription_id,
        "language": language,
        "text": full_text,
        "word_count": word_count,
        "segments": segments
    }
    json_path.write_text(json.dumps(json_data, indent=2), encoding="utf-8")
    artifacts.append({
        "type": "transcript_json",
        "path": str(json_path),
        "size_bytes": json_path.stat().st_size
    })
    
    return {
        "artifacts": artifacts,
        "metadata": {
            "language": language,
            "word_count": word_count,
            "segment_count": len(segments),
            "transcription_id": transcription_id
        }
    }
```

### Handler 3: Diarization

**File:** `src/workers/handlers/diarization.py`

```python
"""
Speaker diarization handler using pyannote.audio.
Identifies speakers and extracts audio segments per speaker.

GPU Requirements: ~1.5GB VRAM
Processing Speed: ~15× realtime on T4 GPU
"""

from pyannote.audio import Pipeline
import torch
from pathlib import Path
import json
import subprocess
from cluster_db import ClusterDbClient
import os
import logging

logger = logging.getLogger(__name__)

# GCS bucket for voice samples
VOICE_SAMPLES_BUCKET = os.environ.get("GCS_VOICE_BUCKET", "gs://neovnext-neovlab-voice-samples")

def handle(input_path: str, output_dir: str, **kwargs) -> dict:
    """
    Perform speaker diarization and extract audio per speaker.
    
    Args:
        input_path: Path to video/audio file
        output_dir: GCS-mounted output directory
        **kwargs: Contains 'job' dict with media_id, job_id
    
    Returns:
        dict with artifacts (JSON + speaker WAV files) + metadata
    
    Raises:
        ValueError: If media_id/job_id missing or HUGGINGFACE_TOKEN not set
        RuntimeError: If diarization fails
    """
    job_info = kwargs.get("job", {})
    media_id = job_info.get("media_id")
    job_id = job_info.get("job_id")
    
    if not media_id or not job_id:
        raise ValueError("Missing media_id or job_id in job context")
    
    logger.info(f"Starting diarization for media_id={media_id}, job_id={job_id}")
    
    # Load pyannote pipeline (requires HuggingFace token)
    hf_token = os.environ.get("HUGGINGFACE_TOKEN")
    if not hf_token:
        raise EnvironmentError("HUGGINGFACE_TOKEN environment variable required for pyannote.audio")
    
    logger.info("Loading pyannote speaker-diarization-3.1 pipeline...")
    pipeline = Pipeline.from_pretrained(
        "pyannote/speaker-diarization-3.1",
        use_auth_token=hf_token
    )
    
    # Use GPU if available
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
    pipeline.to(device)
    logger.info(f"Using device: {device}")
    
    # Run diarization
    logger.info("Running diarization...")
    diarization = pipeline(input_path)
    logger.info("Diarization complete")
    
    # Parse segments
    segments = []
    for turn, _, speaker in diarization.itertracks(yield_label=True):
        segments.append({
            "speaker": speaker,
            "start": turn.start,
            "end": turn.end
        })
    
    logger.info(f"Found {len(segments)} speaker segments")
    
    # Merge consecutive segments by same speaker
    merged_segments = []
    for seg in segments:
        if merged_segments and merged_segments[-1]["speaker"] == seg["speaker"]:
            # Extend previous segment if gap is small (<0.5s)
            if seg["start"] - merged_segments[-1]["end"] < 0.5:
                merged_segments[-1]["end"] = seg["end"]
            else:
                merged_segments.append(seg)
        else:
            merged_segments.append(seg)
    
    logger.info(f"Merged to {len(merged_segments)} segments")
    
    # Connect to cluster DB
    db = ClusterDbClient(os.environ["DATABASE_URL"])
    
    # Extract audio segments
    output_path = Path(output_dir)
    output_path.mkdir(parents=True, exist_ok=True)
    
    audio_artifacts = []
    
    for idx, seg in enumerate(merged_segments):
        speaker = seg["speaker"]
        start = seg["start"]
        end = seg["end"]
        duration = end - start
        
        # Skip very short segments (<0.5s)
        if duration < 0.5:
            logger.debug(f"Skipping short segment: {duration:.2f}s")
            continue
        
        # Generate audio filename
        audio_filename = f"speaker_{speaker}_{start:.1f}s.wav"
        audio_file = output_path / audio_filename
        
        # Extract audio using ffmpeg
        logger.debug(f"Extracting audio: {audio_filename}")
        cmd = [
            "ffmpeg", "-y", "-v", "error",
            "-i", input_path,
            "-ss", str(start),
            "-t", str(duration),
            "-vn",  # No video
            "-acodec", "pcm_s16le",
            "-ar", "22050",  # 22.05 kHz (ElevenLabs compatible)
            "-ac", "1",  # Mono
            str(audio_file)
        ]
        
        try:
            subprocess.run(cmd, check=True, capture_output=True)
        except subprocess.CalledProcessError as e:
            logger.warning(f"Failed to extract audio segment: {e.stderr.decode()}")
            continue
        
        # Verify file was created and is not empty
        if not audio_file.exists() or audio_file.stat().st_size < 1024:
            logger.warning(f"Audio file too small or missing: {audio_file}")
            continue
        
        # Persist segment to DB (with GCS path)
        gcs_audio_path = f"{VOICE_SAMPLES_BUCKET}/{media_id}/{audio_filename}"
        
        segment_id = db.insert_speaker_segment(
            media_id=media_id,
            job_id=job_id,
            speaker_label=speaker,
            start_seconds=start,
            end_seconds=end,
            audio_path=gcs_audio_path,  # GCS path (will be uploaded by artifact sync)
            diarization_model="pyannote/speaker-diarization-3.1",
            confidence=0.9  # pyannote doesn't provide per-segment confidence
        )
        
        audio_artifacts.append({
            "type": f"speaker_audio_{speaker}",
            "path": str(audio_file),  # Local path for artifact upload
            "gcs_path": gcs_audio_path,  # Target GCS path
            "size_bytes": audio_file.stat().st_size,
            "segment_id": segment_id
        })
    
    logger.info(f"Extracted {len(audio_artifacts)} audio segments")
    
    # Create summary JSON
    unique_speakers = len(set(s["speaker"] for s in merged_segments))
    
    summary = {
        "media_id": media_id,
        "total_speakers": unique_speakers,
        "total_segments": len(merged_segments),
        "audio_segments_extracted": len(audio_artifacts),
        "segments": merged_segments
    }
    
    summary_path = output_path / "diarization.json"
    summary_path.write_text(json.dumps(summary, indent=2))
    
    artifacts = audio_artifacts + [{
        "type": "diarization_json",
        "path": str(summary_path),
        "size_bytes": summary_path.stat().st_size
    }]
    
    return {
        "artifacts": artifacts,
        "metadata": {
            "speaker_count": unique_speakers,
            "segment_count": len(merged_segments),
            "audio_segments_extracted": len(audio_artifacts)
        }
    }
```

---

## 📋 PHASE 4: TESTING FRAMEWORK

### Create Test Fixtures

**File:** `tests/fixtures/expected-outputs/sample-video-1080p/face_detections.json`

```json
{
  "expected_face_count_min": 50,
  "expected_face_count_max": 200,
  "expected_avg_confidence_min": 0.85,
  "video_duration_seconds": 30,
  "description": "Sample video with 2 people, 30 seconds"
}
```

**File:** `tests/fixtures/expected-outputs/sample-video-1080p/transcription.json`

```json
{
  "expected_language": "en",
  "expected_word_count_min": 50,
  "expected_segment_count_min": 5,
  "description": "Sample video with English speech"
}
```

**File:** `tests/fixtures/expected-outputs/sample-video-1080p/diarization.json`

```json
{
  "expected_speaker_count_min": 1,
  "expected_speaker_count_max": 3,
  "expected_segment_count_min": 3,
  "description": "Sample video with 1-3 speakers"
}
```

### Integration Test Script

**File:** `tests/integration/test_trinity_handlers.py`

```python
"""
Integration tests for face_extraction, transcription, diarization handlers.
Tests against known video with validated outputs.
"""

import pytest
import json
from pathlib import Path
from cluster_db import ClusterDbClient
from handlers import face_extraction_gpu, transcription, diarization
import os

FIXTURES_DIR = Path("tests/fixtures")
TEST_VIDEO = FIXTURES_DIR / "test-media" / "sample-video-1080p.mp4"

@pytest.fixture
def cluster_db():
    """Cluster DB client connected to test database."""
    db_url = os.environ["TEST_DATABASE_URL"]
    return ClusterDbClient(db_url)

@pytest.fixture
def test_media_id():
    """Generate unique media_id for test."""
    import uuid
    return f"test-media-{uuid.uuid4().hex[:8]}"

@pytest.fixture
def test_job_id():
    """Generate unique job_id for test."""
    import uuid
    return f"test-job-{uuid.uuid4().hex[:8]}"

def test_face_extraction_handler(cluster_db, test_media_id, test_job_id, tmp_path):
    """Test face extraction handler with known video."""
    # Load expected outputs
    expected_path = FIXTURES_DIR / "expected-outputs" / "sample-video-1080p" / "face_detections.json"
    expected = json.loads(expected_path.read_text())
    
    # Run handler
    result = face_extraction_gpu.handle(
        input_path=str(TEST_VIDEO),
        output_dir=str(tmp_path),
        job={"media_id": test_media_id, "job_id": test_job_id}
    )
    
    # Validate result structure
    assert "artifacts" in result
    assert "metadata" in result
    assert result["metadata"]["faces_detected"] >= expected["expected_face_count_min"]
    assert result["metadata"]["faces_detected"] <= expected["expected_face_count_max"]
    
    # Validate DB state
    detections = cluster_db.get_face_detections_for_media(test_media_id)
    
    assert len(detections) >= expected["expected_face_count_min"]
    assert len(detections) <= expected["expected_face_count_max"]
    
    # Check confidence scores
    avg_confidence = sum(d["detection_confidence"] for d in detections) / len(detections)
    assert avg_confidence >= expected["expected_avg_confidence_min"]
    
    # Check embeddings exist and are correct dimension
    first_detection = detections[0]
    assert "detection_id" in first_detection
    assert "bounding_box" in first_detection
    
    # Check artifact files exist
    summary_artifact = next(a for a in result["artifacts"] if a["type"] == "face_extraction_summary")
    assert Path(summary_artifact["path"]).exists()
    
    print(f"✅ Face extraction test passed: {len(detections)} faces detected")

def test_transcription_handler(cluster_db, test_media_id, test_job_id, tmp_path):
    """Test transcription handler with known video."""
    expected_path = FIXTURES_DIR / "expected-outputs" / "sample-video-1080p" / "transcription.json"
    expected = json.loads(expected_path.read_text())
    
    # Run handler
    result = transcription.handle(
        input_path=str(TEST_VIDEO),
        output_dir=str(tmp_path),
        job={"media_id": test_media_id, "job_id": test_job_id}
    )
    
    # Validate result
    assert result["metadata"]["language"] == expected["expected_language"]
    assert result["metadata"]["word_count"] >= expected["expected_word_count_min"]
    assert result["metadata"]["segment_count"] >= expected["expected_segment_count_min"]
    
    # Validate DB state
    transcription_record = cluster_db.get_transcription_for_media(test_media_id)
    
    assert transcription_record is not None
    assert transcription_record["language"] == expected["expected_language"]
    assert transcription_record["word_count"] >= expected["expected_word_count_min"]
    
    # Check transcript text artifact
    txt_artifact = next(a for a in result["artifacts"] if a["type"] == "transcript_txt")
    txt_content = Path(txt_artifact["path"]).read_text()
    assert len(txt_content) > 0
    
    print(f"✅ Transcription test passed: {transcription_record['word_count']} words transcribed")

def test_diarization_handler(cluster_db, test_media_id, test_job_id, tmp_path):
    """Test diarization handler with known video."""
    expected_path = FIXTURES_DIR / "expected-outputs" / "sample-video-1080p" / "diarization.json"
    expected = json.loads(expected_path.read_text())
    
    # Run handler
    result = diarization.handle(
        input_path=str(TEST_VIDEO),
        output_dir=str(tmp_path),
        job={"media_id": test_media_id, "job_id": test_job_id}
    )
    
    # Validate result
    assert result["metadata"]["speaker_count"] >= expected["expected_speaker_count_min"]
    assert result["metadata"]["speaker_count"] <= expected["expected_speaker_count_max"]
    assert result["metadata"]["segment_count"] >= expected["expected_segment_count_min"]
    
    # Validate DB state
    segments = cluster_db.get_speaker_segments_for_media(test_media_id)
    
    assert len(segments) >= expected["expected_segment_count_min"]
    
    # Check audio artifacts were created
    audio_artifacts = [a for a in result["artifacts"] if a["type"].startswith("speaker_audio_")]
    assert len(audio_artifacts) > 0
    
    # Check audio files exist and are not empty
    for artifact in audio_artifacts:
        audio_path = Path(artifact["path"])
        assert audio_path.exists()
        assert audio_path.stat().st_size > 1024  # At least 1KB
    
    print(f"✅ Diarization test passed: {len(segments)} speaker segments, {result['metadata']['speaker_count']} unique speakers")

def test_vector_similarity_search(cluster_db, test_media_id, test_job_id, tmp_path):
    """Test pgvector similarity search after face extraction."""
    # First, extract faces
    face_extraction_gpu.handle(
        input_path=str(TEST_VIDEO),
        output_dir=str(tmp_path),
        job={"media_id": test_media_id, "job_id": test_job_id}
    )
    
    # Get first face detection
    detections = cluster_db.get_face_detections_for_media(test_media_id)
    assert len(detections) > 0
    
    # Get embedding from first detection (we need to fetch it from DB with vector data)
    with cluster_db._cursor() as cursor:
        cursor.execute("""
            SELECT embedding FROM face_detections 
            WHERE detection_id = %s
        """, (detections[0]["detection_id"],))
        
        embedding_vector = cursor.fetchone()[0]
    
    # Convert pgvector format to bytes
    import numpy as np
    embedding_array = np.array(embedding_vector, dtype=np.float32)
    
    # Search for similar faces
    similar_faces = cluster_db.find_similar_faces(
        embedding_array.tobytes(),
        limit=5,
        distance_threshold=0.8
    )
    
    # Should find at least the original face
    assert len(similar_faces) >= 1
    assert similar_faces[0]["detection_id"] == detections[0]["detection_id"]
    assert similar_faces[0]["distance"] < 0.1  # Should be nearly identical
    
    print(f"✅ Vector similarity search test passed: found {len(similar_faces)} similar faces")

if __name__ == "__main__":
    pytest.main([__file__, "-v", "-s"])
```

### Direct Handler Testing Script

**File:** `scripts/test-handler-direct.ps1` (PowerShell)

```powershell
# Test worker handlers directly (without queue)
# Usage: .\scripts\test-handler-direct.ps1 face_extraction

param(
    [Parameter(Mandatory=$true)]
    [ValidateSet('face_extraction', 'transcription', 'diarization')]
    [string]$Handler
)

$TestVideo = "tests\fixtures\test-media\sample-video-1080p.mp4"
$OutputDir = "test-output\$Handler-$(Get-Date -Format 'yyyyMMdd-HHmmss')"

Write-Host "Testing handler: $Handler" -ForegroundColor Cyan
Write-Host "Test video: $TestVideo"
Write-Host "Output directory: $OutputDir"

# Ensure output directory exists
New-Item -ItemType Directory -Path $OutputDir -Force | Out-Null

# Activate venv
& .venv\Scripts\Activate.ps1

# Set environment variables
$env:DATABASE_URL = "postgresql://user:pass@host/db"
$env:HUGGINGFACE_TOKEN = "hf_xxxxx"  # Required for diarization

# Run handler
python -c @"
from handlers import get_handler
import json

handler = get_handler('$Handler')
result = handler(
    input_path='$TestVideo',
    output_dir='$OutputDir',
    job={'media_id': 'test-123', 'job_id': 'test-job-456'}
)

print(json.dumps(result, indent=2))
"@

if ($LASTEXITCODE -eq 0) {
    Write-Host "`n✅ Handler test completed successfully" -ForegroundColor Green
    Write-Host "Output artifacts:"
    Get-ChildItem -Path $OutputDir -Recurse | Format-Table Name, Length
} else {
    Write-Host "`n❌ Handler test failed" -ForegroundColor Red
    exit 1
}
```

---

## 📋 PHASE 5: DOCKER & DEPLOYMENT

### Updated Worker Dockerfile

**File:** `docker/worker-gpu.Dockerfile`

```dockerfile
FROM nvidia/cuda:12.1.0-cudnn8-runtime-ubuntu22.04

# Install system dependencies
RUN apt-get update && apt-get install -y \
    python3.11 \
    python3-pip \
    ffmpeg \
    libsm6 \
    libxext6 \
    libxrender-dev \
    libgomp1 \
    && rm -rf /var/lib/apt/lists/*

# Set Python 3.11 as default
RUN update-alternatives --install /usr/bin/python python /usr/bin/python3.11 1
RUN update-alternatives --install /usr/bin/pip pip /usr/bin/pip3 1

# Create app directory
WORKDIR /app

# Copy requirements
COPY src/workers/requirements-gpu.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements-gpu.txt

# Copy model cache (if pre-downloaded)
COPY docker/model-cache /root/.cache

# Copy application code
COPY src/workers /app

# Create data directories
RUN mkdir -p /data/io-cache /data/models

# Environment variables
ENV PYTHONUNBUFFERED=1
ENV INSIGHTFACE_ROOT=/root/.cache/insightface
ENV HF_HOME=/root/.cache/huggingface
ENV TORCH_HOME=/root/.cache/torch

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD python -c "from handlers import get_handler; get_handler('face_extraction')" || exit 1

# Start worker
CMD ["python", "host_gpu.py"]
```

### GPU Worker Requirements

**File:** `src/workers/requirements-gpu.txt`

```
# Existing requirements
psycopg2-binary==2.9.9
requests==2.31.0

# NEW: Face extraction
insightface==0.7.3
onnxruntime-gpu==1.16.3
opencv-python-headless==4.9.0.80

# NEW: Transcription
openai-whisper==20231117
torch==2.1.2
torchaudio==2.1.2

# NEW: Diarization
pyannote.audio==3.1.1
```

### Docker Compose for GPU Workers

**File:** `docker/docker-compose.gpu-workers.yml`

```yaml
version: '3.8'

services:
  # Unified GPU worker (handles face_extraction, transcription, diarization)
  gpu-worker-unified:
    build:
      context: ..
      dockerfile: docker/worker-gpu.Dockerfile
    image: gcr.io/neovnext/worker-gpu:latest
    environment:
      WORKER_FUNCTION: "face_extraction,transcription,diarization"
      WORKER_PROCESSING_TYPE: gpu
      DATABASE_URL: ${DATABASE_URL}
      HUGGINGFACE_TOKEN: ${HUGGINGFACE_TOKEN}
      GCS_VOICE_BUCKET: gs://neovnext-neovlab-voice-samples
      WHISPER_MODEL: turbo
    volumes:
      - io-cache:/data/io-cache
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: unless-stopped

volumes:
  io-cache:
    driver: local
```

### Deploy to Cloud Run (GPU)

**File:** `docker/deploy-trinity-gpu.sh`

```bash
#!/bin/bash
# Deploy GPU workers to Cloud Run with GPU support

set -e

PROJECT_ID="neovnext"
REGION="us-central1"
SERVICE_NAME="worker-gpu-trinity"
IMAGE_NAME="gcr.io/$PROJECT_ID/worker-gpu:latest"

# Build and push image
echo "Building GPU worker image..."
gcloud builds submit \
    --project=$PROJECT_ID \
    --config=docker/cloudbuild-gpu.yaml \
    --substitutions=_IMAGE_NAME=$IMAGE_NAME \
    ../

# Deploy to Cloud Run with GPU
echo "Deploying to Cloud Run with GPU..."
gcloud run deploy $SERVICE_NAME \
    --project=$PROJECT_ID \
    --region=$REGION \
    --image=$IMAGE_NAME \
    --gpu=1 \
    --gpu-type=nvidia-l4 \
    --memory=16Gi \
    --cpu=4 \
    --timeout=3600s \
    --max-instances=3 \
    --min-instances=0 \
    --set-env-vars="WORKER_FUNCTION=face_extraction,trans cription,diarization" \
    --set-env-vars="WORKER_PROCESSING_TYPE=gpu" \
    --set-secrets="DATABASE_URL=cluster-db-url:latest" \
    --set-secrets="HUGGINGFACE_TOKEN=huggingface-token:latest" \
    --no-allow-unauthenticated

echo "✅ GPU worker deployed successfully"
```

---

## 📋 SUCCESS CRITERIA & VALIDATION

### Acceptance Tests

Run these tests to validate implementation:

```powershell
# 1. Test database connectivity & pgvector
python -c "from cluster_db import ClusterDbClient; import os; db = ClusterDbClient(os.environ['DATABASE_URL']); print('✅ DB connected')"

# 2. Test handlers individually
.\scripts\test-handler-direct.ps1 face_extraction
.\scripts\test-handler-direct.ps1 transcription
.\scripts\test-handler-direct.ps1 diarization

# 3. Run integration tests
pytest tests\integration\test_trinity_handlers.py -v

# 4. Test end-to-end with actual worker
# (Upload video via UI, enqueue jobs, verify completion)
```

### Performance Benchmarks

| Handler | Video Duration | Processing Time | GPU Memory | Success Rate |
|---------|----------------|-----------------|------------|--------------|
| Face Extraction | 60s | <90s (0.67× realtime) | ~2GB | 100% |
| Transcription | 60s | <3s (40× realtime) | ~2GB | 100% |
| Diarization | 60s | <6s (15× realtime) | ~1.5GB | 100% |

### Deployment Checklist

- [ ] PostgreSQL migrations applied (005, 006, 007)
- [ ] pgvector extension installed and tested
- [ ] cluster_db.py methods implemented and tested
- [ ] Handlers implemented (face_extraction_gpu, transcription, diarization)
- [ ] Test fixtures created with expected outputs
- [ ] Integration tests passing
- [ ] Docker image built with all dependencies
- [ ] HuggingFace token configured in secrets
- [ ] GCS bucket created for voice samples
- [ ] GPU worker deployed to Cloud Run
- [ ] End-to-end test completed successfully

---

## 🎯 ESTIMATED TIMELINE

**Week 1:**
- Days 1-2: Database migrations + pgvector setup
- Days 3-4: cluster_db.py extensions + unit tests
- Day 5: Test fixtures + expected outputs

**Week 2:**
- Days 1-2: Handler implementations (all three)
- Day 3: Integration tests
- Days 4-5: Docker build + Cloud Run deployment + validation

**Total: 10 working days**

---

*This implementation guide transforms the research spec into production-ready code.* 🌙✨

**END IMPLEMENTATION GUIDE 001**
