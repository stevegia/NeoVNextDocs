# Deployment Checklist: ComfyUI Timeout & Cache Optimization

**Target:** Fix timeout failures and improve cold start performance  
**ETA:** 15-20 minutes for deployment + verification  

---

## Pre-Deployment

- [x] ✅ Code changes complete
  - [x] `docker/deploy-cloudrun-gpu.sh` — timeout 2100s, env vars
  - [x] `docker/comfyui-entrypoint.sh` — parallel cache population
  - [x] Bash syntax validated (Unix line endings)
- [x] ✅ Documentation written
  - [x] [COMFYUI_TIMEOUT_FIX.md](COMFYUI_TIMEOUT_FIX.md) — full technical details
  - [x] This checklist
- [x] ✅ Regression check passed
  - No conflicts with recent SubmitComfy input staging changes
  - Keepalive logic intact
  - Pull worker unchanged

---

## Deployment Steps

### 1. Build & Push Docker Image

**IMPORTANT:** Set `REBUILD_IMAGE=1` to apply entrypoint changes.

```bash
cd /mnt/z/NeoVNext/docker  # Or your workspace path

export PROJECT="neovnext"
export REGION="us-central1"  # Or your preferred region
export REBUILD_IMAGE=1

./deploy-cloudrun-gpu.sh "${PROJECT}" "${REGION}"
```

**Expected output:**
```
--- Rebuilding GPU worker image (REBUILD_IMAGE=1) ---
Successfully built gcr.io/neovnext/neovlab-gpu-worker-comfyui:latest
--- Pushing to GCR ---
--- Deploying to Cloud Run GPU ---
Service URL: https://neovlab-gpu-worker-us-central1-...run.app
```

**Duration:** 8-12 minutes (includes Docker build with large layers)

---

### 2. Verify Deployment

```bash
# Get service URL
SERVICE_URL=$(gcloud run services describe neovlab-gpu-worker-${REGION} \
    --region "${REGION}" \
    --format 'value(status.url)')

echo "Service URL: ${SERVICE_URL}"

# Check environment variables
gcloud run services describe neovlab-gpu-worker-${REGION} \
    --region "${REGION}" \
    --format='value(spec.template.spec.containers[0].env)' | \
    grep -E 'COMFYUI_TIMEOUT|ENABLE_MODEL_CACHE'
```

**Expected output:**
```
COMFYUI_TIMEOUT_SECONDS=2000
COMFYUI_STARTUP_TIMEOUT_SECONDS=420
ENABLE_MODEL_CACHE=true
```

---

### 3. Test Cold Start (Optional but Recommended)

Trigger a cold start to verify parallel cache works:

```bash
# Force scale-to-zero (wait 10+ minutes for idle timeout)
# OR manually delete all instances:
gcloud run services update neovlab-gpu-worker-${REGION} \
    --region "${REGION}" \
    --max-instances 0

# Wait 30s for scale-down
sleep 30

# Restore max instances
gcloud run services update neovlab-gpu-worker-${REGION} \
    --region "${REGION}" \
    --max-instances 1

# Trigger a test job via API (replace with your backend URL)
curl -X POST "https://your-backend.run.app/api/jobs/comfy" \
    -H "Content-Type: application/json" \
    -d '{
        "mediaId": "test-media-id",
        "workflow": { "your": "workflow" },
        "options": { "priority": 5 }
    }'
```

**Watch logs:**
```bash
gcloud logging read "resource.type=cloud_run_revision \
    AND resource.labels.service_name=neovlab-gpu-worker-${REGION}" \
    --project="${PROJECT}" \
    --limit=50 \
    --format="value(timestamp,severity,textPayload)" \
    --freshness=5m
```

**Expected log sequence (within 2 minutes):**
```
[entrypoint] Model symlinks configured (hybrid cache + GCS, cache population will run in background)
[entrypoint] Starting ComfyUI on 0.0.0.0:8188 ...
[entrypoint-cache] Populating model cache in background (cold start) ...
[entrypoint-cache]   Caching vae/ (~254MB) ...
[entrypoint] ComfyUI ready after 38s.  ← Should be < 60s
[entrypoint] Starting NeoVLab pull-model GPU worker on port 8080 ...
Worker registered: gpu-comfyui-us-central1
[entrypoint-cache]   Caching text_encoders/ (~7GB) ...
[entrypoint-cache] Model cache populated in 187s ...  ← Completes in background
```

---

### 4. Monitor First Real Job

Submit a real ComfyUI job and watch for completion:

```bash
# Get job ID from submission response
JOB_ID="your-job-id"

# Poll job status
while true; do
    STATUS=$(gcloud sql connect neovlab-cluster --user=neovlab --database=cluster \
        --quiet <<< "SELECT status FROM job_queue WHERE job_id = '${JOB_ID}';" 2>/dev/null | tail -n2 | head -n1 | xargs)
    echo "Job ${JOB_ID}: ${STATUS}"
    [[ "$STATUS" =~ ^(completed|failed|synced)$ ]] && break
    sleep 10
done
```

**Success criteria:**
- ✅ Job status transitions: `queued` → `running` → `completed`
- ✅ No timeout errors in logs
- ✅ Completion time < 35 min (even for WAN models)
- ✅ Artifacts uploaded to GCS

---

## Rollback Plan

If deployment fails or jobs break:

```bash
# Revert to last known good revision
LAST_GOOD_REVISION="00053-l26"  # Update with your last stable revision

gcloud run services update-traffic neovlab-gpu-worker-${REGION} \
    --region "${REGION}" \
    --to-revisions="${LAST_GOOD_REVISION}=100"
```

**Then investigate:**
1. Check Cloud Run logs for entrypoint errors
2. Verify model cache directory structure: `docker run --rm -it gcr.io/neovnext/... ls -la /comfyui/models`
3. Test locally with docker-compose if possible

---

## Success Metrics

After 24 hours of production traffic:

| Metric | Target | Where to Check |
|--------|--------|----------------|
| Job completion rate | > 95% | PostgreSQL: `SELECT COUNT(*) FILTER (WHERE status='completed') * 100.0 / COUNT(*) FROM job_queue WHERE worker_function='comfyui'` |
| Average cold start time | < 2 min | Cloud Run logs: time from container start to "Worker registered" |
| Timeout failures | 0 | Cloud Run logs: grep for "Cloud Run request timeout" |
| Cache hit rate | > 80% | Infer from model load times in logs (instant = cached, 30s+ = GCS) |

---

## Known Issues & Next Steps

### ✅ Fixed in This Release
- Cloud Run timeout too short (20 min → 35 min)
- Model cache blocks ComfyUI startup (now parallel)
- Handler startup timeout too short (300s → 420s)

### ⚠️ Not Fixed (Future Work)
- WAN model load time (91GB, 15-20 min from GCS FUSE)
  - **Workaround:** New 35-min timeout is sufficient
  - **Future:** Cache WAN primary model (~20GB) or switch to signed URLs

### 📋 Monitoring TODO
- Set up alert for jobs in `running` status > 30 min
- Dashboard for cold start times (Cloud Monitoring)
- Model cache hit/miss telemetry

---

## Contact

Questions or issues? Check:
- [COMFYUI_TIMEOUT_FIX.md](COMFYUI_TIMEOUT_FIX.md) — Technical details
- [runbook-comfyui-cloud-run-debugging.md](runbook-comfyui-cloud-run-debugging.md) — Debug procedures
- Cloud Run logs: `gcloud logging read ...` (see step 3 above)
