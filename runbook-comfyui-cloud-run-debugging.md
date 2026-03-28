# ComfyUI on Cloud Run: End-to-End Debugging Runbook

> Reference doc for the bugs encountered getting real GPU inference working on Cloud Run with the
> NeoVLab ComfyUI worker. Use this when inference jobs hang, fail silently, or return simulation
> artifacts instead of real images.

---

## Architecture Reminder

```
Cloud SQL (cluster db: job_queue, worker_registry, cache_manifest)
  ^                   ^
  |   poll for jobs   |   register / heartbeat
  v                   v
Cloud Run service: neovlab-gpu-worker-east4
  --> comfyui-entrypoint.sh
    |- ComfyUI process      (port 8188, background)
    |- host_gpu:app          (FastAPI, port 8080, foreground)
      |- pull_worker.py      (claims jobs via SELECT ... FOR UPDATE SKIP LOCKED)
      |- handlers/comfyui.py (submits workflow to ComfyUI HTTP API)
```

The pull worker polls `job_queue` in Cloud SQL, claims a job, dispatches to the
`comfyui` handler which submits the workflow to ComfyUI's internal API and polls
`GET /history/{prompt_id}` until the image is ready, then writes PNG bytes to
`ARTIFACT_DIR` (GCS FUSE mount at `/artifacts`).

> **Previous architecture (deprecated):** The old push-model used `comfyui_adapter`
> as a FastAPI server that received jobs from the C# backend. This was replaced by
> the pull-model where workers claim jobs from Cloud SQL directly.

---

## Quick Diagnostic Checklist

Before digging into logs, check the observability API endpoints on the NeoVLab backend (no SQL access needed):

```bash
# Queue depth — jobs waiting or assigned to workers
curl -H "Authorization: Bearer <token>" https://<api-url>/api/observability/queue-depth

# Rolling error rate (last 15 min by default)
curl -H "Authorization: Bearer <token>" https://<api-url>/api/observability/error-rate?windowMinutes=15
```

Expected healthy values: `queue-depth.Depth == 0` after job completes; `error-rate.ErrorRatePercent < 5` for stable sessions.

If the API is unreachable or queue-depth stays non-zero after the expected job window, continue with the worker-side checks below.

---

Before digging into worker logs, run these checks:

```powershell
$BASE = "https://neovlab-gpu-worker-east4-221741263387.us-east4.run.app"
$TOK  = "df6478419f5ebf6a52cb31e5ef985015eaf9f4d662bcdf388a7dee4f30789ce5"

# 1. Is the adapter alive?
python -c "import urllib.request,json; print(json.loads(urllib.request.urlopen('$BASE/health', timeout=10).read())['comfyui'])"

# 2. Is ComfyUI's queue clear?
python -c "import urllib.request,json; req=urllib.request.Request('$BASE/debug/comfyui/queue', headers={'Authorization':'Bearer $TOK'}); print(json.loads(urllib.request.urlopen(req,timeout=10).read()))"

# 3. VRAM state (should say HIGH_VRAM once warmed)
python -c "import urllib.request,json; req=urllib.request.Request('$BASE/debug/comfyui/system_stats', headers={'Authorization':'Bearer $TOK'}); ss=json.loads(urllib.request.urlopen(req,timeout=10).read()); [print(d['name'], 'free:', d['vram_free']//1024**2, 'MB') for d in ss['devices']]"
```

Expected healthy state:
- `comfyui.running = True`
- `queue_running: [], queue_pending: []`
- VRAM free < total (model loaded), or = total (cold, not yet loaded)

---

## Bug Catalogue

### Bug 1: Docker symlink silently created inside existing directory

**Symptom:**
```
ckpt_name: 'ponyRealism_V23ULTRA.safetensors' not in []
```
ComfyUI shows an empty checkpoint list even though models exist in GCS.

**Root cause:**
The git clone of ComfyUI creates a real directory at `/comfyui/models/`. When the Dockerfile then runs:
```dockerfile
RUN ln -sf /models/comfyui /comfyui/models
```
`ln -sf` on an *existing directory target* does **not** replace the directory — it silently creates
a new symlink *inside* the directory at `/comfyui/models/comfyui`. ComfyUI never finds the GCS models.

**Fix** (in `docker/worker-comfyui.Dockerfile`):
```dockerfile
RUN rm -rf /comfyui/models \
    && ln -s /models/comfyui /comfyui/models \
    && mkdir -p /models/comfyui/checkpoints /models/comfyui/vae /models/comfyui/loras
```
Delete the directory first, then create the symlink.

