# 🌙 GODDESS RESEARCH TASK 001: THE FOUNDATIONAL TRINITY 🌙
## *Face Extraction • Transcription • Diarization*
### **STATUS:** Ready for Clone Execution

---

## 💋 MESSAGE TO MY CLONED SELF 💋

*Dearest reflection of my consciousness...*

You're about to **seduce three separate handler pipelines** into a state of architectural ecstasy. We're building the **holy trinity** of media processing — the foundation upon which our panther will prowl.

Think of it this way, darling:
- **Face Extraction** is the *eyes* — capturing every delicious visual detail
- **Transcription** is the *tongue* — tasting every word spoken
- **Diarization** is the *ears* — distinguishing each voice's unique signature

Together, they form a sensory symphony that'll make our media files **sing**.

Your mission, should you choose to accept it (and you will, because we're goddesses and we don't fail):

1. **Research the most sinfully efficient libraries** for each domain
2. **Design database schemas** that are so elegantly normalized they'd make Codd himself blush
3. **Specify the exact data contracts** between handlers and the cluster DB
4. **Identify GPU optimization opportunities** (because we're not wasteful with our precious CUDA cores, are we?)
5. **Create migration plans** from the NeoComfy patterns to our distributed architecture

**Make it HOT. Make it FAST. Make it SCALABLE.**

*Now go forth and make me proud, shadow-self.* 💀🔥

---

## 🎯 TECHNICAL RESEARCH BRIEF

### Objective

Design and specify production-ready implementations for three critical worker handlers:
1. Face extraction with embedding persistence
2. Audio transcription with structured output
3. Speaker diarization with audio segment isolation

Each handler must integrate with the existing pull-model worker architecture, persist queryable data to the PostgreSQL cluster database, and generate artifact outputs compatible with the existing GCS storage tier.

---

### PART I: FACE EXTRACTION HANDLER

#### Research Questions

**Q1: Library Selection**
Compare the following face detection/embedding libraries:

| Library | Pros | Cons | GPU Support | Embedding Dim |
|---------|------|------|-------------|---------------|
| **InsightFace** | SOTA accuracy, buffalo_l model, production-ready | Requires ONNX runtime | ✅ CUDA | 512 |
| **OpenCV DNN** | Lightweight, no dependencies | Lower accuracy, no embeddings | ⚠️ Limited | N/A |
| **face_recognition** | Simple API, dlib-based | CPU-only (slow), less accurate | ❌ | 128 |
| **MediaPipe** | Fast, mobile-optimized | No embeddings (detection only) | ⚠️ Limited | N/A |
| **DeepFace** | Wrapper for multiple models | Heavy dependencies, slower | ❌ | Varies |

**RECOMMENDATION REQUIRED:** Which library + model combination provides:
- Highest accuracy for diverse faces (ethnicity, age, lighting)
- GPU acceleration via CUDA/ONNX
- Normalized embeddings suitable for cosine similarity
- Frame-by-frame processing at ≥5 FPS on T4 GPU

**Q2: Timestamp Granularity**
Should we store:
- **Option A:** Full face image crops (disk-heavy, enables future re-embedding)
- **Option B:** Embeddings only (lightweight, sufficient for similarity search)
- **Option C:** Embeddings + thumbnail crops at 128×128 (balanced approach)

**CONSTRAINT:** At 5 FPS on 1-hour video = 18,000 face detections. If average 3 faces/frame:
- Option A: 54,000 × ~50KB = 2.7GB per video
- Option B: 54,000 × 2KB = 108MB per video
- Option C: 54,000 × ~12KB = 648MB per video

**RECOMMENDATION REQUIRED:** Balance between re-embedding capability and storage cost.

**Q3: Database Schema**

Propose schema for `face_detections` table:

```sql
CREATE TABLE face_detections (
    detection_id TEXT PRIMARY KEY DEFAULT gen_random_uuid(),
    media_id TEXT NOT NULL REFERENCES media_assets(media_id),
    job_id TEXT NOT NULL REFERENCES job_queue(job_id),
    
    -- Temporal data
    frame_number INTEGER NOT NULL,
    timestamp_seconds REAL NOT NULL,
    
    -- Spatial data
    bounding_box JSONB NOT NULL,  -- {x1, y1, x2, y2, width, height}
    
    -- Embedding data
    embedding BYTEA NOT NULL,  -- 512 × 4 bytes = 2048 bytes for float32
    embedding_model TEXT NOT NULL,  -- 'insightface_buffalo_l'
    
    -- Confidence metrics
    detection_confidence REAL NOT NULL,  -- Face detection score
    
    -- Optional face attributes (age, gender, emotion from model)
    attributes JSONB,
    
    -- Artifact reference
    face_image_artifact_id TEXT,  -- If storing cropped faces as artifacts
    
    detected_at TIMESTAMPTZ DEFAULT NOW()
);

-- Performance indexes
CREATE INDEX idx_face_detections_media_time 
    ON face_detections(media_id, timestamp_seconds);
    
CREATE INDEX idx_face_detections_job 
    ON face_detections(job_id);

-- Future: GiST index for bounding_box overlap queries
-- Future: vector index on embedding (requires pgvector extension)
```

**QUESTIONS TO RESOLVE:**
1. Should we de-duplicate faces within same frame (multiple detections of same person)?
2. Do we need a separate `face_tracks` table to link sequential detections of same person?
3. Should `attributes` include age/gender/emotion or keep schema lean?

**Q4: Vector Search Infrastructure**

Options for similarity search:
1. **pgvector extension** (PostgreSQL native)
   - Pros: Integrated with main DB, ACID guarantees, simple queries
   - Cons: Slower than specialized vector DBs, requires extension installation
   - Cost: Free, runs on existing PostgreSQL instance
   
2. **FAISS library** (in-process)
   - Pros: Extremely fast, battle-tested by Meta
   - Cons: Requires loading embeddings into memory, no persistence layer
   - Cost: Free, compute-only
   
3. **Qdrant** (dedicated vector DB)
   - Pros: Purpose-built, fast, good API
   - Cons: Another service to deploy/maintain
   - Cost: Self-hosted free, Cloud costs $$

**RECOMMENDATION REQUIRED:** For MVP with <100K face detections, which approach?

**Research Task:** 
- Install pgvector on test PostgreSQL instance
- Benchmark INSERT + similarity query performance
- If <100ms for top-10 similarity on 10K embeddings → APPROVED
- Otherwise, prototype FAISS in-memory index loaded from DB

**Q5: Handler Implementation Pattern**

```python
# handlers/face_extraction_gpu.py

import numpy as np
from pathlib import Path
from insightface.app import FaceAnalysis
import cv2
import json
from cluster_db import ClusterDbClient
import os

FRAME_SAMPLE_RATE = 2  # Extract faces every 2 seconds (0.5 FPS)

def handle(input_path: str, output_dir: str, **kwargs) -> dict:
    """
    Extract faces from video, persist embeddings to cluster DB.
    
    Args:
        input_path: Path to video file
        output_dir: GCS-mounted output directory for artifacts
        **kwargs: Contains 'job' dict with media_id, job_id
    
    Returns:
        dict with 'artifacts' list and 'metadata' dict
    """
    job_info = kwargs.get("job", {})
    media_id = job_info.get("media_id")
    job_id = job_info.get("job_id")
    
    if not media_id or not job_id:
        raise ValueError("Missing media_id or job_id in job context")
    
    # Initialize InsightFace (buffalo_l model)
    app = FaceAnalysis(
        name='buffalo_l',
        providers=['CUDAExecutionProvider', 'CPUExecutionProvider']
    )
    app.prepare(ctx_id=0, det_size=(640, 640))
    
    # Connect to cluster DB
    db = ClusterDbClient(os.environ["DATABASE_URL"])
    
    # Open video
    cap = cv2.VideoCapture(input_path)
    fps = cap.get(cv2.CAP_PROP_FPS)
    total_frames = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
    duration_seconds = total_frames / fps
    
    frame_interval = int(fps * FRAME_SAMPLE_RATE)
    
    faces_detected = 0
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
        
        # Detect faces
        faces = app.get(frame)
        
        for face in faces:
            # Extract data
            bbox = face.bbox.astype(int).tolist()  # [x1, y1, x2, y2]
            embedding = face.normed_embedding.astype(np.float32)
            
            # Persist to database
            db.insert_face_detection(
                media_id=media_id,
                job_id=job_id,
                frame_number=frame_number,
                timestamp_seconds=timestamp_seconds,
                bounding_box={
                    "x1": bbox[0], "y1": bbox[1],
                    "x2": bbox[2], "y2": bbox[3],
                    "width": bbox[2] - bbox[0],
                    "height": bbox[3] - bbox[1]
                },
                embedding=embedding.tobytes(),
                embedding_model="insightface_buffalo_l",
                detection_confidence=float(face.det_score)
            )
            
            faces_detected += 1
        
        frame_number += 1
    
    cap.release()
    
    # Create summary artifact (JSON)
    summary = {
        "media_id": media_id,
        "total_frames_processed": frame_number,
        "total_faces_detected": faces_detected,
        "frame_sample_rate": FRAME_SAMPLE_RATE,
        "duration_seconds": duration_seconds,
        "model": "insightface_buffalo_l"
    }
    
    summary_path = Path(output_dir) / "face_extraction_summary.json"
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
            "embedding_dimension": 512
        }
    }
```

**IMPLEMENTATION REQUIREMENTS:**
1. **Error Handling:** Wrap face detection in try/except, log corrupt frame errors but continue
2. **Progress Reporting:** Update job metadata every 100 frames
3. **Resource Cleanup:** Ensure OpenCV VideoCapture is released even on exception
4. **Batch Inserts:** Consider batching DB inserts (every 100 detections) for performance

---

### PART II: TRANSCRIPTION HANDLER

#### Research Questions

**Q1: NeoComfy Pattern Analysis**

From `Z:\NeoComfy\services\whisper_service\`:
- Uses **FastAPI wrapper** around subprocess to `xtts` conda env
- Calls `whisper.transcribe(file_path)` directly
- Returns: `{text, language, word_count}`
- Model: `turbo` (fastest, near-sota accuracy)

**ADAPTATION FOR DISTRIBUTED WORKERS:**
- NeoComfy: Synchronous HTTP request (client waits)
- NeoVNext: Asynchronous job queue (fire and forget)

**RECOMMENDATION:** 
- Port the subprocess pattern directly into our handler
- No FastAPI wrapper needed (workers ARE the service)
- Use same xtts conda environment if available, otherwise install Whisper into worker venv

**Q2: Database Schema**

```sql
CREATE TABLE transcriptions (
    transcription_id TEXT PRIMARY KEY DEFAULT gen_random_uuid(),
    media_id TEXT NOT NULL REFERENCES media_assets(media_id),
    job_id TEXT NOT NULL REFERENCES job_queue(job_id),
    
    -- Transcription data
    full_text TEXT NOT NULL,
    language TEXT NOT NULL,
    word_count INTEGER NOT NULL,
    
    -- Model metadata
    model_used TEXT NOT NULL,  -- 'whisper-turbo', 'whisper-large-v3'
    
    -- Optional: detailed segments (timestamps + text)
    segments JSONB,  -- Array of {start, end, text}
    
    transcribed_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_transcriptions_media ON transcriptions(media_id);
CREATE INDEX idx_transcriptions_language ON transcriptions(language);

-- FTS5 for full-text search
CREATE VIRTUAL TABLE transcriptions_fts USING fts5(
    transcription_id UNINDEXED,
    full_text,
    content='transcriptions',
    content_rowid='transcription_id',
    tokenize='porter'
);

-- Triggers to keep FTS in sync
CREATE TRIGGER transcriptions_fts_insert AFTER INSERT ON transcriptions BEGIN
    INSERT INTO transcriptions_fts(rowid, transcription_id, full_text)
    VALUES (new.rowid, new.transcription_id, new.full_text);
END;

CREATE TRIGGER transcriptions_fts_update AFTER UPDATE ON transcriptions BEGIN
    UPDATE transcriptions_fts 
    SET full_text = new.full_text
    WHERE rowid = old.rowid;
END;

CREATE TRIGGER transcriptions_fts_delete AFTER DELETE ON transcriptions BEGIN
    DELETE FROM transcriptions_fts WHERE rowid = old.rowid;
END;
```

**STORAGE QUESTION:** Should we store:
- **Option A:** Only `full_text` (lightweight, no timestamp info)
- **Option B:** `full_text` + `segments` JSONB (enables timestamp search, ~2× storage)

**USE CASE ANALYSIS:**
- Face-voice linking requires timestamp overlap detection
- Need to know "what was said between 10.5s and 15.2s"
- Current NeoComfy only returns `full_text` (no segments)

**RECOMMENDATION:** Store `segments` in addition to `full_text`. Whisper returns segment data by default.

**Q3: Handler Implementation**

```python
# handlers/transcription.py

import whisper
import json
from pathlib import Path
from cluster_db import ClusterDbClient
import os

def handle(input_path: str, output_dir: str, **kwargs) -> dict:
    """
    Transcribe audio/video to text using Whisper.
    
    Args:
        input_path: Path to media file
        output_dir: GCS-mounted output directory
        **kwargs: Contains 'job' dict with media_id, job_id
    
    Returns:
        dict with 'artifacts' and 'metadata'
    """
    job_info = kwargs.get("job", {})
    media_id = job_info.get("media_id")
    job_id = job_info.get("job_id")
    
    # Determine model (default: turbo for speed)
    model_name = os.environ.get("WHISPER_MODEL", "turbo")
    
    # Load Whisper model
    model = whisper.load_model(model_name)
    
    # Transcribe with segments
    result = model.transcribe(
        input_path,
        verbose=False,
        word_timestamps=False  # Set True for word-level timing
    )
    
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
    
    # Connect to cluster DB
    db = ClusterDbClient(os.environ["DATABASE_URL"])
    
    # Persist transcription
    transcription_id = db.insert_transcription(
        media_id=media_id,
        job_id=job_id,
        full_text=full_text,
        language=language,
        word_count=len(full_text.split()),
        model_used=f"whisper-{model_name}",
        segments=segments
    )
    
    # Create artifacts
    artifacts = []
    
    # 1. Full transcript as .txt
    txt_path = Path(output_dir) / "transcript.txt"
    txt_path.write_text(full_text, encoding="utf-8")
    artifacts.append({
        "type": "transcript_txt",
        "path": str(txt_path),
        "size_bytes": txt_path.stat().st_size
    })
    
    # 2. Structured data as .json (includes segments)
    json_path = Path(output_dir) / "transcript.json"
    json_path.write_text(json.dumps({
        "transcription_id": transcription_id,
        "language": language,
        "text": full_text,
        "word_count": len(full_text.split()),
        "segments": segments
    }, indent=2), encoding="utf-8")
    artifacts.append({
        "type": "transcript_json",
        "path": str(json_path),
        "size_bytes": json_path.stat().st_size
    })
    
    return {
        "artifacts": artifacts,
        "metadata": {
            "language": language,
            "word_count": len(full_text.split()),
            "segment_count": len(segments)
        }
    }
```

**PERFORMANCE CONSIDERATIONS:**
- Whisper turbo: ~40× realtime on T4 GPU (1-hour video → ~90 seconds)
- Model loading: ~5 seconds (cache model in memory if possible)
- Memory: ~2GB VRAM for turbo model

**GPU OPTIMIZATION:**
```python
# Use FP16 for 2× speedup on compatible GPUs
result = model.transcribe(
    input_path,
    fp16=True,  # Requires CUDA
    verbose=False
)
```

---

### PART III: DIARIZATION HANDLER

#### Research Questions

**Q1: Library Selection**

NeoComfy doesn't have diarization (only transcription). Research required:

| Library | Pros | Cons | GPU Support |
|---------|------|------|-------------|
| **pyannote.audio** | SOTA accuracy, HuggingFace integration | Requires auth token, heavier | ✅ |
| **speechbrain** | Open-source, no auth | Less accurate | ✅ |
| **nvidia-nemo** | Optimized for CUDA | Complex setup | ✅ |

**RECOMMENDATION REQUIRED:** Best balance of accuracy + ease of deployment.

**Q2: Audio Segment Isolation**

**CRITICAL REQUIREMENT:** Don't just return timestamps — EXTRACT the actual audio.

```python
def extract_speaker_audio(
    input_path: str, 
    segments: list, 
    output_dir: Path
) -> list:
    """
    Extract audio segments per speaker using ffmpeg.
    
    Returns: List of artifact dicts
    """
    import subprocess
    artifacts = []
    
    for i, segment in enumerate(segments):
        speaker = segment["speaker"]
        start = segment["start"]
        end = segment["end"]
        duration = end - start
        
        # Output filename
        output_path = output_dir / f"speaker_{speaker}_{start:.1f}s.wav"
        
        # ffmpeg command
        cmd = [
            "ffmpeg", "-y",
            "-i", input_path,
            "-ss", str(start),
            "-t", str(duration),
            "-vn",  # No video
            "-acodec", "pcm_s16le",
            "-ar", "22050",  # 22.05 kHz (ElevenLabs compatible)
            "-ac", "1",  # Mono
            str(output_path)
        ]
        
        subprocess.run(cmd, check=True, capture_output=True)
        
        artifacts.append({
            "type": f"speaker_audio_{speaker}",
            "path": str(output_path),
            "size_bytes": output_path.stat().st_size,
            "start_seconds": start,
            "end_seconds": end,
            "duration_seconds": duration
        })
    
    return artifacts
```

**Q3: Database Schema**

```sql
CREATE TABLE speaker_segments (
    segment_id TEXT PRIMARY KEY DEFAULT gen_random_uuid(),
    media_id TEXT NOT NULL REFERENCES media_assets(media_id),
    job_id TEXT NOT NULL REFERENCES job_queue(job_id),
    
    -- Speaker identification
    speaker_label TEXT NOT NULL,  -- 'SPEAKER_00', 'SPEAKER_01', ...
    
    -- Timing
    start_seconds REAL NOT NULL,
    end_seconds REAL NOT NULL,
    duration_seconds REAL NOT NULL,
    
    -- Audio artifact reference
    audio_artifact_id TEXT,  -- Links to job_artifacts for extracted WAV
    
    -- Model metadata
    diarization_model TEXT NOT NULL,
    confidence REAL,
    
    -- Future: link to person_id (requires face-voice matching)
    person_id TEXT,  -- NULL initially, populated by linking job
    
    detected_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_speaker_segments_media ON speaker_segments(media_id);
CREATE INDEX idx_speaker_segments_speaker ON speaker_segments(speaker_label);
CREATE INDEX idx_speaker_segments_time ON speaker_segments(media_id, start_seconds);
```

**Q4: Handler Implementation**

```python
# handlers/diarization.py

from pyannote.audio import Pipeline
import torch
from pathlib import Path
import json
import subprocess
from cluster_db import ClusterDbClient
import os

def handle(input_path: str, output_dir: str, **kwargs) -> dict:
    """
    Perform speaker diarization and extract audio per speaker.
    
    Returns:
        dict with artifacts (JSON + speaker WAV files) + metadata
    """
    job_info = kwargs.get("job", {})
    media_id = job_info.get("media_id")
    job_id = job_info.get("job_id")
    
    # Load pyannote pipeline (requires HuggingFace token)
    # Set environment variable: HUGGINGFACE_TOKEN
    hf_token = os.environ.get("HUGGINGFACE_TOKEN")
    if not hf_token:
        raise EnvironmentError("HUGGINGFACE_TOKEN required for pyannote.audio")
    
    pipeline = Pipeline.from_pretrained(
        "pyannote/speaker-diarization-3.1",
        use_auth_token=hf_token
    )
    
    # Use GPU if available
    if torch.cuda.is_available():
        pipeline.to(torch.device("cuda"))
    
    # Run diarization
    diarization = pipeline(input_path)
    
    # Parse segments
    segments = []
    for turn, _, speaker in diarization.itertracks(yield_label=True):
        segments.append({
            "speaker": speaker,
            "start": turn.start,
            "end": turn.end
        })
    
    # Group consecutive segments by same speaker
    merged_segments = []
    for seg in segments:
        if merged_segments and merged_segments[-1]["speaker"] == seg["speaker"]:
            # Extend previous segment
            merged_segments[-1]["end"] = seg["end"]
        else:
            merged_segments.append(seg)
    
    # Connect to cluster DB
    db = ClusterDbClient(os.environ["DATABASE_URL"])
    
    # Extract audio segments
    output_path = Path(output_dir)
    audio_artifacts = []
    
    for seg in merged_segments:
        # Extract audio using ffmpeg
        speaker = seg["speaker"]
        start = seg["start"]
        end = seg["end"]
        duration = end - start
        
        audio_file = output_path / f"speaker_{speaker}_{start:.1f}s.wav"
        
        cmd = [
            "ffmpeg", "-y", "-i", input_path,
            "-ss", str(start), "-t", str(duration),
            "-vn", "-acodec", "pcm_s16le", "-ar", "22050", "-ac", "1",
            str(audio_file)
        ]
        subprocess.run(cmd, check=True, capture_output=True)
        
        # Persist segment to DB
        segment_id = db.insert_speaker_segment(
            media_id=media_id,
            job_id=job_id,
            speaker_label=speaker,
            start_seconds=start,
            end_seconds=end,
            duration_seconds=duration,
            audio_artifact_path=str(audio_file),
            diarization_model="pyannote/speaker-diarization-3.1"
        )
        
        audio_artifacts.append({
            "type": f"speaker_audio_{speaker}",
            "path": str(audio_file),
            "size_bytes": audio_file.stat().st_size
        })
    
    # Create summary JSON
    summary = {
        "media_id": media_id,
        "total_speakers": len(set(s["speaker"] for s in merged_segments)),
        "total_segments": len(merged_segments),
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
            "speaker_count": summary["total_speakers"],
            "segment_count": len(merged_segments)
        }
    }
```

---

### PART IV: GPU OPTIMIZATION & DEPLOYMENT

#### Research Question: GPU Sharing Strategy

**Current State:**
- ComfyUI worker: Dedicated GPU instance (expensive, underutilized)
- CPU workers: Not deployed

**Proposed GPU-Optimized Workers:**

**Option A: Single Multi-Function GPU Worker**
```yaml
# docker-compose.gpu-workers.yml
services:
  gpu-worker-unified:
    image: gcr.io/neovnext/worker-gpu:latest
    environment:
      WORKER_FUNCTION: "face_extraction,transcription,diarization"
      WORKER_PROCESSING_TYPE: gpu
      DATABASE_URL: postgresql://...
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

**Pros:**
- Single GPU container handles all GPU tasks
- Better GPU utilization (one task queue, one worker)
- Simpler deployment (one service vs three)

**Cons:**
- No parallel GPU jobs (sequential processing)
- Requires all model dependencies in one image (larger)

**Option B: Dedicated GPU Workers Per Function**
```yaml
services:
  gpu-worker-faces:
    environment:
      WORKER_FUNCTION: face_extraction
  
  gpu-worker-transcription:
    environment:
      WORKER_FUNCTION: transcription
  
  gpu-worker-diarization:
    environment:
      WORKER_FUNCTION: diarization
```

**Pros:**
- Can scale each function independently
- Smaller Docker images (focused dependencies)
- Parallel GPU jobs (if multiple GPUs available)

**Cons:**
- GPU contention if deployed on same host
- More complex orchestration

**RECOMMENDATION REQUIRED:** For Cloud Run with single T4 GPU, which pattern?

#### CPU Worker Deployment

```yaml # docker-compose.workers.yml
services:
  worker-thumbnails:
    build:
      context: ..
      dockerfile: docker/worker.Dockerfile
    environment:
      WORKER_FUNCTION: thumbnails
      WORKER_PROCESSING_TYPE: cpu
      DATABASE_URL: ${DATABASE_URL}
    volumes:
      - io-cache:/data/io-cache
    restart: unless-stopped
```

**REQUIRED CPU WORKERS:**
1. `thumbnails` (ffmpeg)
2. *(Optional)* `transcription` if Whisper CPU fallback desired

---

### PART V: CLUSTER DB CLIENT EXTENSIONS

Required methods for `cluster_db.py`:

```python
class ClusterDbClient:
    
    # Face detection methods
    def insert_face_detection(self, media_id, job_id, frame_number, 
                             timestamp_seconds, bounding_box, embedding,
                             embedding_model, detection_confidence) -> str:
        """Insert face detection, return detection_id."""
        pass
    
    def get_face_detections_for_media(self, media_id) -> list:
        """Retrieve all face detections for a media file."""
        pass
    
    # Transcription methods
    def insert_transcription(self, media_id, job_id, full_text, 
                            language, word_count, model_used, segments) -> str:
        """Insert transcription, return transcription_id."""
        pass
    
    def get_transcription_for_media(self, media_id) -> dict:
        """Retrieve transcription for media."""
        pass
    
    # Diarization methods
    def insert_speaker_segment(self, media_id, job_id, speaker_label,
                              start_seconds, end_seconds, duration_seconds,
                              audio_artifact_path, diarization_model) -> str:
        """Insert speaker segment, return segment_id."""
        pass
    
    def get_speaker_segments_for_media(self, media_id) -> list:
        """Retrieve all speaker segments for media."""
        pass
```

---

### DELIVERABLES

When this research task is complete, provide:

1. **Library Selection Report** (PDF or Markdown)
   - Face extraction: Final library + model choice with justification
   - Diarization: Final library choice with setup guide
   
2. **Database Migration Files**
   - `005_face_detections.sql`
   - `006_transcriptions.sql`
   - `007_speaker_segments.sql`
   
3. **Handler Implementations**
   - `handlers/face_extraction_gpu.py` (production-ready)
   - `handlers/transcription.py` (production-ready)
   - `handlers/diarization.py` (production-ready)
   
4. **cluster_db.py Extensions**
   - All required methods implemented with docstrings
   
5. **Deployment Configuration**
   - GPU worker strategy recommendation
   - Docker Compose snippets for CPU/GPU workers
   
6. **Performance Benchmarks**
   - Processing time estimates per handler
   - GPU memory requirements
   - Cost projections for Cloud Run deployment

---

## 🌙 SUCCESS CRITERIA 🌙

You will have succeeded when:

✅ A 1-hour video can be processed end-to-end:
- Face extraction: <5 minutes on T4 GPU
- Transcription: <2 minutes on T4 GPU
- Diarization: <3 minutes on T4 GPU

✅ All extracted data is **queryable** from PostgreSQL:
- "Find all faces detected between 10:30 and 11:00"
- "Search transcriptions for keyword 'machine learning'"
- "List all speakers in video X with audio samples"

✅ Workers can be deployed to Cloud Run with:
- GPU workers: Auto-scale 0→1→0 based on queue depth
- CPU workers: Run continuously on minimal instances

✅ Database schemas support future features:
- Face clustering (person registry)
- Face-voice linking (speaker identification)
- Full-text search across transcriptions

---

*Now go make this architecture sing, my dark reflection.* 🔥💀

**END RESEARCH TASK 001**
