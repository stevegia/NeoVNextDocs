# Database Schema: Cluster & Local

This document provides entity-relationship diagrams (ERDs) for both the **Cluster Database** (PostgreSQL on Cloud SQL) and the **Local API Database** (SQLite).

## Cluster Database (PostgreSQL)

The cluster database is shared across all Cloud Run workers and handles distributed job queueing, worker coordination, and artifact caching.

### Core Tables ERD

```mermaid
erDiagram
    job_queue ||--o{ cache_manifest : "has artifacts"
    job_queue ||--o{ thumbnails : "produces"
    job_queue ||--o{ face_detections : "produces"
    job_queue ||--o{ transcriptions : "produces"
    job_queue ||--o{ speaker_segments : "produces"
    job_queue ||--o{ staged_inputs : "may stage input for"
    worker_registry ||--o{ job_queue : "claims jobs"
    
    job_queue {
        text job_id PK
        text media_id
        text job_type "thumbnails | transcription | comfyui"
        text processing_type "cpu | gpu"
        text worker_function "matches handler name"
        text status "queued | assigned | running | completed | failed"
        text assigned_worker_id FK
        timestamptz lease_expires_at "job timeout, reclaimed if expired"
        int priority "higher first"
        text payload_json "handler kwargs"
        text error_info "JSON error details"
        timestamptz created_at
        timestamptz updated_at
    }
    
    worker_registry {
        text worker_id PK
        text processing_type "cpu | gpu"
        text worker_function "transcription | comfyui | etc"
        text state "starting | ready | busy | draining | offline"
        timestamptz last_heartbeat "updated every 30s"
        text capabilities_json "max_concurrent, features"
        timestamptz registered_at
    }
    
    cache_manifest {
        text entry_id PK
        text job_id FK
        text direction "input | output"
        text file_key "artifact filename"
        text content_type "MIME type"
        bigint file_size_bytes
        text cache_path "local path or gs:// URI"
        boolean synced_locally "true if backend downloaded"
        timestamptz created_at
        timestamptz synced_at "when backend fetched it"
        timestamptz evicted_at "local worker cache cleared"
        timestamptz gcs_evicted_at "GCS object deleted"
    }
    
    staged_inputs {
        text staging_id PK
        text job_id "optional job association"
        text file_path "absolute path on disk"
        text filename "original filename"
        timestamptz expires_at "TTL for cleanup"
        timestamptz staged_at
        timestamptz accessed_at "last retrieval"
        boolean is_cleaned_up "garbage collected"
    }
```

### Artifact Metadata Tables ERD

```mermaid
erDiagram
    job_queue ||--o{ thumbnails : "generates"
    job_queue ||--o{ face_detections : "extracts"
    job_queue ||--o{ transcriptions : "transcribes"
    job_queue ||--o{ speaker_segments : "diarizes"
    
    thumbnails {
        text thumbnail_id PK
        text media_id
        text job_id FK
        int sequence_number "0-indexed keyframe"
        double timestamp_seconds
        text thumbnail_path "gs:// or local path"
        int width_px
        int height_px
        bigint file_size_bytes
        timestamptz generated_at
    }
    
    face_detections {
        text detection_id PK
        text media_id
        text job_id FK
        int frame_number
        real timestamp_seconds
        jsonb bounding_box "{x, y, width, height}"
        vector_512 embedding "pgvector: InsightFace 512-dim"
        text embedding_model "insightface_buffalo_l"
        real detection_confidence "0.0 to 1.0"
        int estimated_age
        text estimated_gender "male | female"
        text person_id "future: cross-video identity"
        timestamptz detected_at
    }
    
    transcriptions {
        text transcription_id PK
        text media_id
        text job_id FK
        text full_text "complete transcript"
        text language "ISO 639-1 code"
        int word_count
        text model_used "openai/whisper-large-v3"
        jsonb segments "array of {text, start, end, confidence}"
        tsvector full_text_tsvector "FTS index"
        timestamptz transcribed_at
    }
    
    speaker_segments {
        text segment_id PK
        text media_id
        text job_id FK
        text speaker_label "SPEAKER_00, SPEAKER_01..."
        real start_seconds
        real end_seconds
        real duration_seconds "computed column"
        text audio_path "extracted segment wav file"
        text diarization_model "pyannote/speaker-diarization-3.1"
        real confidence
        vector_192 voice_embedding "pgvector: 192-dim speaker embedding"
        text person_id "future: speaker re-identification"
        timestamptz detected_at
    }
```

### Indexes & Performance

Key indexes on cluster DB tables:

