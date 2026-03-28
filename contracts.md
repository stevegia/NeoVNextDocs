# Cross-Layer Contracts

This document defines the interfaces and data contracts between the major layers of NeoVLab: the API, the GPU supervisor, the Python handler subprocesses, ComfyUI, and the shared databases.

For ERD diagrams, see [docs/diagrams/db-schema.md](diagrams/db-schema.md).
For job lifecycle sequences, see [docs/diagrams/job-lifecycle.md](diagrams/job-lifecycle.md).

---

## Architecture Overview

```
Frontend (React)
    |
    v
API (C# / .NET 9, runs on desktop)
    |
    |--- writes to ---> Local SQLite (media_assets, scan_sessions, etc.)
    |--- writes to ---> Cluster PostgreSQL (job_queue, cache_manifest, etc.)
    |--- uploads inputs ---> GCS bucket (io-cache)
    |--- triggers ---> GCP Cloud Run warm-up
    |
    v
GPU Supervisor (Go, runs on Cloud Run with L4 GPU)
    |--- polls ---> Cluster PostgreSQL (claim jobs, heartbeat)
    |--- manages ---> ComfyUI process (localhost:8188)
    |--- manages ---> Python handler subprocesses (localhost:9001-9006)
    |--- registers output ---> Cluster PostgreSQL (cache_manifest)
    |
    v
API (polls cluster DB, syncs artifacts back to local storage)
```

---

## 1. API -> Database Contracts

### 1.1 Local SQLite Database

Source: `src/data/migrations/001_local_api_schema.sql`

The local database runs on the desktop alongside the API. It stores media library metadata and configuration that does not need to be shared with GPU workers.

| Table | Purpose |
|---|---|
| `media_assets` | Media library catalog (file path, dimensions, duration, codec, file hash). FTS5 index on file_name/file_path. |
| `scan_sessions` | Filesystem scan tracking (root_path, discovered_count, status). |
| `model_registry` | GPU model files registry (filename, volume_path, size_bytes, source_url). |
| `gpu_platform_config` | GCP Cloud Run GPU configuration (enabled flag, gpu_type, max_workers, idle_timeout). |
| `execution_policies` | Per-job-type routing rules (execution_mode, gpu_tier, requires_gpu). |
| `budget_tracking` | Monthly spend tracking and caps (budget_cents, spent_cents, gpu-specific counters). |
| `job_routing_log` | Audit log for job dispatch decisions (policy_name, decision, reason). |
| `tier_config` | Storage tier definitions (hot = io-cache with 5-day retention, cold = local-storage). |
| `job_artifacts` | Local artifact registry (artifact_type, file_path, sync_eligible). |
| `staged_inputs` | ComfyUI input image staging (staging_id, file_path, expires_at). |

### 1.2 Cluster PostgreSQL Database

Source: `src/data/migrations/cluster/001_cluster_schema.sql` through `008_handler_config.sql`

The cluster database is Cloud SQL PostgreSQL, shared between the desktop API and GPU workers.

| Table | Purpose |
|---|---|
| `job_queue` | Central job queue with priority-based pull-model dispatch. |
| `worker_registry` | Worker heartbeat and capability tracking. |
| `cache_manifest` | Input/output file tracking for GCS FUSE-mounted I/O cache. |
| `handler_config` | Runtime enable/disable and priority for each handler function. |
| `staged_inputs` | Durable staging for ComfyUI input files (replaces in-memory dict). |
| `thumbnails` | Per-video thumbnail metadata (sequence_number, timestamp, dimensions). |
| `face_detections` | Per-frame face detection with 512-dim pgvector embeddings (InsightFace). |
| `transcriptions` | Whisper transcription output with tsvector full-text search. |
| `speaker_segments` | Pyannote diarization segments with 192-dim voice embeddings. |

### 1.3 Job Enqueue Flow (API -> PostgreSQL)

When the API receives a job request, `JobQueueService.EnqueueAsync()` performs:

