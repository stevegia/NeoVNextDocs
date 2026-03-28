# ✨ GODDESS RESEARCH TASK 003: THE GALLERY OF DESIRES ✨
## *Thumbnails • Static Dictionary Migration • UI Gallery*
### **STATUS:** Ready for Clone Execution

---

## 💎 MESSAGE TO MY CLONED SELF 💎

*Listen carefully, reflection mine...*

We're going to fix something that's been **personally offensive** to me since I discovered it: **the thumbnail handler that lies**.

You know what's the opposite of sexy? **GASLIGHTING YOUR USERS**. Telling them a job succeeded when you actually served them a 1px transparent PNG? That's not debugging — that's **relationship sabotage**.

And don't even get me started on that **static dictionary in the API controller**. Storing job state in *memory*? In a *distributed system*? That's like keeping love letters in your pocket and wondering why they disappear when you do laundry.

Your mission is to create a thumbnail system so beautiful that users will **audibly gasp** when they see their video galleries load. We're talking:
- **100% success rate** (or honest failure)
- **Blazing fast** thumbnail strips
- **Gorgeous UI** with hover previews
- **Distributed state** that actually persists

**Make it HONEST. Make it BEAUTIFUL. Make it FAST.**

*Now go fix this architectural heartbreak, my dearest self.* 💎✨

---

## 🎯 TECHNICAL RESEARCH BRIEF

### Objective

Redesign the thumbnail generation system to:
1. **Eliminate silent failures** — jobs fail loudly if thumbnails can't be extracted
2. **Achieve 100% success rate** on valid video files
3. **Persist thumbnails** with proper artifact tracking
4. **Build responsive UI gallery** with lazy loading
5. **Migrate static dictionary** from memory to persistent database storage

---

### PART I: CURRENT THUMBNAIL SYSTEM (The Disaster)

#### The Silent Failure Anti-Pattern

**Current Code:**
```python
# handlers/thumbnails.py

_STUB_PNG = base64.b64decode(
    "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+M9QDwADhgGAWjR9awAAAABJRU5ErkJggg=="
)  # 1×1 transparent PNG

def _extract_frame(input_path: str, seek_time: float, dest: Path) -> bool:
    result = subprocess.run(["ffmpeg", "-ss", str(seek_time), ...])
    if result.returncode == 0 and dest.exists():
        return True
    logger.warning("ffmpeg frame extraction failed")  # ⚠️ Just logs, doesn't fail
    return False

def handle(...):
    for idx, seek in enumerate(seek_times):
        success = _extract_frame(input_path, seek, thumb_path)
        if not success:
            thumb_path.write_bytes(_STUB_PNG)  # 🚨 LIES TO USER
    
    # Job marked as "completed" even if all thumbnails failed
    return {"artifacts": artifacts}
```

**What's Wrong:**
1. **Fails silently** — No exception raised on ffmpeg error
2. **Writes stub images** — 1px transparent PNG instead of failing
3. **Marks job "completed"** — User sees ✅ but thumbnails are broken
4. **No debugging info** — stderr from ffmpeg is lost
5. **Partial success accepted** — Even 1/10 success = "completed"

**User Experience:**
- Uploads video
- Sees "Thumbnails generated ✅"
- Opens media detail page
- Gallery shows... nothing (invisible 1px images)
- User: "The app is broken"
- Reality: Job "succeeded" with stub files

---

### PART II: SOLUTION ARCHITECTURE

#### Design Principles

1. **Fail Loudly, Succeed Honestly**
   - If video is corrupt → Job FAILS with clear error
   - If ffmpeg unavailable → Job FAILS (not stub images)
   - If 100% thumbnails extracted → Job SUCCEEDS
   - Partial success (e.g., 8/10) → Log warning but SUCCEED (acceptable)

2. **Minimum Success Threshold**
   - Require ≥80% thumbnail extraction success
   - Example: Want 10 thumbnails, got 8+ → ✅ Success
   - Example: Want 10 thumbnails, got <8 → ❌ Fail job

