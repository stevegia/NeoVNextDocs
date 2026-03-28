# NeoVLab DB & Schema Gap Analysis

**Date:** 2026-03-26
**Scope:** All SQL migrations, Python cluster_db.py, Go db/ package, C# Dapper repositories
**Source of truth:** Migration files in `src/data/migrations/`; docs are secondary

---

## 1. Current Schema Inventory

### 1.1 Local SQLite (desktop API)

Source: `src/data/migrations/001_local_api_schema.sql`

| Table | PK Type | Key Columns | Purpose |
|---|---|---|---|
| `media_assets` | TEXT | file_path, file_hash, mime_type, duration_seconds, width, height, codec | Media library catalog |
| `media_assets_fts` | virtual (FTS5) | file_name, file_path | Full-text search over media_assets; kept in sync via triggers |
| `scan_sessions` | TEXT | root_path, status, discovered_count, completed_at | Filesystem scan history |
| `model_registry` | TEXT | filename, volume_path, size_bytes, source_url, is_available | GPU model file registry |
| `gpu_platform_config` | TEXT | enabled, gpu_type, max_workers, idle_timeout_minutes, model_volume_id | Single-row GCP Cloud Run config |
| `execution_policies` | TEXT | job_type (UNIQUE), execution_mode, gpu_tier, requires_gpu | Per-job-type routing rules |
| `budget_tracking` | TEXT (period_id) | budget_cents, spent_cents, gpu_budget_cents, gpu_spent_cents, gpu_cap_reached | Monthly spend + GPU cap tracking |
| `job_routing_log` | TEXT | job_id, job_type, policy_name, decision, reason, worker_id, evaluated_at | Immutable routing audit log |
| `tier_config` | TEXT (tier_id) | tier_name, display_name, retention_days, auto_sync | Storage tier definitions; seeded with 'io-cache' + 'local-storage' |
| `job_artifacts` | TEXT | job_id, media_id, artifact_type, file_path, storage_domain, sync_eligible, is_primary | Local artifact registry |
| `staged_inputs` | TEXT (staging_id) | job_id, file_path, filename, expires_at, is_cleaned_up | ComfyUI input staging (SQLite mirror of cluster table) |

**Indexes:** file_path (media_assets), file_hash partial, unhashed partial, job_id + evaluated_at (routing_log), job_id + media_id (job_artifacts), expires_at partial + job_id partial (staged_inputs).

---

### 1.2 Cluster PostgreSQL (Cloud SQL, shared with GPU workers)

#### Migration 001 — `001_cluster_schema.sql`

| Table | PK Type | Key Columns | Purpose |
|---|---|---|---|
| `job_queue` | TEXT | media_id, job_type, processing_type, worker_function, status, assigned_worker_id, lease_expires_at, priority, payload_json, error_info | Central job queue; pull-model with FOR UPDATE SKIP LOCKED |
| `cache_manifest` | TEXT | job_id (FK→job_queue CASCADE), direction, file_key, content_type, file_size_bytes, cache_path, synced_locally, synced_at, evicted_at | GCS FUSE input/output file tracking |
| `worker_registry` | TEXT | processing_type, worker_function, state, last_heartbeat, capabilities_json | Worker heartbeat + capability registry |

**Indexes:** status, worker_function, assigned_worker_id, composite routing (status, worker_function, processing_type, priority DESC, created_at ASC) on job_queue; job_id, direction, synced_at, eviction partial on cache_manifest; state, worker_function, composite type+function+state on worker_registry.

#### Migration 002 — `002_gcs_eviction.sql`

ALTER TABLE cache_manifest adds: `gcs_evicted_at TIMESTAMPTZ` + index on (synced_locally, synced_at) WHERE gcs_evicted_at IS NULL.

#### Migration 003 — `003_thumbnails.sql`

| Table | PK | Key Columns | Purpose |
|---|---|---|---|
| `thumbnails` | TEXT (gen_random_uuid) | media_id, job_id (FK→job_queue CASCADE), sequence_number, timestamp_seconds, thumbnail_path, width_px, height_px, file_size_bytes | Per-video thumbnail metadata |