1. Resolves `processing_type` (cpu/gpu) and `worker_function` from `job_type`:

| job_type | processing_type | worker_function |
|---|---|---|
| `comfyui` | gpu | `comfyui` |
| `wan_*` | gpu | `comfyui` |
| `face_extraction` | gpu | `face_extraction` |
| `visual_tagging` | gpu | `visual_tagging` |
| `transcription` | gpu | `transcription` |
| `diarization` | gpu | `diarization` |
| `thumbnails` | cpu | `thumbnails` |
| `transcript_analysis` | cpu | `transcript_analysis` |

2. Inserts into `job_queue` with `status = 'queued'`.
3. For non-ComfyUI jobs: stages the media file to GCS via `InputStagingService`, inserts into `cache_manifest` with `direction = 'input'`.
4. For ComfyUI jobs: input images are staged separately via `POST /api/comfy/input` before submission.
5. Triggers GPU warm-up via `IGpuPlatformAdapter.EnsureWarmAsync()` if `processing_type = 'gpu'`.
6. Logs the routing decision to `job_routing_log`.

---

## 2. GPU Supervisor -> Database Contracts

Source: `src/gpu-supervisor/internal/db/`

The Go supervisor connects directly to the cluster PostgreSQL database.

### 2.1 Worker Registration

On startup, the supervisor calls `db.RegisterWorker()`:

```sql
INSERT INTO worker_registry (worker_id, processing_type, worker_function, state, last_heartbeat, capabilities_json, registered_at)
VALUES ($1, $2, $3, 'starting', NOW(), $4, NOW())
ON CONFLICT (worker_id) DO UPDATE SET ...
```

`capabilities_json` contains:
```json
{
  "max_concurrent": 1,
  "processing_type": "gpu",
  "worker_functions": ["comfyui", "face_extraction", "visual_tagging", "transcription", "diarization"],
  "gpu_mode": "shared"
}
```

### 2.2 Heartbeat

Every 30 seconds, `db.Heartbeat()` updates `worker_registry.last_heartbeat` and `state`. Worker states: `starting`, `ready`, `busy`, `draining`, `offline`.

### 2.3 Job Claim (Priority Polling)

The supervisor polls every 3 seconds (configurable via `POLL_INTERVAL_SECONDS`). For each enabled handler (sorted by `handler_config.priority` DESC), it calls `db.ClaimNextJob()`:

```sql
WITH next_job AS (
    SELECT job_id FROM job_queue
    WHERE (status = 'queued'
           OR (status IN ('assigned','running') AND lease_expires_at < NOW()))
      AND worker_function = $1
      AND processing_type = $2
    ORDER BY priority DESC, created_at ASC
    LIMIT 1
    FOR UPDATE SKIP LOCKED
)
UPDATE job_queue
SET status = 'assigned', assigned_worker_id = $3,
    lease_expires_at = NOW() + ($4 * INTERVAL '1 minute')
FROM next_job WHERE job_queue.job_id = next_job.job_id
RETURNING ...
```

Key behaviors:
- `FOR UPDATE SKIP LOCKED` prevents contention across workers.
- Expired leases on `assigned`/`running` jobs are automatically re-claimed.
- Default lease duration: 10 minutes, renewed at half the interval.
- Handler config is refreshed from `handler_config` table every 30 seconds.
- Polling respects GPU mode: in `exclusive` mode, only `comfyui` jobs are claimed. In `draining`/`restoring` modes, no jobs are claimed.

### 2.4 Job State Transitions

```
queued -> assigned -> running -> completed
                  \-> failed
```

| Function | SQL |
|---|---|
| `ClaimNextJob` | `SET status='assigned', assigned_worker_id=$3, lease_expires_at=...` |
| `SetJobRunning` | `SET status='running'` |
| `CompleteJob` | `SET status='completed'` |
| `FailJob` | `SET status='failed', error_info=$2` (truncated to 4000 chars) |
| `RenewLease` | `SET lease_expires_at=...` (WHERE assigned_worker_id matches) |