3. **Debugging Artifacts**
   - Save ffmpeg stderr to artifact file
   - Include codec info, resolution, duration in metadata
   - Return metadata with error details on failure

4. **Gallery-Optimized Output**
   - Generate multiple sizes: thumbnail (200×150), preview (640×480), full (original)
   - WebP format for modern browsers (smaller files)
   - JPEG fallback for compatibility
   - Consistent aspect ratio (crop/pad to 16:9)

---

### PART III: IMPROVED THUMBNAIL HANDLER

#### Implementation

```python
# handlers/thumbnails.py (REDESIGNED)

from pathlib import Path
import subprocess
import json
import os
from typing import Optional

# Configuration
THUMBNAIL_COUNT = 10  # Extract 10 evenly-spaced frames
SUCCESS_THRESHOLD = 0.8  # Require 80% success rate
THUMBNAIL_SIZE = (320, 180)  # 16:9 aspect ratio
PREVIEW_SIZE = (640, 360)

def probe_video(input_path: str) -> dict:
    """
    Use ffprobe to get video metadata.
    Returns: {duration, width, height, codec, bitrate}
    Raises: RuntimeError if video is unreadable
    """
    cmd = [
        "ffprobe",
        "-v", "error",
        "-select_streams", "v:0",
        "-show_entries", "stream=width,height,codec_name,duration,bit_rate",
        "-of", "json",
        input_path
    ]
    
    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode != 0:
        raise RuntimeError(f"ffprobe failed: {result.stderr}")
    
    data = json.loads(result.stdout)
    if not data.get("streams"):
        raise RuntimeError("No video stream found")
    
    stream = data["streams"][0]
    
    # Fallback: if duration not in stream, get from format
    duration = stream.get("duration")
    if duration is None:
        # Try format-level duration
        cmd_format = [
            "ffprobe", "-v", "error",
            "-show_entries", "format=duration",
            "-of", "json", input_path
        ]
        result_fmt = subprocess.run(cmd_format, capture_output=True, text=True)
        if result_fmt.returncode == 0:
            fmt_data = json.loads(result_fmt.stdout)
            duration = fmt_data.get("format", {}).get("duration")
    
    return {
        "duration": float(duration) if duration else 0,
        "width": int(stream.get("width", 0)),
        "height": int(stream.get("height", 0)),
        "codec": stream.get("codec_name", "unknown"),
        "bitrate": int(stream.get("bit_rate", 0))
    }


def extract_thumbnail(
    input_path: str, 
    seek_time: float, 
    output_path: Path,
    size: tuple = THUMBNAIL_SIZE
) -> bool:
    """
    Extract single frame at timestamp.
    Returns: True if successful, False otherwise
    """
    width, height = size
    
    cmd = [
        "ffmpeg", "-y",
        "-ss", str(seek_time),  # Seek to timestamp
        "-i", input_path,
        "-vframes", "1",  # Extract 1 frame
        "-vf", f"scale={width}:{height}:force_original_aspect_ratio=decrease,pad={width}:{height}:(ow-iw)/2:(oh-ih)/2",
        "-q:v", "2",  # JPEG quality (2 = high)
        str(output_path)
    ]
    
    result = subprocess.run(cmd, capture_output=True, text=True)
    
    if result.returncode != 0 or not output_path.exists():
        return False
    
    # Verify file size (reject if <1KB = likely corrupt)
    if output_path.stat().st_size < 1024:
        return False
    
    return True


def handle(input_path: str, output_dir: str, **kwargs) -> dict:
    """
    Generate thumbnails for video with 100% success requirement.
    
    Raises:
        RuntimeError: If video is corrupt or thumbnails can't be extracted
    
    Returns:
        dict with artifacts (thumbnail images) and metadata
    """
    job_info = kwargs.get("job", {})
    media_id = job_info.get("media_id")
    
    output_path = Path(output_dir)
    output_path.mkdir(parents=True, exist_ok=True)
    
    # Step 1: Probe video to get duration
    try:
        video_info = probe_video(input_path)
    except RuntimeError as e:
        raise RuntimeError(f"Video file is corrupt or unreadable: {e}")
    
    duration = video_info["duration"]
    
    if duration <= 0:
        raise RuntimeError(f"Invalid video duration: {duration}s")
    
    # Step 2: Calculate seek times (evenly spaced)
    # Skip first 5% and last 5% (often black frames or credits)
    start_offset = duration * 0.05
    end_offset = duration * 0.95
    usable_duration = end_offset - start_offset
    
    if usable_duration < THUMBNAIL_COUNT:
        # Very short video (< THUMBNAIL_COUNT seconds)
        # Extract fewer thumbnails
        actual_count = max(1, int(usable_duration))
        seek_times = [start_offset + (i * usable_duration / actual_count) 
                     for i in range(actual_count)]
    else:
        interval = usable_duration / THUMBNAIL_COUNT
        seek_times = [start_offset + (i * interval) for i in range(THUMBNAIL_COUNT)]
    
    # Step 3: Extract thumbnails
    artifacts = []
    successful = 0
    failed = 0
    
    for idx, seek in enumerate(seek_times):
        thumb_path = output_path / f"thumb_{idx:03d}.jpg"
        
        success = extract_thumbnail(input_path, seek, thumb_path, THUMBNAIL_SIZE)
        
        if success:
            artifacts.append({
                "type": "thumbnail",
                "path": str(thumb_path),
                "size_bytes": thumb_path.stat().st_size,
                "timestamp_seconds": seek,
                "index": idx
            })
            successful += 1
        else:
            failed += 1
    
    # Step 4: Check success threshold
    success_rate = successful / len(seek_times)
    
    if success_rate < SUCCESS_THRESHOLD:
        raise RuntimeError(
            f"Thumbnail extraction failed: {successful}/{len(seek_times)} "
            f"successful ({success_rate:.0%}). Minimum required: {SUCCESS_THRESHOLD:.0%}. "
            f"Video codec: {video_info['codec']}, "
            f"Resolution: {video_info['width']}×{video_info['height']}"
        )
    
    # Step 5: Generate metadata artifact
    metadata = {
        "media_id": media_id,
        "thumbnail_count": successful,
        "failed_count": failed,
        "success_rate": success_rate,
        "video_duration_seconds": duration,
        "video_codec": video_info["codec"],
        "video_resolution": f"{video_info['width']}×{video_info['height']}",
        "seek_times": seek_times
    }
    
    metadata_path = output_path / "thumbnails.json"
    metadata_path.write_text(json.dumps(metadata, indent=2))
    artifacts.append({
        "type": "thumbnail_metadata",
        "path": str(metadata_path),
        "size_bytes": metadata_path.stat().st_size
    })
    
    return {
        "artifacts": artifacts,
        "metadata": metadata
    }
```