**Indexes:** media_id, job_id, UNIQUE (media_id, sequence_number).

#### Migration 004 — `004_staged_inputs.sql`

| Table | PK | Key Columns | Purpose |
|---|---|---|---|
| `staged_inputs` | TEXT | job_id, file_path, filename, expires_at, accessed_at, is_cleaned_up | Durable ComfyUI input staging; replaces in-memory dict |

Includes PL/pgSQL `cleanup_expired_staged_inputs()` function. **No scheduled invocation** exists anywhere in the codebase — this function is never called.

#### Migration 005 — `005_face_detections.sql`

Enables `CREATE EXTENSION IF NOT EXISTS vector`.

| Table | PK | Key Columns | Purpose |
|---|---|---|---|
| `face_detections` | TEXT (gen_random_uuid) | media_id, job_id (FK→job_queue CASCADE), frame_number, timestamp_seconds, bounding_box (JSONB), embedding vector(512), embedding_model, detection_confidence, estimated_age, estimated_gender, **person_id TEXT** | Per-frame face detections with pgvector embeddings |

**Indexes:** (media_id, timestamp_seconds), job_id, (media_id, frame_number). No IVFFlat or HNSW vector index.

#### Migration 006 — `006_transcriptions.sql`

| Table | PK | Key Columns | Purpose |
|---|---|---|---|
| `transcriptions` | TEXT (gen_random_uuid) | media_id, job_id (FK→job_queue CASCADE), full_text, language, word_count, model_used, segments (JSONB), full_text_tsvector | Whisper output with full-text search |

Trigger `trg_transcriptions_tsvector` auto-updates `full_text_tsvector` on INSERT/UPDATE of full_text or language.

#### Migration 007 — `007_speaker_segments.sql`

| Table | PK | Key Columns | Purpose |
|---|---|---|---|
| `speaker_segments` | TEXT (gen_random_uuid) | media_id, job_id (FK→job_queue CASCADE), speaker_label, start_seconds, end_seconds, duration_seconds (GENERATED STORED), audio_path, diarization_model, confidence, **voice_embedding vector(192)**, **person_id TEXT** | Pyannote speaker segments |

**Indexes:** media_id, job_id, speaker_label, (media_id, start_seconds). No vector index on voice_embedding.

#### Migration 008 — `008_handler_config.sql`

| Table | PK | Key Columns | Purpose |
|---|---|---|---|
| `handler_config` | TEXT (worker_function) | enabled, priority, updated_at | Runtime enable/disable + priority for each handler |

Seeded with: comfyui (priority 10), face_extraction, visual_tagging, transcription, diarization, thumbnails, transcript_analysis (all priority 5).

#### Migration 009 — `009_persons.sql`

| Table | PK | Key Columns | Purpose |
|---|---|---|---|
| `persons` | TEXT (gen_random_uuid) | name, face_thumbnail, voice_sample_path, face_embedding vector(512), voice_embedding vector(192), auto_generated | Identity records for face/voice persona pipeline |
| `person_appearances` | TEXT (gen_random_uuid) | person_id (FK→persons CASCADE), media_id, first_seen_sec, last_seen_sec, detection_count, source ('face'\|'voice'\|'both') | Per-media appearance records for a person |

**Indexes:** person_id, media_id on person_appearances. No vector indexes on persons.face_embedding or persons.voice_embedding.

---

## 2. Schema Gaps

### 2.1 Undocumented `job_queue` status values in active use

The migration DDL defines no CHECK constraint on `job_queue.status`. The documented lifecycle is:
`queued -> assigned -> running -> completed -> failed`

The C# `ArtifactSyncService` writes two additional statuses that appear nowhere in migrations or contracts.md:

| Undocumented status | Written by | When |
|---|---|---|
| `synced` | `ArtifactSyncService.SyncCompletedJobAsync` | After all output artifacts are copied locally |
| `failed_acked` | `ArtifactSyncService.SyncFailedJobAsync` | After failed job is acknowledged and SignalR emitted |