### 2.5 Input Manifest

Before dispatching a job, the supervisor reads input files from `cache_manifest`:

```sql
SELECT * FROM cache_manifest WHERE job_id = $1 AND direction = 'input' ORDER BY created_at ASC
```

If one input entry: uses its `cache_path` directly.
If multiple entries: uses the parent directory of the first entry's `cache_path`.

### 2.6 Output Registration

After a job completes, output files are registered in `cache_manifest`:

```sql
INSERT INTO cache_manifest (entry_id, job_id, direction, file_key, content_type, file_size_bytes, cache_path)
VALUES ($1, $2, 'output', $3, $4, $5, $6, NOW())
```

The API's `ArtifactSyncService` later downloads these from GCS and marks `synced_locally = TRUE`.

---

## 3. GPU Supervisor -> Handler Subprocess Protocol

Source: `src/gpu-supervisor/internal/handler/process.go`

The supervisor manages Python handler subprocesses as child processes. Each handler runs as an HTTP server on a fixed port.

### 3.1 Port Assignments

| Handler | Port |
|---|---|
| `transcription` | 9001 |
| `diarization` | 9002 |
| `face_extraction` | 9003 |
| `visual_tagging` | 9004 |
| `thumbnails` | 9005 |
| `transcript_analysis` | 9006 |

### 3.2 Subprocess Startup

The supervisor launches each handler with:
```
python3 -u /app/handler_server.py
```

Environment variables injected:
- `HANDLER_NAME` - handler name (e.g., `transcription`)
- `HANDLER_PORT` - port number

The supervisor waits up to 120 seconds for the handler's `/health` endpoint to return HTTP 200.

### 3.3 Health Check Endpoint

```
GET http://127.0.0.1:{port}/health
```

- Expected response: HTTP 200
- Timeout: 5 seconds per attempt, polled every 2 seconds during startup
- Used by `healthMonitorLoop` (every 30 seconds) to detect crashed handlers and auto-restart them

### 3.4 Job Dispatch Endpoint

```
POST http://127.0.0.1:{port}/handle
Content-Type: application/json
Timeout: 30 minutes
```

Request body:
```json
{
  "input_path": "/data/io-cache/<job_id>/input/<filename>",
  "output_dir": "/data/io-cache/<job_id>/output",
  "job": {
    "job_id": "uuid",
    "media_id": "uuid",
    "worker_function": "transcription",
    "payload_json": "..."
  }
}
```

Expected response (HTTP 200):
```json
{
  "artifacts": [
    {
      "type": "transcript_json",
      "path": "/data/io-cache/<job_id>/output/transcript.json",
      "size_bytes": 12345
    }
  ],
  "metadata": { ... }
}
```

The supervisor collects output files from:
1. The `artifacts[].path` values in the response (preferred).
2. Fallback: any files found in `output_dir`.
3. Last resort: serializes the response JSON itself as `{worker_function}.json`.

### 3.5 Handler Protocol (Python side)

Source: `src/workers/handlers/__init__.py`

Each handler module implements the `Handler` protocol:
```python
class Handler(Protocol):
    def init(self) -> None: ...
    def handle(self, input_path: str, output_dir: str, **kwargs: Any) -> dict[str, Any]: ...
```

The `handle()` return dict must include:
- `artifacts`: list of `{"type": str, "path": str, "size_bytes": int}` dicts
- `metadata` (optional): handler-specific structured data

### 3.6 Handler Summary

| Handler | Model/Tool | Input | Output Artifacts |
|---|---|---|---|
| `transcription` | OpenAI Whisper (`turbo`) | Audio/video file | `transcript.json`, `transcript.txt` |
| `diarization` | pyannote 3.1 | Audio/video file | `diarization.json`, per-speaker `.wav` segments |
| `face_extraction` | InsightFace buffalo_l (ONNX/CUDA) | Video file | `face_detections.json` (512-dim embeddings) |
| `visual_tagging` | ViT/CLIP via timm (CUDA) | Image file | Tag predictions with confidence scores |
| `thumbnails` | ffmpeg | Video file | `thumbnail_0.png` .. `thumbnail_5.png` (default 6) |
| `transcript_analysis` | LLM (stub) | Transcript text | Analysis JSON |