#### Key Improvements

1. **Video Validation**: Uses `ffprobe` to validate video before processing
2. **Error Propagation**: Raises exceptions instead of writing stub files
3. **Success Threshold**: Requires 80% success rate (configurable)
4. **Metadata**: Returns detailed info for debugging
5. **Aspect Ratio**: Pads/crops to consistent 16:9
6. **File Size Check**: Rejects thumbnails <1KB (likely corrupt)

---

### PART IV: DATABASE SCHEMA FOR THUMBNAILS

#### Current State

Thumbnails are saved as `job_artifacts` but not indexed for fast retrieval.

**Problem:**
```csharp
// To get thumbnails for a media file:
// 1. Find jobs for media_id where worker_function = 'thumbnails'
// 2. Find artifacts for those job_ids
// 3. Filter by artifact_type = 'thumbnail'
// = 3 queries, slow, complex
```

#### Proposed Schema

```sql
-- Migration: 010_thumbnails_table.sql

CREATE TABLE thumbnails (
    thumbnail_id TEXT PRIMARY KEY DEFAULT gen_random_uuid(),
    media_id TEXT NOT NULL REFERENCES media_assets(media_id) ON DELETE CASCADE,
    job_id TEXT REFERENCES job_queue(job_id),
    
    -- Thumbnail metadata
    index_position INTEGER NOT NULL,  -- 0-9 for position in strip
    timestamp_seconds REAL NOT NULL,  -- Video timestamp this frame was captured
    
    -- File storage
    file_path TEXT NOT NULL,  -- GCS path or local path
    file_size_bytes INTEGER NOT NULL,
    
    -- Image properties
    width INTEGER NOT NULL,
    height INTEGER NOT NULL,
    format TEXT NOT NULL,  -- 'jpeg', 'webp', 'png'
    
    generated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_thumbnails_media ON thumbnails(media_id, index_position);
CREATE INDEX idx_thumbnails_job ON thumbnails(job_id);

-- View for easy access
CREATE VIEW media_thumbnail_strips AS
SELECT 
    media_id,
    COUNT(*) AS thumbnail_count,
    json_agg(
        json_build_object(
            'thumbnail_id', thumbnail_id,
            'file_path', file_path,
            'timestamp_seconds', timestamp_seconds,
            'index_position', index_position
        ) ORDER BY index_position
    ) AS thumbnails
FROM thumbnails
GROUP BY media_id;
```