`GetErrorRateStatsAsync` in `JobQueueRepository` also queries for `failed_acked` in its COUNT. Neither status is seeded in handler_config, documented in contracts.md, or mentioned in the migration. The Go supervisor's `ClaimNextJob` does not reclaim `synced` or `failed_acked` jobs (correct), but the omission from DDL means there is no constraint preventing typos from creating phantom statuses.

**Additionally:** `ArtifactSyncService.PollClusterCompletionsAsync` queries for both `"completed"` AND `"complete"` (the latter is a typo alias, line 147). The Go supervisor only ever writes `"completed"`. The `"complete"` query is dead code but adds an unnecessary DB round-trip per poll cycle.

### 2.2 `cache_manifest.UpsertAsync` does not write `gcs_evicted_at`

`CacheManifestRepository.UpsertAsync` (C#) has an INSERT...ON CONFLICT that explicitly lists all columns to update — but does not include `gcs_evicted_at` in either the INSERT column list or the ON CONFLICT SET clause. `gcs_evicted_at` was added by migration 002, after the upsert was written. A conflict on an existing entry would silently drop a previously recorded `gcs_evicted_at` value, causing the GCS eviction sweep to re-attempt deletion of an already-evicted object.

### 2.3 `face_detections.person_id` and `speaker_segments.person_id` — no FK, no index

Both `person_id TEXT` columns exist in the DDL (migrations 005 and 007 respectively) as nullable placeholders. Migration 009 now creates the `persons` table that these are meant to reference, but:

- Neither column has a FK constraint to `persons(person_id)`. Orphaned person_id values will accumulate silently when persons are deleted.
- Neither column has an index. The `update_face_detection_person_id()` function in cluster_db.py issues `UPDATE face_detections WHERE detection_id = %s` (single-row, fine), but the `find_similar_faces` / persona pipeline will need to query `WHERE person_id IS NULL` across potentially large tables without an index.

### 2.4 `persons` table — no vector indexes, no `updated_at` trigger

Migration 009 creates `persons` with `face_embedding vector(512)` and `voice_embedding vector(192)`, and `find_similar_persons_by_face` in cluster_db.py does a cosine distance scan against `face_embedding`. With no IVFFlat or HNSW index, this is a full sequential scan. The same issue applies to `face_detections.embedding` and `speaker_segments.voice_embedding`. The next-features.md doc acknowledges IVFFlat is needed but defers it to >1000 vectors; at that scale a sequential scan is still viable, but there is no migration that adds these indexes and no mechanism to trigger migration after the threshold is crossed.

The `persons` table also has no trigger to auto-update `updated_at` on row modification (unlike transcriptions which has a tsvector trigger). The `create_person` function correctly sets both on INSERT; however there is no `update_person` function in cluster_db.py at all, meaning any future UPDATE to a person record will not refresh `updated_at` unless the caller explicitly does so.

### 2.5 `person_appearances` — no UNIQUE constraint

`person_appearances` has no UNIQUE constraint on `(person_id, media_id, source)`. The identity resolution pipeline (not yet implemented) will aggregate face detections and speaker segments per media item and write appearances. Without a uniqueness constraint, re-running clustering after adding new detections will insert duplicate rows rather than upsert. The next-features.md proposal includes `first_seen_sec`, `last_seen_sec`, and `detection_count`, all of which should be updatable via ON CONFLICT DO UPDATE — but there is no target for the conflict clause.

### 2.6 `staged_inputs` exists in both SQLite and PostgreSQL with incompatible schemas

| Column | SQLite (`001_local_api_schema.sql`) | PostgreSQL (`004_staged_inputs.sql`) |
|---|---|---|
| `is_cleaned_up` | `INTEGER NOT NULL DEFAULT 0` | `BOOLEAN NOT NULL DEFAULT FALSE` |
| Partial indexes | `WHERE is_cleaned_up = 0` | `WHERE is_cleaned_up = FALSE` |
| `accessed_at` | `TEXT` | `TIMESTAMPTZ` |
| Cleanup function | None | `cleanup_expired_staged_inputs()` PL/pgSQL |

The SQLite version uses INTEGER for boolean (SQLite norm); the PostgreSQL version uses BOOLEAN. This is not a bug — they serve different concerns — but no code in the C# API appears to use the cluster `staged_inputs` table directly. `InputStagingService` writes to the SQLite version. The cluster table was introduced in migration 004 to support horizontal API scaling, but there is no C# repository or service that reads or writes to it. It is dead schema.

### 2.7 `cleanup_expired_staged_inputs()` PostgreSQL function is never called

The PL/pgSQL function in migration 004 marks expired staged_inputs rows as cleaned up. No Go code, Python code, or C# code in the codebase calls it. Expired rows accumulate indefinitely.

### 2.8 `handler_config` seed does not include future handlers

Migration 008 seeds: comfyui, face_extraction, visual_tagging, transcription, diarization, thumbnails, transcript_analysis. The next-features plan calls for: `face_clustering` (port 9007), `voice_embedding` (port 9008), `identity_resolution`. These are absent from the seed data. The Go supervisor's opt-out model means missing rows default to enabled, so new handlers will run without any priority configuration — they will default to 5 via Go's fallback, but the cluster-side `handler_config` table will be out of sync with reality.

### 2.9 `thumbnail_path` vs. actual GCS/FUSE path in `thumbnails` table

`thumbnails.thumbnail_path` is TEXT NOT NULL. The Python `insert_thumbnail` writes the value from the handler's `thumbnail_path` field. In Cloud Run, this will be a GCS FUSE mount path (e.g., `/data/io-cache/<job_id>/output/thumbnail_0.png`) which is not directly accessible from the desktop API. The thumbnail records in the cluster DB are therefore only queryable for path; actual retrieval goes through `cache_manifest` + `ArtifactSyncService`. There is no FK from `thumbnails.job_id` to `cache_manifest` nor any query in the C# API that reads from the cluster `thumbnails` table. Thumbnail rows are written by the Python handler via `insert_thumbnail` but never read by the API — the API reads thumbnails through `job_artifacts` (SQLite) after `ArtifactSyncService` copies them. The cluster `thumbnails` table is produced but not consumed.

---

## 3. Cross-Layer Inconsistencies

### 3.1 Go models.go omits `gcs_evicted_at`

`CacheEntry` in `src/gpu-supervisor/internal/db/models.go` does not include a `GcsEvictedAt` field. This is fine for the Go supervisor's use case — it only reads input manifests and writes output entries, never touches eviction fields — but it means the Go layer's `CacheEntry` struct is silently incomplete relative to the actual table schema.

### 3.2 Go `RegisterOutputArtifact` does not pass `synced_locally`

`db.RegisterOutputArtifact` in `jobs.go` does not include `synced_locally` in the INSERT column list (it relies on the DEFAULT FALSE). The C# `CacheManifestRepository.UpsertAsync` does include it explicitly. Behaviorally equivalent, but the omission means the column's intent is less obvious when reading the Go code.

### 3.3 Python `insert_speaker_segment` never writes `voice_embedding`

`ClusterDbClient.insert_speaker_segment` in `cluster_db.py` accepts no `voice_embedding` parameter and does not include it in the INSERT. The column exists in the migration (007) and is read back by `get_speaker_segments_for_media` (`voice_embedding::TEXT`). The diarization handler writes speaker segments but the embedding is never populated by any current code. `voice_embedding` is always NULL in production.

This is called out in next-features.md ("No voice embedding extraction"), but the gap is: the schema is ready, the read path exists in cluster_db.py, but the write path (`insert_speaker_segment`) has no parameter for it. A future `voice_embedding` handler would need to either add a parameter to `insert_speaker_segment` or add a separate `update_speaker_segment_voice_embedding()` function.

### 3.4 Python `search_transcriptions` hardcodes `'english'` tsquery config

`search_transcriptions` calls `plainto_tsquery('english', %s)` unconditionally. The tsvector trigger in migration 006 correctly uses the language column to select a regconfig (english, spanish, french, etc.). The mismatch means non-English transcriptions are indexed with their correct language config but queried against the English dictionary, degrading search quality for non-English content.

### 3.5 C# `ArtifactSyncService` uses `"complete"` status alias

As noted in §2.1: `PollClusterCompletionsAsync` queries for both `"completed"` and `"complete"`. The Go supervisor writes only `"completed"`. The Python `cluster_db.py` writes only `"completed"`. The `"complete"` query is dead — it will never return rows — but it is not harmless: it executes a COUNT/SELECT against the cluster DB on every 10-second poll cycle.

### 3.6 C# `JobQueueService.ResolveProcessingType` includes `transcript_analysis` as CPU, but `handler_config` has no CPU handler distinction

`transcript_analysis` resolves to `processing_type = 'cpu'` in C#. The `handler_config` table is only checked by the Go supervisor, which polls using both `worker_function` AND `processing_type` in `ClaimNextJob`. A CPU supervisor would need to exist for `transcript_analysis` and `thumbnails` jobs to be claimed. Currently both job types route to the same GPU Cloud Run worker, which accepts `processing_type = 'cpu'` jobs because the worker registers with `"processing_type": "gpu"` in capabilities but its `ClaimNextJob` calls pass the per-handler `processing_type` from `handler_config`... which does not store `processing_type`. The actual `processing_type` filter in `ClaimNextJob` comes from the supervisor's startup config, not `handler_config`. The Go supervisor in practice claims all `processing_type` values it is configured for. This works today but creates ambiguity: `handler_config` has no `processing_type` column, so there is no cluster-side distinction between CPU and GPU handlers.

### 3.7 `job_artifacts.storage_domain` (SQLite) vs `tier_config.tier_name` — stale column name

`job_artifacts` has a `storage_domain TEXT` column. `tier_config` replaced the legacy `storage_domains` table (the migration comment says "replaces legacy storage_domains"). `StorageDomainResolver.cs` maps artifact types to storage domain strings. The column name `storage_domain` is a legacy name; the current tier system uses `tier_name`. No FK exists between `job_artifacts.storage_domain` and `tier_config.tier_name`. This is cosmetic but means joins between the two tables require a manual equality condition rather than a FK constraint.

---

## 4. Proposed Schema Changes

This section describes what migration 009 delivers, what it is missing, and what additional migrations are needed.

### 4.1 Migration 009 as written — what it does and what it is missing

Migration 009 (`009_persons.sql`) correctly creates `persons` and `person_appearances`. It is minimal and functional. What it does not do:

**Missing from 009:**

```sql
-- 1. FK constraints from existing tables back to persons
ALTER TABLE face_detections
    ADD CONSTRAINT fk_face_detections_person
    FOREIGN KEY (person_id) REFERENCES persons(person_id) ON DELETE SET NULL;

ALTER TABLE speaker_segments
    ADD CONSTRAINT fk_speaker_segments_person
    FOREIGN KEY (person_id) REFERENCES persons(person_id) ON DELETE SET NULL;

-- 2. Indexes for person_id lookups on child tables
CREATE INDEX IF NOT EXISTS idx_face_detections_person_id
    ON face_detections(person_id) WHERE person_id IS NOT NULL;

CREATE INDEX IF NOT EXISTS idx_speaker_segments_person_id
    ON speaker_segments(person_id) WHERE person_id IS NOT NULL;

-- 3. UNIQUE constraint on person_appearances to allow upserts
CREATE UNIQUE INDEX IF NOT EXISTS idx_person_appearances_unique
    ON person_appearances(person_id, media_id, source);

-- 4. updated_at trigger on persons
CREATE OR REPLACE FUNCTION persons_set_updated_at() RETURNS trigger AS $$
BEGIN
    NEW.updated_at := NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_persons_updated_at
    BEFORE UPDATE ON persons
    FOR EACH ROW EXECUTE FUNCTION persons_set_updated_at();
```

### 4.2 Required: Migration 010 — vector indexes (deferred trigger)

The project intends to add IVFFlat indexes after >1000 vectors are accumulated. Since there is no automatic migration trigger for data volume thresholds, this migration should be authored now with a `CREATE INDEX CONCURRENTLY` (safe on a live table) and applied manually when ready. Suggested content:

```sql
-- 010_vector_indexes.sql
-- Apply after face_detections has 1000+ rows and speaker_segments has 1000+ rows.
-- CONCURRENTLY allows creation without locking; cannot run inside a transaction block.

CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_face_detections_embedding_ivfflat
    ON face_detections USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);

CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_speaker_segments_voice_embedding_ivfflat
    ON speaker_segments USING ivfflat (voice_embedding vector_cosine_ops)
    WITH (lists = 50);

CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_persons_face_embedding_ivfflat
    ON persons USING ivfflat (face_embedding vector_cosine_ops)
    WITH (lists = 50);

CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_persons_voice_embedding_ivfflat
    ON persons USING ivfflat (voice_embedding vector_cosine_ops)
    WITH (lists = 20);
```

`lists` values above are starting estimates; tune to `sqrt(row_count)` at application time.

### 4.3 Required: Migration 011 — status constraint + synced/failed_acked formalization

```sql
-- 011_job_queue_status_constraint.sql
-- Formalizes all valid job_queue.status values including API-side statuses.

ALTER TABLE job_queue
    ADD CONSTRAINT chk_job_queue_status CHECK (
        status IN ('queued', 'assigned', 'running', 'completed', 'failed', 'cancelled', 'synced', 'failed_acked')
    );
```

### 4.4 Required: fix `CacheManifestRepository.UpsertAsync` to include `gcs_evicted_at`

This is a code fix, not a migration. The ON CONFLICT SET clause in `CacheManifestRepository.UpsertAsync` must add:

```sql
gcs_evicted_at = EXCLUDED.gcs_evicted_at
```

and the INSERT column list must include `gcs_evicted_at` with the value from the model.

### 4.5 Recommended: Migration 012 — `handler_config` additions for future handlers

```sql
-- 012_handler_config_additions.sql
INSERT INTO handler_config (worker_function, enabled, priority, updated_at)
VALUES
    ('face_clustering',     TRUE, 4, NOW()),
    ('voice_embedding',     TRUE, 4, NOW()),
    ('identity_resolution', TRUE, 3, NOW())
ON CONFLICT (worker_function) DO NOTHING;
```

### 4.6 Correct migration ordering

All migrations should be applied in sequence. Migration 009 depends on the `vector` extension enabled by 005. Migrations 010–012 depend on 009. The ClusterDbMigrationRunner applies all `.sql` files in `migrations/cluster/` in filename order (alphabetical, which equals numeric for this naming scheme), so ordering is automatic.

---

## 5. Data Flow Gaps

### 5.1 Cluster `thumbnails` table: produced, never consumed by the API

**Flow:** Python `thumbnails` handler calls `insert_thumbnail()` → rows written to cluster `thumbnails` table.

**Consumers:** None in the C# API. The API reads thumbnails only through `job_artifacts` (SQLite) + `ArtifactSyncService`, which materializes files from `cache_manifest` without ever reading the cluster `thumbnails` table.

**Impact:** Every thumbnail job writes a complete set of metadata rows that no consumer reads. The rows are not harmful but they are dead storage. Either the API should expose a `GET /api/thumbnails/{mediaId}` endpoint that reads from the cluster `thumbnails` table (useful for faster thumbnail retrieval without local file sync), or the cluster `thumbnails` table should be documented as a deprecated intermediate table.

### 5.2 `speaker_segments.voice_embedding`: schema exists, write path missing

**Flow:** Diarization handler (`insert_speaker_segment`) → voice_embedding always NULL → `get_speaker_segments_for_media` returns NULL embeddings → `find_similar_persons_by_face` cannot use voice for persona matching.

**Impact:** The entire voice arm of the persona pipeline (§4 of next-features.md, voice_embedding handler) has no write path into the DB. The schema and read path are ready; the missing piece is either a `voice_embedding` parameter added to `insert_speaker_segment`, or a new `update_speaker_segment_voice_embedding(segment_id, embedding)` function in cluster_db.py.

### 5.3 `persons` table: schema and write path exist, no read path in the API layer

Migration 009 and cluster_db.py both exist. Python code can create persons and update face_detection.person_id. However:

- No C# model for `persons` or `person_appearances` exists in `src/backend/NeoVLab.Api/Models/`.
- No C# repository exists for persons.
- No API endpoint exposes `GET /api/persons`, `PATCH /api/persons/{id}`, or `GET /api/persons/{id}/appearances`.

The persona pipeline can write person records via Python workers but the data is invisible to the frontend until the C# layer is extended.

### 5.4 `face_detections.bounding_box` is JSONB but no schema is enforced

`bounding_box JSONB NOT NULL` stores the bounding box. The Python handler writes `json.dumps(bounding_box)` where `bounding_box` is whatever InsightFace returns (a dict with x1/y1/x2/y2 or similar). No CHECK constraint or documented schema enforces the JSONB structure. Any consumer parsing bounding boxes must handle schema variation defensively.

### 5.5 `transcriptions.segments` JSONB — word-level timestamps produced but not queryable

The Whisper handler writes word/segment-level timestamps into `segments JSONB`. There is no GIN index on `segments`, no tsvector for segment-level text, and no API endpoint that exposes segment-level search. The data is produced and stored but only accessible by deserializing the full `segments` blob per row.

### 5.6 `job_artifacts.storage_domain` populated but `tier_config` never joined

`ArtifactSyncService` calls `StorageDomainResolver.ResolveStorageDomain(artifactType)` to set `storage_domain` on each artifact. The resulting string value (e.g., `"io-cache"` or `"local-storage"`) matches `tier_config.tier_name`. However no query in the codebase joins `job_artifacts` to `tier_config`. The `retention_days` and `auto_sync` fields in `tier_config` are never read. Tier-based retention enforcement would require this join.

### 5.7 `staged_inputs` (PostgreSQL) — produced by migration, consumed by nobody

As documented in §2.6: the cluster `staged_inputs` table was created in migration 004 to support durable ComfyUI input staging. The `InputStagingService` in C# writes to the SQLite `staged_inputs` table, not the PostgreSQL one. No C# service, Go code, or Python code reads or writes to the cluster `staged_inputs` table. It is orphaned schema.

---

## Summary Severity Table

| Issue | Severity | Type |
|---|---|---|
| `cache_manifest.UpsertAsync` drops `gcs_evicted_at` on conflict | **Critical** | Bug — can cause redundant GCS delete attempts |
| `"complete"` status alias queried in poll loop | **Warning** | Dead code + unnecessary DB round-trip |
| `synced` / `failed_acked` statuses have no DDL constraint | **Warning** | Schema hygiene — no protection against typos |
| `voice_embedding` column always NULL — write path missing | **Warning** | Data flow gap — entire voice persona arm non-functional |
| `face_detections.person_id` / `speaker_segments.person_id` have no FK or index | **Warning** | Referential integrity + future query performance |
| `person_appearances` has no UNIQUE constraint | **Warning** | Will produce duplicates when pipeline re-runs |
| `persons` has no `updated_at` trigger | **Warning** | Silent stale timestamps |
| No vector indexes (IVFFlat/HNSW) on any embedding column | **Warning** | Sequential scan; acceptable now, not at scale |
| `cleanup_expired_staged_inputs()` never called | **Warning** | Unbounded row accumulation |
| Cluster `staged_inputs` table never used | **Nitpick** | Dead schema |
| Cluster `thumbnails` table produced but not consumed | **Nitpick** | Data produced with no reader |
| `search_transcriptions` hardcodes `'english'` tsquery | **Nitpick** | Degraded search for non-English content |
| `job_artifacts.storage_domain` / `tier_config` never joined | **Nitpick** | Tier retention logic is unreachable |
| `handler_config` not seeded for future handlers | **Nitpick** | Will default to priority 5 but is undocumented |
| `persons` table has no C# model/repository/endpoints | **Nitpick** | Work in progress — expected gap |