---

## 4. GPU Supervisor -> ComfyUI HTTP Protocol

Source: `src/gpu-supervisor/internal/handler/comfyui.go`

ComfyUI runs as a separate process on `http://127.0.0.1:8188`, managed directly by the supervisor (not through the handler subprocess protocol).

### 4.1 Startup

The supervisor starts ComfyUI with:
```
python3 /comfyui/main.py --listen 0.0.0.0 --port 8188 --disable-auto-launch --disable-metadata --highvram --disable-smart-memory
```

Readiness check: polls `GET /system_stats` every 5 seconds, up to 300 seconds (configurable).

### 4.2 Job Execution Flow

1. **Stage inputs**: Copy input files from GCS FUSE mount into `/comfyui/input/`.
2. **Clear stale queue**: `POST /interrupt` + `POST /queue` with `{"clear": true}`.
3. **Submit workflow**: `POST /prompt` with `{"prompt": <workflow_json>, "client_id": <job_id>}`. Returns `{"prompt_id": "..."}`.
4. **Poll for completion**: `GET /history/{prompt_id}` every 3 seconds until `outputs` appear or job timeout (default 1800 seconds).
5. **Download artifacts**: For each output item in `images`/`gifs`/`videos` arrays, `GET /view?filename=...&subfolder=...&type=output`. Validates media magic bytes before saving.
6. **Verify primary output**: First output file must exist and be non-empty (FR-040).

### 4.3 GPU Mode Transitions

When a Wan 14B workflow is detected (payload contains `"wan/"`, `"wan2"`, `"image_to_video"`, or `"start_end_frame"`), the supervisor:

1. Transitions GPU mode: `shared` -> `draining` -> `exclusive`
2. Stops all analysis handler subprocesses
3. Calls `POST /free` with `{"unload_models": true, "free_memory": true}` to release VRAM
4. Runs the ComfyUI job with full GPU access
5. After completion: `exclusive` -> `restoring` -> `shared` (restarts analysis handlers)

GPU modes:
| Mode | ComfyUI | Analysis Handlers | Job Claims |
|---|---|---|---|
| `shared` | Active (SD/SDXL) | Active | All handlers |
| `draining` | Active | Stopping | None |
| `exclusive` | Active (Wan 14B) | Stopped | comfyui only |
| `restoring` | Active | Restarting | None |

### 4.4 ComfyUI Endpoints Used

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/system_stats` | Startup readiness check |
| `POST` | `/prompt` | Submit workflow |
| `GET` | `/history/{prompt_id}` | Poll job status |
| `GET` | `/view?filename=...` | Download output artifact |
| `POST` | `/free` | Unload models, free VRAM |
| `POST` | `/interrupt` | Cancel current execution |
| `POST` | `/queue` | Clear job queue |

---

## 5. API HTTP Endpoints

Source: `src/backend/NeoVLab.Api/Controllers/`

### 5.1 Job Management (`/api/jobs`)

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/jobs?status=` | List all jobs, optionally filtered by status |
| `GET` | `/api/jobs/{jobId}` | Get single job by ID |
| `POST` | `/api/jobs` | Enqueue a job `{mediaId, jobType}` |
| `DELETE` | `/api/jobs/{jobId}` | Cancel a queued/assigned job |
| `POST` | `/api/jobs/comfy` | Submit ComfyUI workflow with full payload |

### 5.2 Worker Registry (`/api/workers`)

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/workers` | List all registered workers |
| `GET` | `/api/workers/{workerId}` | Get single worker by ID |

### 5.3 GPU Platform (`/api/gpu`)

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/gpu/status` | GPU platform health and instance count |
| `POST` | `/api/gpu/pause` | Pause GPU service (scale to zero) |
| `POST` | `/api/gpu/resume` | Resume GPU service |