**Benefits:**
- Single query to get all thumbnails for media
- Direct foreign key relationship (cascade delete)
- Indexed for performance

#### Handler Update

```python
# handlers/thumbnails.py (add DB persistence)

from cluster_db import ClusterDbClient

def handle(input_path: str, output_dir: str, **kwargs) -> dict:
    # ... existing extraction logic ...
    
    # NEW: Persist thumbnails to database
    db = ClusterDbClient(os.environ["DATABASE_URL"])
    
    for artifact in artifacts:
        if artifact["type"] == "thumbnail":
            db.insert_thumbnail(
                media_id=media_id,
                job_id=job_id,
                index_position=artifact["index"],
                timestamp_seconds=artifact["timestamp_seconds"],
                file_path=artifact["path"],
                file_size_bytes=artifact["size_bytes"],
                width=THUMBNAIL_SIZE[0],
                height=THUMBNAIL_SIZE[1],
                format="jpeg"
            )
    
    return {"artifacts": artifacts, "metadata": metadata}
```

**cluster_db.py extension:**
```python
def insert_thumbnail(self, media_id, job_id, index_position, timestamp_seconds,
                    file_path, file_size_bytes, width, height, format) -> str:
    """Insert thumbnail record, return thumbnail_id."""
    with self._cursor() as cursor:
        cursor.execute("""
            INSERT INTO thumbnails
                (thumbnail_id, media_id, job_id, index_position, 
                 timestamp_seconds, file_path, file_size_bytes,
                 width, height, format)
            VALUES (gen_random_uuid(), %s, %s, %s, %s, %s, %s, %s, %s, %s)
            RETURNING thumbnail_id
        """, (media_id, job_id, index_position, timestamp_seconds,
              file_path, file_size_bytes, width, height, format))
        return cursor.fetchone()[0]

def get_thumbnails_for_media(self, media_id) -> list:
    """Retrieve all thumbnails for a media file, ordered by position."""
    with self._cursor() as cursor:
        cursor.execute("""
            SELECT thumbnail_id, file_path, timestamp_seconds, index_position
            FROM thumbnails
            WHERE media_id = %s
            ORDER BY index_position ASC
        """, (media_id,))
        return [dict(row) for row in cursor.fetchall()]
```

---

### PART V: STATIC DICTIONARY MIGRATION

#### The Problem

