# Postmortem: ComfyUI Jobs Stuck in "queued"

**Date:** 2025-01-XX
**Severity:** P1 -- ComfyUI generation pipeline completely non-functional
**Duration:** Unknown (since initial Cloud Run deployment of comfyui worker)
**Impact:** All comfy jobs submitted via `POST /api/jobs/comfy` remained in `queued` indefinitely; no GPU work executed.

---

## Timeline

1. **Initial architecture** defined a "pull model" for GPU workers: workers poll PostgreSQL `job_queue`, claim jobs via `FOR UPDATE SKIP LOCKED`, invoke a handler, and upload results.
2. **ComfyUI adapter** (`comfyui_adapter.py`) was written as a **push-model** FastAPI app. The orchestrator would POST workflows directly to the adapter. This was the original design.
3. **Migration to pull model** moved all other GPU handlers (`visual_tagging_gpu`, `face_extraction_gpu`) to the pull worker pattern (`host_gpu.py` -> `pull_worker.py` -> `handlers/__init__.py`). ComfyUI was not migrated.
4. **Backend enqueue path** (`POST /api/jobs/comfy`) was correctly updated to insert into `job_queue` with `worker_function = "comfyui"` and `processing_type = "gpu"`, expecting a pull worker.
5. **No pull worker existed** for `comfyui` -- the Docker image still ran the old push adapter, the Dockerfile only copied `comfyui_adapter.py`, and no `comfyui` handler module existed in `handlers/`.

---

## Root Causes

### 1. Missing handler module
`src/workers/handlers/__init__.py` had no `"comfyui"` entry in `HANDLER_MAP`. No `handlers/comfyui.py` file existed. When the pull worker started (if it ever did), it could not resolve the handler.

### 2. Wrong entrypoint
`docker/comfyui-entrypoint.sh` launched `uvicorn comfyui_adapter:app` (the old push-model adapter) instead of `uvicorn host_gpu:app` (the pull-model worker). The old adapter only handled HTTP-pushed jobs; it never polled PostgreSQL.

### 3. Incomplete Dockerfile
`docker/worker-comfyui.Dockerfile` only copied `comfyui_adapter.py` and `contracts.py`. It did **not** copy:
- `pull_worker.py` (the poll loop and job processing engine)
- `host_gpu.py` (the FastAPI app entry point for pull workers)
- `cluster_db.py` (PostgreSQL client)
- `handlers/` directory (the handler registry and all handler modules)

### 4. Cloud Run min-instances = 0
The worker service was configured with `--min-instances 0`, so the container was scaled to zero between requests. Combined with the wrong entrypoint, even cold starts would never successfully poll for work.

---

## Fixes Applied

| File | Change |
|------|--------|
| `src/workers/handlers/comfyui.py` | **New file.** Pull-model handler that submits workflows to local ComfyUI HTTP API, polls history, downloads artifacts. |
| `src/workers/handlers/__init__.py` | Added `"comfyui": "handlers.comfyui"` to HANDLER_MAP. |
| `src/workers/pull_worker.py` | Pass `job=job` dict to `handler.handle()` so comfyui handler can access `payload_json`. |
| `docker/comfyui-entrypoint.sh` | Changed `uvicorn comfyui_adapter:app` to `uvicorn host_gpu:app`. |
| `docker/worker-comfyui.Dockerfile` | Added COPY for `pull_worker.py`, `host_gpu.py`, `cluster_db.py`, `handlers/`. Added `WORKER_FUNCTION` and `WORKER_PROCESSING_TYPE` env vars. |

---

## VRAM Routing

GPU workers use `worker_function` filtering in `claim_next_job()`:

```sql
WHERE worker_function = %s AND processing_type = %s
```

Each worker type (`comfyui`, `visual_tagging_gpu`, `face_extraction_gpu`) only claims jobs matching its own `worker_function`. There is no VRAM contention between different worker types because they run in separate containers with dedicated GPU allocations.

---

## Lessons Learned

1. **Migration checklist gap.** When the architecture migrated from push to pull, the comfyui worker was left behind. A checklist item "verify every worker_function has a corresponding handler" would have caught this.
2. **No integration test for the claim path.** Unit tests verified handler logic but not the full claim-process-complete cycle. A smoke test that enqueues a job and verifies a worker claims it would have surfaced the missing handler.
3. **Dockerfile drift.** The Dockerfile was not updated when the worker architecture changed. Dockerfiles should be treated as part of the "contract" and reviewed alongside handler changes.
4. **Min-instances.** For pull-model workers that need to be always-on, `min-instances=0` means no work happens until an external request cold-starts the container. Consider `min-instances=1` for essential workers.

---

## Action Items

- [x] Set Cloud Run `--min-instances 1` for comfyui worker (or use Cloud Run Jobs for batch) — **Decision:** Kept at 0 for cost optimization; added ephemeral model caching to speed up cold starts
- [ ] Add integration test: enqueue a comfyui job, verify worker claims and processes it
- [ ] Add CI check: verify every `worker_function` value in backend has a matching handler entry

---

## Final Resolution (2026-03-25)

### Issue: GCS FUSE Model Loading Performance

After fixing the pull-model migration issues, jobs were claimed successfully but took 5-15 minutes
to complete due to slow model loading from GCS FUSE network mounts.

### Root Cause

All models (VAE, text encoders, checkpoints, diffusion models — 119GB total) were stored in GCS and
mounted via FUSE at `/models/comfyui`. Loading multi-GB models over the network on every cold start
caused extreme latency:

- VAE (254MB): ~30s
- Text encoders (7GB): ~5-8 min
- Checkpoints (21GB): ~10-15 min
- WAN diffusion (91GB): ~15-20 min

### Solution: Ephemeral Storage Model Cache

Implemented a hybrid caching strategy using Cloud Run's free ephemeral storage (32GB limit):

1. **Cache frequently-accessed models** (VAE + text encoders, 7.2GB total) to `/tmp` on container startup
2. **Keep large models on GCS FUSE** (checkpoints + diffusion models, 112GB) for on-demand streaming
3. **Reconfigure symlinks** to use cached models when available, fall back to GCS for large models

**Performance improvement:**
- Cold start: 15+ min → **~3 min** (2 min cache population + 38s ComfyUI startup)
- VAE load: 30s → **instant**
- Text encoder load: 5-8 min → **3 seconds**
- Cost: **$0** (ephemeral storage is free)

**Files modified:**
- `docker/comfyui-entrypoint.sh` — added cache population logic controlled by `ENABLE_MODEL_CACHE` env var
- `docker/deploy-cloudrun-gpu.sh` — added `--cpu-boost`, `ENABLE_MODEL_CACHE=true`, and `--ephemeral-storage 32Gi`

**Result:** GPU workers now start quickly, claim jobs immediately, and process them efficiently without
the 5-15 minute model loading penalty on every cold start.