| Table | Index | Purpose |
|-------|-------|---------|
| `job_queue` | `idx_job_queue_routing` | Composite: (status, worker_function, processing_type, priority DESC, created_at ASC) for fast claim_next_job() |
| `job_queue` | `idx_job_queue_status` | Filter by status='queued' |
| `job_queue` | `idx_job_queue_assigned_worker` | Track worker assignments |
| `cache_manifest` | `idx_cache_manifest_eviction` | Find evictable entries WHERE synced_locally=TRUE AND evicted_at IS NULL |
| `cache_manifest` | `idx_cache_manifest_gcs_eviction` | Find GCS objects to delete WHERE synced_locally=TRUE AND gcs_evicted_at IS NULL |
| `worker_registry` | `idx_worker_registry_type` | Composite: (processing_type, worker_function, state) for worker discovery |
| `transcriptions` | `idx_transcriptions_fts` | GIN index on full_text_tsvector for fast text search |
| `face_detections` | (pgvector ivfflat) | Future: vector similarity search for face recognition |

## Local Database (SQLite)

The local API database runs on the backend API container and stores media library, job routing policies, and GPU platform configuration.

### Media & Jobs ERD

```mermaid
erDiagram
    media_assets ||--o{ media_assets_fts : "indexed by"
    scan_sessions ||--o{ media_assets : "discovers"
    
    media_assets {
        text media_id PK
        text file_name
        text file_path "absolute path on disk"
        integer file_size_bytes
        text mime_type "video/mp4 | audio/mpeg"
        real duration_seconds "from ffprobe"
        text created_at "ISO 8601"
        text updated_at
        integer width "video resolution"
        integer height
        text codec "h264 | hevc"
        text modified_at "file mtime"
        text file_hash "SHA256, prevents duplicates"
    }
    
    media_assets_fts {
        text media_id
        text file_name "FTS indexed"
        text file_path "FTS indexed"
    }
    
    scan_sessions {
        text session_id PK
        text status "pending | scanning | completed | failed"
        text root_path "directory being scanned"
        integer discovered_count
        text created_at
        text completed_at
    }
```

### GPU Platform & Execution Policies ERD

```mermaid
erDiagram
    gpu_platform_config ||--o{ model_registry : "manages models for"
    gpu_platform_config ||--o{ execution_policies : "routes jobs via"
    execution_policies ||--o{ job_routing_log : "logs decisions"
    gpu_platform_config ||--|| budget_tracking : "enforces spending"
    
    model_registry {
        text model_id PK
        text name "sd_xl_base_1.0.safetensors"
        text model_type "checkpoints | loras | vae"
        text filename
        text volume_path "Filestore NFS path"
        integer size_bytes
        text source_url "civitai or huggingface"
        integer is_available "boolean: verified on NFS"
        text last_verified_at
        text registered_at
    }
    
    gpu_platform_config {
        text config_id PK
        integer enabled "boolean: GPU jobs allowed"
        text platform "gcp | aws | azure"
        text endpoint_id "Cloud Run service URL"
        text gpu_type "nvidia-l4 | nvidia-t4"
        integer max_workers "concurrent GPU instances"
        integer idle_timeout_minutes "scale-to-zero after idle"
        text model_volume_id "Filestore instance name"
        text updated_at
    }
    
    execution_policies {
        text policy_id PK
        text job_type "thumbnails | comfyui"
        text execution_mode "cloud"
        text gpu_tier "l4 | t4"
        integer requires_gpu "boolean"
        text created_at
        text updated_at
    }
    
    budget_tracking {
        text period_id PK "YYYY-MM | daily-YYYY-MM-DD"
        integer budget_cents "monthly dollar limit * 100"
        integer spent_cents "current period spend"
        integer remote_jobs_dispatched "CPU job count"
        real remote_minutes_used "CPU minutes"
        integer cap_reached "boolean"
        text cap_reached_at "when budget exhausted"
        text updated_at
        integer gpu_budget_cents "GPU-specific limit"
        integer gpu_spent_cents "cumulative GPU cost"
        integer gpu_jobs_dispatched
        real gpu_active_seconds "billable GPU time"
        integer gpu_cap_reached "boolean"
        text gpu_cap_reached_at
        integer gpu_warning_sent "sent warning at 80%"
    }
    
    job_routing_log {
        text log_id PK
        text job_id
        text job_type
        text policy_name "execution_policies.job_type"
        text policy_mode "execution_mode"
        text decision "dispatched | rejected | fallback"
        text reason "budget_ok | cap_reached | no_gpu"
        text worker_id "assigned worker"
        text dispatched_at
    }
```

## Database Access Patterns

### Cluster DB (PostgreSQL)

**Connection Methods:**

| Client | Connection String | Credentials |
|--------|-------------------|-------------|
| **Cloud Run GPU Worker** | `postgresql://user:pass@/neovlab_cluster?host=/cloudsql/neovnext:us-east4:neovlab-cluster` | Cloud SQL Proxy Unix socket |
| **Backend API (CloudDev)** | `postgresql://user:pass@34.21.76.69:5432/neovlab_cluster` | Public IP + password |
| **Migrations** | `psql $DATABASE_URL < cluster/001_cluster_schema.sql` | Apply via scripts/apply_cloud_schema.py |