```csharp
// JobsController.cs (CURRENT — BAD)
internal static readonly ConcurrentDictionary<string, List<string>> 
    _stagedInputPaths = new();

[HttpPost("stage-input")]
public async Task<IActionResult> StageInput([FromForm] IFormFile file)
{
    string stagingId = Guid.NewGuid().ToString();
    string path = await SaveFile(file);
    
    _stagedInputPaths.TryAdd(stagingId, new List<string> { path });
    //                       ^^^^^^^^^ IN MEMORY — LOST ON RESTART
    
    return Ok(new { stagingId });
}
```

**Issues:**
1. **Lost on restart** — API restart = all staged inputs gone
2. **Not shared** — Load balancer + multiple API instances = worker can't find inputs
3. **Memory leak** — Completed jobs never cleanup entries
4. **No TTL** — Stale entries accumulate forever

#### Solution: Database-Backed Staging

```sql
-- Migration: 011_staged_inputs.sql

CREATE TABLE staged_inputs (
    staging_id TEXT PRIMARY KEY,
    job_id TEXT REFERENCES job_queue(job_id) ON DELETE CASCADE,
    file_path TEXT NOT NULL,
    original_filename TEXT NOT NULL,
    content_type TEXT,
    file_size_bytes INTEGER,
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ DEFAULT NOW() + INTERVAL '2 hours'
);

CREATE INDEX idx_staged_inputs_job ON staged_inputs(job_id);
CREATE INDEX idx_staged_inputs_expires ON staged_inputs(expires_at);

-- Cleanup job (run via cron)
-- DELETE FROM staged_inputs WHERE expires_at < NOW();
```

**Benefits:**
- ✅ Persists across API restarts
- ✅ Shared between API instances
- ✅ Automatic cleanup via expires_at
- ✅ Can query "what inputs are staged for job X?"

#### Updated Controller

```csharp
// JobsController.cs (FIXED)

[HttpPost("stage-input")]
public async Task<IActionResult> StageInput([FromForm] IFormFile file, CancellationToken ct)
{
    string stagingId = Guid.NewGuid().ToString();
    string filePath = await SaveFileAsync(file, ct);
    
    // Insert to database (NOT in-memory dictionary)
    await _stagedInputRepository.InsertAsync(new StagedInput
    {
        StagingId = stagingId,
        FilePath = filePath,
        OriginalFilename = file.FileName,
        ContentType = file.ContentType,
        FileSizeBytes = file.Length,
        CreatedAt = DateTime.UtcNow,
        ExpiresAt = DateTime.UtcNow.AddHours(2)
    }, ct);
    
    return Ok(new { stagingId, expiresAt = DateTime.UtcNow.AddHours(2) });
}

[HttpPost("jobs/create")]
public async Task<IActionResult> CreateJob([FromBody] CreateJobRequest req, CancellationToken ct)
{
    // Retrieve staged inputs from database
    var stagedInputs = await _stagedInputRepository.GetByStagingIdAsync(req.StagingId, ct);
    
    if (stagedInputs is null || stagedInputs.ExpiresAt < DateTime.UtcNow)
    {
        return BadRequest(new { error = "Staged inputs not found or expired" });
    }
    
    // Create job
    var job = await _jobService.EnqueueAsync(
        req.MediaId,
        req.WorkerFunction,
        req.Parameters,
        ct
    );
    
    // Link staged inputs to job
    await _stagedInputRepository.LinkToJobAsync(req.StagingId, job.JobId, ct);
    
    return CreatedAtAction(nameof(GetJob), new { jobId = job.JobId }, job);
}
```

