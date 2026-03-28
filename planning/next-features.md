# Next Features: GPU Handler VRAM Budget and Persona/Cast Pipeline

**Last updated:** 2026-03-26

## 1. Handler VRAM Budget Table

Based on reading every handler in `src/workers/handlers/`:

| Handler | Model(s) | VRAM at Rest | VRAM Peak | GPU Required | Notes |
|---|---|---|---|---|---|
| **comfyui** (SD/SDXL) | Pony Realism v2.3 ULTRA (SDXL) | ~6.5 GB | ~8-10 GB | Yes | `--highvram` keeps model pinned |
| **comfyui** (Wan 14B) | Wan2.1 14B (video gen) | ~14 GB | ~20-22 GB | Yes, exclusive | Triggers `ModeExclusive` transition |
| **transcription** | OpenAI Whisper `turbo` | ~1.5 GB | ~2.5 GB | Yes | ~800M params |
| **diarization** | pyannote/speaker-diarization-3.1 | ~1.2 GB | ~2.0 GB | Yes | segmentation + embedding models |
| **face_extraction** | InsightFace buffalo_l (ONNX/CUDA) | ~0.5 GB | ~0.8 GB | Yes | Lightweight ONNX Runtime |
| **visual_tagging** | ViT-Base/16-224 (timm, ImageNet) | ~0.4 GB | ~0.6 GB | Yes | 86M param ViT |
| **thumbnails** | ffmpeg (CPU) | 0 GB | 0 GB | No | Subprocess-based |
| **transcript_analysis** | Stub (no model) | 0 GB | 0 GB | No | CPU-only placeholder |
| **compute_file_hashes** | None (SHA-256) | 0 GB | 0 GB | No | CPU-only |

**Total VRAM for all analysis handlers simultaneously:** ~3.6 GB at rest, ~5.9 GB peak

**Total for ComfyUI (SDXL) + all analysis handlers:** ~10.1 GB at rest, ~15.9 GB peak

## 2. Concurrency Matrix

### NVIDIA L4 (24 GB) -- Production

| Combination | VRAM Budget | Feasible? | Notes |
|---|---|---|---|
| ComfyUI (SDXL) + all 4 analysis handlers | ~10 GB / ~16 GB peak | **Yes** | Current `ModeShared`; 8+ GB headroom |
| ComfyUI (Wan 14B) alone | ~14 GB / ~22 GB peak | **Yes** | `ModeExclusive`; only 2 GB headroom |
| Wan 14B + any analysis handler | 22+ GB peak | **No** | Supervisor stops all handlers |
| All 4 analysis handlers (no ComfyUI) | ~3.6 GB / ~5.9 GB peak | **Yes** | 18 GB free |
| face_extraction alone | ~0.5 GB / ~0.8 GB peak | **Yes** | Very lightweight |

### GTX 1080 Ti (11 GB, CC 6.1) -- Local Dev

| Combination | VRAM Budget | Feasible? | Notes |
|---|---|---|---|
| face_extraction alone | ~0.8 GB peak | **Yes** | InsightFace ONNX works on CC 6.1 |
| visual_tagging alone | ~0.6 GB peak | **Yes** | ViT-B supports CC 6.1 |
| transcription (Whisper turbo) alone | ~2.5 GB peak | **Yes** | Whisper supports older CUDA |
| diarization alone | ~2.0 GB peak | **Yes** | pyannote works on older GPUs |
| All 4 analysis handlers together | ~5.9 GB peak | **Yes** | 5 GB headroom |
| ComfyUI (SDXL) alone | ~8-10 GB peak | **Marginal** | May OOM with large images |
| ComfyUI (SDXL) + any handler | 10+ GB peak | **No** | OOM risk |
| ComfyUI (Wan 14B) | ~22 GB peak | **No** | Impossible |

## 3. Face Extraction Deep Dive

### Current Status: Fully Implemented

The face_extraction handler at `src/workers/handlers/face_extraction.py` is **complete and production-ready**:

- **Model:** InsightFace `buffalo_l` via ONNX Runtime with CUDAExecutionProvider
- **Input:** Video file (any format OpenCV can read)
- **Processing:** Samples frames every `FACE_FRAME_INTERVAL` seconds (default 2.0s)
- **Output per detection:** 512-dim embedding, bounding box, confidence, estimated age/gender, frame number, timestamp

### Database Schema: Complete

Migration `005_face_detections.sql` applied. The `face_detections` table has:
- `embedding vector(512)` using pgvector
- `person_id TEXT` -- **already present but unused** (placeholder for persona linkage)
- Indexes on `(media_id, timestamp_seconds)` and `(media_id, frame_number)`
- No IVFFlat vector index yet (needed after enough data accumulated)

### Cluster DB Integration: Complete

- `insert_face_detection()` -- inserts individual detections with embeddings
- `get_face_detections_for_media()` -- retrieves all detections for a media item
- `find_similar_faces()` -- **pgvector cosine distance search** (default threshold 0.6)

### What is Missing for Persona/Cast

1. **No `persons` table** -- `person_id` column exists on `face_detections` and `speaker_segments` but no parent entity
2. **No face clustering pipeline** -- `find_similar_faces()` exists but nothing calls it automatically
3. **No voice embedding extraction** -- diarization extracts segments but not `voice_embedding` vectors
4. **No cross-modal linkage** -- no pipeline to correlate faces with speakers within the same media
5. **No IVFFlat index** -- vector search is sequential scan
6. **No face thumbnail extraction** -- handler outputs embeddings but doesn't crop/save face thumbnails