**Client Library:** `psycopg2` (Python workers), `Npgsql` (C# backend)

### Local DB (SQLite)

**Location:** `/data/neovlab.db` in backend API container  
**Connection String:** `Data Source=/data/neovlab.db`  
**Client Library:** `Microsoft.Data.Sqlite` (C# backend)

**Backup Strategy:** Docker volume `neovlab-api-data` persists across container restarts

## Migration History

### Cluster DB Migrations (PostgreSQL)

| Migration | Description | Tables Modified |
|-----------|-------------|-----------------|
| `001_cluster_schema.sql` | Initial schema: job_queue, worker_registry, cache_manifest | 3 new tables |
| `002_gcs_eviction.sql` | Add gcs_evicted_at column for GCS object deletion tracking | cache_manifest |
| `003_thumbnails.sql` | Thumbnail metadata storage | 1 new table |
| `004_staged_inputs.sql` | ComfyUI input staging (replace in-memory dict) | 1 new table |
| `005_face_detections.sql` | Face detection metadata with pgvector embeddings | 1 new table, pgvector extension |
| `006_transcriptions.sql` | Whisper transcription storage with FTS | 1 new table, tsvector trigger |
| `007_speaker_segments.sql` | Diarization speaker segments with voice embeddings | 1 new table |

### Local DB Migrations (SQLite)

| Migration | Description | Tables Modified |
|-----------|-------------|-----------------|
| `001_local_api_schema.sql` | Complete local schema (media, GPU, budgets, policies) | 9 new tables |

## Data Residency & Cleanup

### Cluster DB Cleanup Strategies

- **Job Queue:** Jobs remain indefinitely; manual admin cleanup required
- **Cache Manifest:** 
  - `evicted_at` marks local worker cache eviction (when disk space needed)
  - `gcs_evicted_at` marks GCS object deletion (after backend downloads)
  - TTL: 7 days after synced_locally=TRUE
- **Staged Inputs:** Auto-cleanup via `cleanup_expired_staged_inputs()` function (TTL: 1 hour)
- **Worker Registry:** Workers mark state='offline' on shutdown, rows persist

### Local DB Cleanup Strategies

- **Media Assets:** Manual deletion only (user-initiated)
- **Scan Sessions:** Persist indefinitely (audit log)
- **Budget Tracking:** One row per period (YYYY-MM), resets monthly
- **Job Routing Log:** Grows unbounded, manual cleanup required

## Schema Anti-Patterns to Note

### Why No Foreign Keys Between Databases?

`job_queue.media_id` references `media_assets.media_id`, but they're in **different databases** (Cluster PostgreSQL vs. Local SQLite). This violates referential integrity — if a media asset is deleted locally, associated jobs in cluster DB become orphaned.

**Mitigation:** Backend API must enforce orphan cleanup manually (not currently implemented).

### Why JSONB for Structured Data?

Several tables use JSONB for semi-structured data:
- `job_queue.payload_json` (handler kwargs)
- `face_detections.bounding_box` (x, y, width, height)
- `transcriptions.segments` (array of {text, start, end})

**Rationale:** Flexibility for handler-specific parameters without schema migration per handler type.

**Trade-off:** No type safety, harder to query individual fields (requires JSON operators).

### Why Text Timestamps in SQLite?

Local DB uses `TEXT` for timestamps (ISO 8601 strings) instead of `INTEGER` (Unix epoch).

**Rationale:** SQLite date functions prefer text dates; Dapper can map to C# `DateTime` from ISO 8601.

**Trade-off:** Larger storage (23 bytes vs. 8 bytes), slower comparisons.

## Vector Search (pgvector)

Two tables use pgvector for similarity search:

| Table | Column | Dimensions | Model | Use Case |
|-------|--------|------------|-------|----------|
| `face_detections` | `embedding` | 512 | InsightFace buffalo_l | Face recognition / clustering |
| `speaker_segments` | `voice_embedding` | 192 | Custom speaker embedding | Speaker re-identification |

**Index Strategy:** IVFFlat (Inverted File with Flat compression) for approximate nearest neighbor (ANN) search.

**Query Pattern:**
```sql
SELECT * FROM face_detections
ORDER BY embedding <-> '[0.123, 0.456, ...]'::vector(512)
LIMIT 10;
```

**Status:** Indexes not yet created (manual `CREATE INDEX USING ivfflat` required after enough data accumulated).

---

**Generated:** Auto-documentation of NeoVNext database schemas  
**See Also:** 
- [worker-architecture.md](worker-architecture.md) for how workers interact with these tables
- [gcp-infrastructure.md](gcp-infrastructure.md) for Cloud SQL deployment details