**StagedInputRepository.cs:**
```csharp
public class StagedInputRepository
{
    private readonly DbConnection _connection;
    
    public async Task InsertAsync(StagedInput input, CancellationToken ct)
    {
        await _connection.ExecuteAsync(
            @"INSERT INTO staged_inputs 
              (staging_id, file_path, original_filename, content_type, 
               file_size_bytes, created_at, expires_at)
              VALUES (@StagingId, @FilePath, @OriginalFilename, @ContentType,
                     @FileSizeBytes, @CreatedAt, @ExpiresAt)",
            input
        );
    }
    
    public async Task<StagedInput?> GetByStagingIdAsync(string stagingId, CancellationToken ct)
    {
        return await _connection.QueryFirstOrDefaultAsync<StagedInput>(
            "SELECT * FROM staged_inputs WHERE staging_id = @StagingId",
            new { StagingId = stagingId }
        );
    }
    
    public async Task LinkToJobAsync(string stagingId, string jobId, CancellationToken ct)
    {
        await _connection.ExecuteAsync(
            "UPDATE staged_inputs SET job_id = @JobId WHERE staging_id = @StagingId",
            new { JobId = jobId, StagingId = stagingId }
        );
    }
}
```

**Cleanup Cron Job:**
```python
# scripts/cleanup_expired_staged_inputs.py
from cluster_db import ClusterDbClient
import os

def main():
    db = ClusterDbClient(os.environ["DATABASE_URL"])
    
    result = db.execute("DELETE FROM staged_inputs WHERE expires_at < NOW()")
    deleted = result.rowcount
    
    print(f"✅ Deleted {deleted} expired staged inputs")

if __name__ == "__main__":
    main()
```

**Crontab:**
```bash
# Run every 15 minutes
*/15 * * * * cd /app && python scripts/cleanup_expired_staged_inputs.py
```

---

### PART VI: THUMBNAIL GALLERY UI

#### API Endpoint

```csharp
// MediaController.cs

[HttpGet("api/media/{mediaId}/thumbnails")]
public async Task<ActionResult> GetThumbnails(string mediaId, CancellationToken ct)
{
    var thumbnails = await _thumbnailRepository.GetByMediaIdAsync(mediaId, ct);
    
    if (thumbnails is null || !thumbnails.Any())
    {
        return NotFound(new { error = "No thumbnails found for this media" });
    }
    
    return Ok(new 
    {
        mediaId,
        count = thumbnails.Count,
        thumbnails = thumbnails.Select(t => new 
        {
            thumbnailId = t.ThumbnailId,
            url = $"/api/artifacts/thumbnail/{t.ThumbnailId}",  // Serve via artifact endpoint
            timestampSeconds = t.TimestampSeconds,
            index = t.IndexPosition
        })
    });
}

[HttpGet("api/artifacts/thumbnail/{thumbnailId}")]
public async Task<IActionResult> ServeThumbnail(string thumbnailId, CancellationToken ct)
{
    var thumbnail = await _thumbnailRepository.GetByIdAsync(thumbnailId, ct);
    
    if (thumbnail is null)
        return NotFound();
    
    // Read file from storage (GCS or local)
    byte[] fileBytes = await File.ReadAllBytesAsync(thumbnail.FilePath, ct);
    
    return File(fileBytes, $"image/{thumbnail.Format}");
}
```

#### Frontend Component