## 4. Pipeline Architecture: Face-Voice-Persona

```
Phase 1: Raw Extraction (existing handlers)
───────────────────────────────────────────
  Video File
     ├─► face_extraction → face_detections (512-dim embeddings, bboxes, timestamps)
     ├─► transcription   → transcriptions (full text, word-level segments)
     └─► diarization     → speaker_segments (speaker labels, audio clips, timestamps)

Phase 2: Enrichment (new work)
──────────────────────────────
  face_detections
     └─► face_clustering job (new)
           ├─► Cluster by embedding similarity (DBSCAN or agglomerative)
           ├─► Cross-reference with existing persons (pgvector ANN)
           ├─► Extract "best" face thumbnail per cluster
           └─► Write person_id back to face_detections

  speaker_segments
     └─► voice_embedding job (new)
           ├─► Compute 192-dim embeddings (speechbrain ECAPA-TDNN)
           ├─► Cross-reference with existing persons
           └─► Write voice_embedding + person_id to speaker_segments

Phase 3: Identity Resolution (new work)
────────────────────────────────────────
  Temporal alignment:
     ├─► For each face cluster, find overlapping speaker_segments
     ├─► Merge face person_id and voice person_id into unified identity
     └─► Create or update persons table

Phase 4: UI / Cast Sheet
─────────────────────────
  persons table
     ├─► Name (user-assigned or auto-suggested)
     ├─► Representative face thumbnail + voice sample
     ├─► Appearance list: which media, at which timestamps
     └─► Frontend: "Cast" panel across library
```

### New Database Tables Required

```sql
CREATE TABLE persons (
    person_id         TEXT PRIMARY KEY DEFAULT gen_random_uuid()::TEXT,
    name              TEXT,
    face_thumbnail    TEXT,
    voice_sample_path TEXT,
    face_embedding    vector(512),   -- Centroid of face cluster
    voice_embedding   vector(192),   -- Centroid of voice cluster
    auto_generated    BOOLEAN DEFAULT TRUE,
    created_at        TIMESTAMPTZ DEFAULT NOW(),
    updated_at        TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE person_appearances (
    appearance_id    TEXT PRIMARY KEY DEFAULT gen_random_uuid()::TEXT,
    person_id        TEXT NOT NULL REFERENCES persons(person_id) ON DELETE CASCADE,
    media_id         TEXT NOT NULL,
    first_seen_sec   REAL NOT NULL,
    last_seen_sec    REAL NOT NULL,
    detection_count  INTEGER NOT NULL DEFAULT 1,
    source           TEXT NOT NULL,  -- 'face' | 'voice' | 'both'
    created_at       TIMESTAMPTZ DEFAULT NOW()
);
```

### New Handlers Required

| Handler | Type | Model | VRAM | Purpose |
|---|---|---|---|---|
| `face_clustering` | CPU | None (scipy/sklearn) | 0 GB | Cluster embeddings, assign person_id |
| `voice_embedding` | GPU | speechbrain ECAPA-TDNN | ~0.3 GB | Compute voice embeddings |
| `identity_resolution` | CPU | None | 0 GB | Temporal face-voice alignment |

## 5. Scaling Strategy

### Single L4 Optimization (Current)

- Shared mode: ComfyUI + all analysis (~10 GB, ~14 GB free)
- Add `face_clustering` as CPU handler (port 9007) -- zero VRAM
- Add `voice_embedding` as lightweight GPU handler (port 9008) -- ~0.3 GB
- Batch face extraction for multiple media_ids
- Create IVFFlat indexes after >1000 vectors accumulated

### Multi-GPU (When Quota Increases)

- Architecture already supports multiple workers via `FOR UPDATE SKIP LOCKED`
- Worker specialization: one ComfyUI-only, one analysis-only
- No code changes needed for basic multi-worker
- `handler_config` table enables runtime toggles

### Local Development on GTX 1080 Ti (11 GB)

- All 4 analysis handlers fit (~5.9 GB peak, 5 GB headroom)
- Set `ENABLED_HANDLERS=face_extraction,visual_tagging,transcription,diarization`
- No ComfyUI locally (OOM risk)
- CUDA 11.x compatible: all models support CC 6.1
- Requires HuggingFace token for pyannote

## 6. Recommended Next Steps (Prioritized)

### Priority 1: Face extraction pipeline completion (2-3 days)

1. Add face thumbnail cropping to face_extraction handler
2. Create migration `009_persons.sql` with `persons` and `person_appearances` tables
3. Implement `face_clustering` CPU handler (sklearn AgglomerativeClustering)
4. Wire API to trigger face_extraction after thumbnails complete

### Priority 2: Voice embedding extraction (1-2 days)

5. Add speechbrain ECAPA-TDNN to Docker image
6. Create `voice_embedding` handler
7. Add `find_similar_voices()` to cluster_db.py

### Priority 3: Identity resolution (2-3 days)

8. Implement `identity_resolution` CPU handler (temporal face-voice alignment)
9. Create pgvector IVFFlat indexes
10. Add API endpoints: `GET /api/persons`, `PATCH /api/persons/{id}`

### Priority 4: Frontend cast panel (2-3 days)

11. Build "Cast" page with face thumbnails, names, appearances
12. Person rename, merge, split UI
13. Person overlay on video player timeline

### Priority 5: Local development setup (1 day)

14. Create `docker-compose.local-gpu.yml` for 1080 Ti
15. Document CUDA 11.x constraints