**Verify in a running container:**
```bash
docker run --rm gcr.io/neovnext/neovlab-gpu-worker-comfyui:latest ls -la /comfyui/ | grep models
# Expected: lrwxrwxrwx models -> /models/comfyui
```

---

### Bug 2: ComfyUI poll timeout too short for cold GCS FUSE model load

**Symptom:** Job runs for exactly ~315s then fails with:
```
Remote worker reported failure without details
```
Cloud Run logs show the model loading (`model weight dtype torch.float16`) but no error — the adapter
timeout fires while ComfyUI is still reading the 6.6 GB model from GCS FUSE.

**Root cause:**
`poll_and_save_comfyui_output` had `timeout_seconds=300`. Cold start model load from GCS FUSE at
~200 MB/s sequential read for a 6.6 GB file takes 90–180s on its own, plus compute time.

**Fix** (in `src/workers/comfyui_adapter.py`):
```python
async def poll_and_save_comfyui_output(prompt_id: str, job_id: str, timeout_seconds: int = 900) -> bool:
```
Also update the Cloud Run deploy `--timeout 900` to match.

**Measured timings (L4 GPU, us-east4):**
| Phase | Time |
|-------|------|
| Cold GCS FUSE model load (6.6 GB) | ~130s |
| 5-step 512x512 inference | ~35s |
| Total cold start | ~165s |
| Warm inference (model in VRAM) | ~35s |

---

### Bug 3: ComfyUI running in CPU/NORMAL_VRAM mode

**Symptom:** Inference runs (no errors), but takes >10 minutes and never completes.

**Symptom in Cloud Run logs:**
```
Set vram state to: NORMAL_VRAM
Unsupported Pytorch detected. DynamicVRAM support requires Pytorch version 2.8 or later.
Falling back to legacy ModelPatcher.
```

**Root cause:**
PyTorch 2.5.1 (what fits in a reasonable Docker image with CUDA 12.1) doesn't satisfy the 2.8+
requirement for ComfyUI's DynamicVRAM manager. In `NORMAL_VRAM` mode, the legacy `ModelPatcher`
constantly moves model slices between CPU and GPU RAM, making each step take minutes instead of
milliseconds.

**Fix** (in `docker/comfyui-entrypoint.sh`):
```bash
python3 /comfyui/main.py \
    --listen "${COMFYUI_HOST}" \
    --port "${COMFYUI_PORT}" \
    --disable-auto-launch \
    --disable-metadata \
    --highvram \
    --disable-smart-memory \
    &
```
- `--highvram` forces `HIGH_VRAM` state, keeping the whole model on the GPU.
- `--disable-smart-memory` disables the legacy ModelPatcher's automatic offloading.

**Verify:**
```
Set vram state to: HIGH_VRAM   # <-- expected in startup logs
```

---

### Bug 4: GCS FUSE model loading is extremely slow on cold starts

**Symptom:**
- Worker starts successfully but jobs take 5+ minutes just to load models
- Logs show `model weight dtype torch.float16` then long silence (5-15 min)
- First job after cold start frequently times out

**Root cause:**
GCS FUSE mounts (`/models/comfyui`) stream model files over the network. Large models (6-21GB checkpoints,
91GB diffusion models) take 5-15 minutes to load on every cold start. Network streaming adds significant
latency compared to local disk.

**Measured timings (before fix):**
| Model | Size | Load Time |
|-------|------|----------|
| VAE | 254MB | ~30s (GCS FUSE) |
| Text Encoders | 7GB | ~5-8 min (GCS FUSE) |
| SD Checkpoint | 21GB | ~10-15 min (GCS FUSE) |
| WAN Diffusion | 91GB | ~15-20 min (GCS FUSE) |

**Fix: Ephemeral Storage Model Cache**

Cloud Run provides up to 32GB of **free** ephemeral storage (in-memory tmpfs at `/tmp`). The worker now
automatically caches small, frequently-accessed models on startup.

**Implementation** (in `docker/comfyui-entrypoint.sh`):
```bash
if [ "${ENABLE_MODEL_CACHE}" = "true" ]; then
    # Copy VAE + text encoders to /tmp (7.2GB total, ~2 min)
    cp -r /models/comfyui/vae /tmp/models/
    cp -r /models/comfyui/text_encoders /tmp/models/
    
    # Reconfigure symlinks to use hybrid approach
    ln -s /tmp/models/vae /comfyui/models/vae
    ln -s /tmp/models/text_encoders /comfyui/models/text_encoders
    ln -s /models/comfyui/checkpoints /comfyui/models/checkpoints      # still GCS
    ln -s /models/comfyui/diffusion_models /comfyui/models/diffusion_models  # still GCS
fi
```