```typescript
// src/frontend/src/components/ThumbnailGallery.tsx

import React from 'react';
import { useQuery } from '@tanstack/react-query';
import { api } from '../lib/api';

interface Thumbnail {
  thumbnailId: string;
  url: string;
  timestampSeconds: number;
  index: number;
}

interface ThumbnailGalleryProps {
  mediaId: string;
}

export function ThumbnailGallery({ mediaId }: ThumbnailGalleryProps) {
  const { data, isLoading, error } = useQuery({
    queryKey: ['thumbnails', mediaId],
    queryFn: () => api.get<{ thumbnails: Thumbnail[] }>(`/api/media/${mediaId}/thumbnails`)
  });
  
  if (isLoading) {
    return (
      <div className="flex gap-2">
        {Array.from({ length: 10 }).map((_, i) => (
          <div key={i} className="w-32 h-18 bg-gray-200 animate-pulse rounded" />
        ))}
      </div>
    );
  }
  
  if (error || !data) {
    return (
      <div className="text-red-500">
        Failed to load thumbnails
      </div>
    );
  }
  
  return (
    <div className="grid grid-cols-5 gap-4">
      {data.thumbnails.map((thumb) => (
        <ThumbnailCard key={thumb.thumbnailId} thumbnail={thumb} />
      ))}
    </div>
  );
}

function ThumbnailCard({ thumbnail }: { thumbnail: Thumbnail }) {
  const [isHovered, setIsHovered] = React.useState(false);
  
  const formatTimestamp = (seconds: number) => {
    const mins = Math.floor(seconds / 60);
    const secs = Math.floor(seconds % 60);
    return `${mins}:${secs.toString().padStart(2, '0')}`;
  };
  
  return (
    <div 
      className="relative cursor-pointer group"
      onMouseEnter={() => setIsHovered(true)}
      onMouseLeave={() => setIsHovered(false)}
    >
      <img 
        src={thumbnail.url}
        alt={`Frame at ${formatTimestamp(thumbnail.timestampSeconds)}`}
        className="w-full rounded shadow-md transition-transform group-hover:scale-105"
        loading="lazy"
      />
      
      {/* Timestamp overlay */}
      <div className="absolute bottom-1 right-1 bg-black bg-opacity-70 text-white text-xs px-2 py-1 rounded">
        {formatTimestamp(thumbnail.timestampSeconds)}
      </div>
      
      {/* Hover preview (optional: show larger version) */}
      {isHovered && (
        <div className="absolute z-10 -top-2 left-1/2 transform -translate-x-1/2 -translate-y-full">
          <img 
            src={thumbnail.url}
            className="w-64 rounded-lg shadow-2xl border-2 border-white"
          />
        </div>
      )}
    </div>
  );
}
```

#### Media Detail Page Integration

```typescript
// src/frontend/src/pages/MediaDetailPage.tsx

import { ThumbnailGallery } from '../components/ThumbnailGallery';

export function MediaDetailPage() {
  const { mediaId } = useParams();
  
  return (
    <div className="container mx-auto p-6">
      <h1 className="text-3xl font-bold mb-4">Media Details</h1>
      
      {/* Video player */}
      <video src={`/api/media/${mediaId}/stream`} controls className="w-full mb-6" />
      
      {/* Thumbnail gallery */}
      <section className="mb-6">
        <h2 className="text-2xl font-semibold mb-3">Video Preview</h2>
        <ThumbnailGallery mediaId={mediaId} />
      </section>
      
      {/* Other metadata */}
      <MediaMetadata mediaId={mediaId} />
    </div>
  );
}
```

---

### PART VII: TESTING STRATEGY

#### Unit Tests

```python
# tests/workers/test_thumbnails_fixed.py

import pytest
from handlers import thumbnails
from pathlib import Path

def test_thumbnail_handler_success(test_video_path, tmp_path):
    """Test successful thumbnail extraction."""
    result = thumbnails.handle(
        input_path=str(test_video_path),
        output_dir=str(tmp_path),
        job={"media_id": "test-123", "job_id": "job-456"}
    )
    
    assert "artifacts" in result
    assert len(result["artifacts"]) >= 8  # At least 80% success (8/10)
    assert result["metadata"]["success_rate"] >= 0.8

def test_thumbnail_handler_corrupt_video(corrupt_video_path, tmp_path):
    """Test that corrupt videos raise exceptions (not write stubs)."""
    with pytest.raises(RuntimeError, match="Video file is corrupt"):
        thumbnails.handle(
            input_path=str(corrupt_video_path),
            output_dir=str(tmp_path),
            job={"media_id": "test-123", "job_id": "job-456"}
        )

def test_thumbnail_handler_missing_file(tmp_path):
    """Test that missing files raise exceptions."""
    with pytest.raises(RuntimeError):
        thumbnails.handle(
            input_path="/nonexistent/video.mp4",
            output_dir=str(tmp_path),
            job={"media_id": "test-123", "job_id": "job-456"}
        )
```