### 5.4 GPU Supervisor Health (on Cloud Run, port 8080)

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Worker state, GPU status, handler list |
| `GET` | `/capabilities` | Supported job types and processing config |

---

## 6. Key Database Tables: Column Reference

### 6.1 `job_queue` (Cluster PostgreSQL)

| Column | Type | Description |
|---|---|---|
| `job_id` | TEXT PK | UUID, generated by API |
| `media_id` | TEXT | Reference to `media_assets.media_id` in local DB |
| `job_type` | TEXT | Original job type from API (e.g., `comfyui`, `wan_img2vid`) |
| `processing_type` | TEXT | `cpu` or `gpu` |
| `worker_function` | TEXT | Canonical handler name (e.g., `comfyui`, `transcription`) |
| `status` | TEXT | `queued`, `assigned`, `running`, `completed`, `failed`, `cancelled` |
| `assigned_worker_id` | TEXT | FK to `worker_registry.worker_id` |
| `lease_expires_at` | TIMESTAMPTZ | Lease deadline; expired leases allow re-claim |
| `priority` | INTEGER | 1-10, higher wins. Default 5, ComfyUI defaults to 10 in handler_config |
| `payload_json` | TEXT | Handler-specific JSON (e.g., ComfyUI workflow + options) |
| `error_info` | TEXT | Error message on failure (max 4000 chars) |
| `created_at` | TIMESTAMPTZ | Job creation time |
| `updated_at` | TIMESTAMPTZ | Last status change |

Indexes: `status`, `worker_function`, `assigned_worker_id`, composite routing index on `(status, worker_function, processing_type, priority DESC, created_at ASC)`.

### 6.2 `worker_registry` (Cluster PostgreSQL)

| Column | Type | Description |
|---|---|---|
| `worker_id` | TEXT PK | Unique worker identifier |
| `processing_type` | TEXT | `cpu` or `gpu` |
| `worker_function` | TEXT | Primary handler function |
| `state` | TEXT | `starting`, `ready`, `busy`, `draining`, `offline` |
| `last_heartbeat` | TIMESTAMPTZ | Updated every 30 seconds |
| `capabilities_json` | TEXT | JSON with max_concurrent, worker_functions, gpu_mode |
| `registered_at` | TIMESTAMPTZ | First registration time |

### 6.3 `cache_manifest` (Cluster PostgreSQL)

| Column | Type | Description |
|---|---|---|
| `entry_id` | TEXT PK | UUID |
| `job_id` | TEXT FK | References `job_queue.job_id` |
| `direction` | TEXT | `input` or `output` |
| `file_key` | TEXT | Logical filename |
| `content_type` | TEXT | MIME type |
| `file_size_bytes` | BIGINT | File size |
| `cache_path` | TEXT | GCS FUSE mount path (e.g., `/data/io-cache/<job_id>/input/file.mp4`) |
| `synced_locally` | BOOLEAN | TRUE after API downloads artifact from GCS |
| `created_at` | TIMESTAMPTZ | Entry creation |
| `synced_at` | TIMESTAMPTZ | When API synced the file locally |
| `evicted_at` | TIMESTAMPTZ | When worker-side cache was evicted |
| `gcs_evicted_at` | TIMESTAMPTZ | When GCS object was deleted after sync |

### 6.4 `handler_config` (Cluster PostgreSQL)

| Column | Type | Description |
|---|---|---|
| `worker_function` | TEXT PK | Handler name (matches `job_queue.worker_function`) |
| `enabled` | BOOLEAN | Whether the handler accepts new jobs |
| `priority` | INTEGER | Polling priority (higher = polled first). ComfyUI=10, others=5 |
| `updated_at` | TIMESTAMPTZ | Last config change |

The supervisor checks this table every 30 seconds. If a handler is missing from the table, it is treated as enabled (opt-out model).