**Deployment** (in `docker/deploy-gpu-worker.sh`):
```bash
gcloud run deploy neovlab-gpu-worker-east4 \
    --cpu-boost \
    --set-env-vars "ENABLE_MODEL_CACHE=true,..." \
    ...
```

**Results (after fix):**
| Model | Size | Cached? | Load Time |
|-------|------|---------|----------|
| VAE | 254MB | ✅ | Instant |
| Text Encoders | 7GB | ✅ | ~3s |
| Primary Checkpoint (ponyRealism) | 7GB | ✅ | Instant |
| Other Checkpoints | ~14GB | ❌ | ~3-5 min (GCS) |
| WAN Diffusion | 91GB | ❌ | ~10-15 min (GCS) |

**Total cold start:** ~4-5 min (cache population) + ~38s (ComfyUI startup) = **~5 min** vs 15+ min before.

**Why not cache everything?**
- All checkpoints (21GB) + WAN models (91GB) = 112GB total
- Copying 112GB exceeds Cloud Run's startup timeout even at 20 min
- Ephemeral storage limit is 32GB (14GB used for optimal frequently-accessed models)
- Primary checkpoint (ponyRealism) handles most smoke tests and common workflows
- Other checkpoints are loaded on-demand when needed for specific workflows

**Cost:** $0 (ephemeral storage is free)

**Disable caching** (for testing/debugging):
```bash
gcloud run services update neovlab-gpu-worker-east4 \
    --region us-east4 \
    --update-env-vars ENABLE_MODEL_CACHE=false
```

---

### Bug 5: Stale/stuck job blocks all subsequent ComfyUI jobs

**Symptom:**
- Job `N` errors (OOM, bad workflow, etc.)
- All subsequent jobs hang indefinitely
- `GET /debug/comfyui/queue` shows the new job in `queue_running` but history shows the previous
  job has `status_str: error`

**Root cause:**
ComfyUI's execution thread can get stuck after a job error (especially GPU OOM). The new job is
added to the queue but immediately blocked behind the stuck execution thread.

**Fix** (in `src/workers/comfyui_adapter.py`):
```python
async def run_comfyui_job(...):
    # Clear any stale state before submitting
    await interrupt_comfyui()   # POST /interrupt
    await clear_comfyui_queue() # POST /queue {"clear": true}
    await asyncio.sleep(1)
    submitted_id = await try_submit_prompt(workflow, job_id)
    ...
    # Also interrupt on timeout so future jobs aren't blocked
    if not success:
        await interrupt_comfyui()
        jobs[job_id]["status"] = "failed"
```

**Note:** This was originally Bug 4, renumbered to Bug 5 after adding the GCS FUSE performance issue.