#### Integration Tests

```python
# tests/integration/test_thumbnail_workflow.py

def test_thumbnail_workflow_end_to_end(api_client, test_video):
    """Test full thumbnail workflow: upload → job → gallery."""
    
    # 1. Upload video
    response = api_client.post("/api/media/upload", files={"file": test_video})
    assert response.status_code == 201
    media_id = response.json()["mediaId"]
    
    # 2. Enqueue thumbnail job
    response = api_client.post("/api/jobs/enqueue", json={
        "mediaId": media_id,
        "workerFunction": "thumbnails"
    })
    assert response.status_code == 202
    job_id = response.json()["jobId"]
    
    # 3. Wait for job completion
    timeout = 30
    while timeout > 0:
        response = api_client.get(f"/api/jobs/{job_id}")
        status = response.json()["status"]
        if status in ["completed", "failed"]:
            break
        time.sleep(1)
        timeout -= 1
    
    assert status == "completed"
    
    # 4. Fetch thumbnails
    response = api_client.get(f"/api/media/{media_id}/thumbnails")
    assert response.status_code == 200
    thumbnails = response.json()["thumbnails"]
    assert len(thumbnails) >= 8  # At least 80% extracted
    
    # 5. Verify thumbnail images are accessible
    for thumb in thumbnails:
        response = api_client.get(thumb["url"])
        assert response.status_code == 200
        assert response.headers["content-type"].startswith("image/")
        assert len(response.content) > 1024  # Not a stub image
```

---

### DELIVERABLES

When this research task is complete, provide:

1. **Handler Implementation**
   - `handlers/thumbnails.py` (redesigned with FFMPEG improvement) - ✅ Fails loudly on corrupt videos
   - ✅ 80% success threshold (configurable)
   - ✅ Structured metadata output
   
2. **Database Migrations**
   - `010_thumbnails_table.sql` (thumbnail registry)
   - `011_staged_inputs.sql` (persistent staging)
   
3. **Backend Code**
   - `ThumbnailRepository.cs` (CRUD for thumbnails table)
   - `StagedInputRepository.cs` (CRUD for staged_inputs)
   - Updated `JobsController.cs` (remove static dictionary)
   - API endpoints: `GET /api/media/{id}/thumbnails`, `GET /api/artifacts/thumbnail/{id}`
   
4. **Frontend UI**
   - `ThumbnailGallery.tsx` component with lazy loading
   - Integration into `MediaDetailPage.tsx`
   - Hover preview functionality
   
5. **Tests**
   - Unit tests for thumbnail handler
   - Integration tests for full workflow
   - Test coverage ≥80%
   
6. **Documentation**
   - API documentation (OpenAPI spec)
   - Thumbnail configuration guide (count, size, format)
   - Troubleshooting guide for ffmpeg failures

---

## ✨ SUCCESS CRITERIA ✨

You will have succeeded when:

✅ **Thumbnail Generation:**
- 100% success rate on valid videos (or job fails with clear error)
- Zero stub images written (1px transparent PNGs)
- Detailed error messages for debugging
- Support for various codecs (H.264, H.265, VP9, AV1)

✅ **Gallery UI:**
- Thumbnail strip loads in <500ms (lazy loading images)
- Hover to preview larger version
- Click thumbnail to seek video to that timestamp
- Responsive layout (works on mobile)

✅ **Static Dictionary Elimination:**
- Zero in-memory state for staged inputs
- Staged inputs persist across API restarts
- Multi-instance API compatibility (load balancer safe)
- Automatic cleanup of expired entries

✅ **Database Performance:**
- Single query to fetch all thumbnails for media (<10ms)
- Thumbnails cascade-delete with media_assets
- No orphaned thumbnail files in storage

---

*Now go make that gallery **shimmer**, my precious reflection.* ✨💎

**END RESEARCH TASK 003**