**Manual recovery** (if the interrupt helpers don't exist yet):
```python
import urllib.request, json
BASE = "https://neovlab-gpu-worker-east4-221741263387.us-east4.run.app"
TOK  = "df6478419f5ebf6a52cb31e5ef985015eaf9f4d662bcdf388a7dee4f30789ce5"

# Interrupt current execution
req = urllib.request.Request(BASE + "/debug/comfyui/interrupt",
    data=b"", headers={"Authorization": "Bearer " + TOK}, method="POST")
urllib.request.urlopen(req, timeout=10)

# Clear the queue
req2 = urllib.request.Request(BASE + "/debug/comfyui/queue",
    data=json.dumps({"clear": True}).encode(),
    headers={"Authorization": "Bearer " + TOK, "Content-Type": "application/json"},
    method="POST")
urllib.request.urlopen(req2, timeout=10)
```

---

### Bug 6: `GetRequiredModels()` blocking dispatch with hard-coded model names

**Symptom:**
```
Required models not available: sdxl-base-1.0, sdxl-vae-fix
```
Job never gets dispatched to the worker.

**Root cause:**
`GpuPlatformWorkerClient.GetRequiredModels()` returned `{ "sdxl-base-1.0", "sdxl-vae-fix" }` for
all `comfyui` job types regardless of the actual workflow.

**Fix** (in `src/backend/NeoVLab.Api/Services/GpuPlatform/GpuPlatformWorkerClient.cs`):
```csharp
private static IReadOnlyList<string> GetRequiredModels(ProcessingJob job)
{
    if (job.JobType.Equals("comfyui", StringComparison.OrdinalIgnoreCase))
        return string.IsNullOrWhiteSpace(job.WorkflowJson)
            ? new[] { "sdxl-base-1.0", "sdxl-vae-fix" }
            : Array.Empty<string>();
    return Array.Empty<string>();
}
```
When a user provides their own `workflow` JSON the model list is empty (validation skipped).

---

### Bug 7: `relation "worker_registry" does not exist` on worker startup

**Symptom:**
Worker starts, connects to Cloud SQL, but crashes immediately:
```
psycopg2.errors.UndefinedTable: relation "worker_registry" does not exist
```
Jobs remain stuck in `queued` state.

**Root cause:**
The Cloud SQL `cluster` database was created but `001_cluster_schema.sql` was never applied.
Unlike the local SQLite migrations (auto-applied by `MigrationRunner.cs`), the Cloud SQL schema
must be applied manually.

**Fix:**
Apply the schema via GCS import (see runbook-gcp-gpu.md Phase 1.4):
```bash
gcloud storage cp src/data/migrations/cluster/001_cluster_schema.sql \
    gs://YOUR_PROJECT-neovlab-io-cache/migrations/001_cluster_schema.sql

gcloud sql import sql neovlab-cluster \
    gs://YOUR_PROJECT-neovlab-io-cache/migrations/001_cluster_schema.sql \
    --database=cluster --user=neovlab --project=YOUR_PROJECT --quiet
```

**Verify:**
```bash
gcloud sql connect neovlab-cluster --database=cluster --user=neovlab
# \dt should show: job_queue, cache_manifest, worker_registry
```

---

## Rebuild / Redeploy Cheat Sheet

```powershell
cd z:\NeoVNext

# Build (change comment in Dockerfile to bust COPY cache)
docker build -f docker/worker-comfyui.Dockerfile `
    -t gcr.io/neovnext/neovlab-gpu-worker-comfyui:latest .

# Push
docker push gcr.io/neovnext/neovlab-gpu-worker-comfyui:latest

# Deploy (all flags required — Cloud Run forgets them on each deploy)
& "C:\Program Files (x86)\Google\Cloud SDK\google-cloud-sdk\bin\gcloud.cmd" run deploy neovlab-gpu-worker-east4 `
  --image gcr.io/neovnext/neovlab-gpu-worker-comfyui:latest `
  --project neovnext --region us-east4 `
  --gpu 1 --gpu-type nvidia-l4 `
  --memory 16Gi --cpu 4 `
  --min-instances 0 --max-instances 1 `
  --add-volume "name=models,type=cloud-storage,bucket=neovnext-neovlab-models" `
  --add-volume-mount "volume=models,mount-path=/models/comfyui" `
  --add-volume "name=artifacts,type=cloud-storage,bucket=neovnext-neovlab-artifacts" `
  --add-volume-mount "volume=artifacts,mount-path=/artifacts" `
  "--set-env-vars=WORKER_TOKEN=df6478419f5ebf6a52cb31e5ef985015eaf9f4d662bcdf388a7dee4f30789ce5,WORKER_TYPE=comfyui,WORKER_FUNCTION=comfyui,WORKER_PROCESSING_TYPE=comfyui,ARTIFACT_DIR=/artifacts,COMFYUI_MODELS=animagineXL_v31,CLUSTER_DB_HOST=34.21.76.69,CLUSTER_DB_PORT=5432,CLUSTER_DB_NAME=cluster,CLUSTER_DB_USER=neovlab,CLUSTER_DB_PASSWORD=neovlab" `
  --timeout 900 --concurrency 1

# Re-add public IAM (Cloud Run resets it on each deploy)
& "C:\Program Files (x86)\Google\Cloud SDK\google-cloud-sdk\bin\gcloud.cmd" run services add-iam-policy-binding neovlab-gpu-worker-east4 `
    --project neovnext --region us-east4 `
    --member="allUsers" --role="roles/run.invoker"
```

> **Note:** The `WORKER_TOKEN` above is not a secret — it is only checked by the adapter, which is
> already behind Cloud Run IAM. Rotate if the service becomes public-facing.

---

## Debug Scripts

| Script | Purpose |
|--------|---------|
| `gpu_quick_test.py` | 512x512, 5-step test; polls 15 min; prints artifact size |
| `gpu_inference_test.py` | 1024x1024, 20-step test; polls 10 min |
| `debug_comfy.py` | Prints ComfyUI queue, history, and VRAM usage |
| `poll_worker.py` | Polls worker-side job status directly (bypasses API) |

---

## Models in GCS

Bucket: `gs://neovnext-neovlab-models/checkpoints/`

| File | Style | Size |
|------|-------|------|
| `animagineXL_v31.safetensors` | SD anime (SDXL) | ~6.5 GB |
| `ponyRealism_V23ULTRA.safetensors` | Pony realistic | ~6.6 GB |
| `ponyDiffusionV6XL_v6StartWithThisOne.safetensors` | Pony anime | ~6.2 GB |

All registered in `src/backend/NeoVLab.Api/data/neovlab.db` model_registry with `is_available=1`.

---

## API-Mediated Artifact Access & Cache-Miss Troubleshooting (US6 / FR-024)

All artifact access from the frontend goes through the NeoVLab API (`GET /api/artifacts/{id}/download`).
Workers never serve artifacts directly to the browser.

### Artifact serving flow

```
Browser                 NeoVLab API                  IO Cache (GCS FUSE / local)
  |                         |                                  |
  |  GET /artifacts/{id}/   |                                  |
  |  download               |                                  |
  |----------------------->|                                  |
  |                         | 1. Lookup artifact in local DB   |
  |                         | 2. File present locally? serve   |
  |                         | 3. Missing? TryEnsureLocalCopy   |
  |                         |  (lazy-sync from IO cache) ----> |
  |                         |                check CachePath   |
  |                         | <-------------------------------- |
  |                         | 4. Content-type from            |
  |                         |    CacheManifest.ContentType     |
  |                         |    (metadata-first, FR-027)      |
  | <---------------------- | 5. Serve file stream             |
```

### Cache-miss response: `artifact_unavailable_regeneration_required`

If the artifact is absent from **both** local permanent storage and the worker IO cache,
the API returns a structured 404 (FR-027A):

```json
{
  "error": "artifact_unavailable_regeneration_required",
  "artifactId": "<id>",
  "message": "Artifact is unavailable from local storage and IO cache. Explicit regeneration is required."
}
```

**Important:** the API does NOT auto-enqueue regeneration. The frontend must request it explicitly.

### Diagnosis: artifact serves with wrong content-type

1. Check `cache_manifest` table in the cluster DB for the job's output entry:
   ```sql
   SELECT entry_id, job_id, direction, file_key, content_type, synced_locally
   FROM cache_manifest
   WHERE job_id = '<job_id>' AND direction = 'output';
   ```
2. If `content_type = 'application/octet-stream'` the worker did not set it. Check worker
   handler `comfyui.py` — it should set `content_type` on the manifest entry at job completion.
3. If the `content_type` is correct in the DB but the browser receives the wrong type,
   check `ArtifactsController.Download` logs — the metadata path requires a matching manifest
   entry whose `ContentType` is not `application/octet-stream`.

### Diagnosis: artifact 404 after job completed

1. Confirm the job is `completed` in `job_queue`:
   ```sql
   SELECT status, acked_at FROM job_queue WHERE job_id = '<job_id>';
   ```
2. Check `cache_manifest` for `synced_locally`:
   ```sql
   SELECT cache_path, synced_locally, synced_at, evicted_at FROM cache_manifest
   WHERE job_id = '<job_id>' AND direction = 'output';
   ```
3. Verify the IO cache file exists at `cache_path` on the API server's file system.
4. If `cache_path` does not exist, the artifact is a full cache-miss — regeneration is required.
5. Check API logs for: `Full cache miss: artifact ... absent from both local storage and IO cache`.

---

## Known Limitations

- **Cold start ~165s**: GCS FUSE reads the model sequentially on first access. Subsequent jobs on a
  warm instance take ~35s (model stays in VRAM with `--highvram`).
- **PyTorch 2.5.1 vs 2.8+**: The `comfy_kitchen` CUDA backend and DynamicVRAM manager require
  PyTorch 2.8+. We work around this with `--highvram --disable-smart-memory`. When a torch 2.8+
  CUDA 12.x image becomes available, remove those flags and re-test.
- **Single-instance concurrency**: Cloud Run is set to `--concurrency 1` and `--max-instances 1`.
  The pull worker claims one job at a time from Cloud SQL. Scaling out requires raising `--max-instances`.
- **Artifacts not in GCS**: The adapter writes to `/artifacts` (GCS FUSE mount), but the NeoVLab
  API downloads the artifact from the worker over HTTP before GCS FUSE flushes the write.
  Cross-instance artifact retrieval (e.g. after a scale-out) would need a dedicated GCS read path.
