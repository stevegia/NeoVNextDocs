# Cloud Run GPU Worker Investigation Runbook

> Step-by-step procedures for diagnosing Cloud Run GPU worker issues.
> Use when: health checks fail, handlers crash, cold starts timeout, or jobs get stuck.

---

## Quick Triage (30 seconds)

```bash
# 1. Is the service healthy?
gcloud run services describe neovlab-gpu-worker-east4 \
    --region us-east4 \
    --format="yaml(status.conditions,status.latestReadyRevisionName,status.traffic)"

# 2. Which revision is serving traffic vs latest created?
gcloud run revisions list --service neovlab-gpu-worker-east4 \
    --region us-east4 \
    --format="table(name,active,creation_timestamp,status.conditions[0].status)" \
    --limit=5
```

**What to look for:**
- `latestCreatedRevisionName` != `latestReadyRevisionName` means the newest revision failed to become healthy
- `active: yes` column shows which revision is actually serving traffic
- A revision with `status: True` but not `active` was superseded

---

## Check Recent Logs

### Warning/Error logs (last 2 days)

```bash
gcloud logging read \
    'resource.type="cloud_run_revision" AND resource.labels.service_name="neovlab-gpu-worker-east4" AND severity>=WARNING' \
    --limit=30 \
    --format="table(timestamp,severity,textPayload)" \
    --freshness=2d
```

### Structured JSON logs (for detailed tracebacks)

```bash
gcloud logging read \
    'resource.type="cloud_run_revision" AND resource.labels.service_name="neovlab-gpu-worker-east4" AND severity>=ERROR' \
    --limit=10 \
    --format=json \
    --freshness=2d
```

### Startup, health, and GPU-related logs

```bash
gcloud logging read \
    'resource.type="cloud_run_revision" AND resource.labels.service_name="neovlab-gpu-worker-east4" AND textPayload:("startup" OR "listening" OR "ready" OR "health" OR "GPU" OR "CUDA" OR "handler")' \
    --limit=30 \
    --format="table(timestamp,severity,textPayload)" \
    --freshness=2d
```

### Tail logs in real-time

```bash
gcloud logging tail \
    "resource.type=cloud_run_revision AND resource.labels.service_name=neovlab-gpu-worker-east4" \
    --format="value(textPayload)"
```

---

## Check Container Images (Artifact Registry / GCR)

```bash
# List repositories
gcloud artifacts repositories list \
    --format="table(name,format,location)"

# List images in GCR (legacy registry)
gcloud container images list-tags gcr.io/neovnext/neovlab-gpu-worker \
    --format="table(digest,tags,timestamp)" \
    --sort-by=~timestamp --limit=10

# List images in GCR for the supervisor
gcloud container images list-tags gcr.io/neovnext/neovlab-gpu-supervisor \
    --format="table(digest,tags,timestamp)" \
    --sort-by=~timestamp --limit=10
```

**What to look for:**
- When was the `latest` tag last pushed? If it's stale, the deploy may be using an old image
- Multiple tags on the same digest = same image

---

## Common Issues and Diagnosis

### 1. Health check timeout (30s TaskCanceledException)

**Symptom:** C# API logs `Cloud Run GPU health check failed ... HttpClient.Timeout of 30 seconds`

**Diagnosis:**
```bash
# Check if the service is scaled to zero (cold start)
gcloud run services describe neovlab-gpu-worker-east4 \
    --region us-east4 \
    --format="value(spec.template.metadata.annotations['autoscaling.knative.dev/minScale'])"
# If 0, cold starts are expected. First request after idle will timeout.

# Check startup time of the latest revision
gcloud logging read \
    'resource.type="cloud_run_revision" AND resource.labels.service_name="neovlab-gpu-worker-east4" AND textPayload:("handler ready" OR "Application startup" OR "Uvicorn running")' \
    --limit=10 \
    --format="table(timestamp,textPayload)" \
    --freshness=1d
```

**Resolution:** Cold starts on GPU Cloud Run with model loading take 2-5 minutes. The C# API's 30s health check will always fail on cold start. This is expected behavior — the API should retry or increase the timeout.

### 2. Authentication warnings (401/403)

**Symptom:** `The request was not authenticated` warnings repeating every ~60s

**Diagnosis:**
```bash
# Check if the service requires authentication
gcloud run services describe neovlab-gpu-worker-east4 \
    --region us-east4 \
    --format="value(spec.template.metadata.annotations['run.googleapis.com/ingress'])"

# Check the invoker SA has the right role
gcloud run services get-iam-policy neovlab-gpu-worker-east4 \
    --region us-east4 \
    --format=json
```

**Resolution:** Ensure the calling service uses OIDC tokens. See IAM section in `runbook-gcp-gpu.md`.

### 3. Handler startup failures (numpy, torch, library errors)

**Symptom:** Tracebacks like `np.NaN was removed in NumPy 2.0` or `libnvrtc.so.13: cannot open shared object file`

**Diagnosis:**
```bash
# Check which handler is crashing
gcloud logging read \
    'resource.type="cloud_run_revision" AND resource.labels.service_name="neovlab-gpu-worker-east4" AND textPayload:("Application startup failed" OR "ERROR" OR "Traceback")' \
    --limit=20 \
    --format="table(timestamp,textPayload)" \
    --freshness=1d

# Check which revision the errors are on
gcloud logging read \
    'resource.type="cloud_run_revision" AND resource.labels.service_name="neovlab-gpu-worker-east4" AND severity>=ERROR' \
    --limit=5 --format=json --freshness=1d | grep revision_name
```

**Common causes:**
- `np.NaN` removed: NumPy 2.0 breaking change. Pin `numpy<2.0` in requirements or update the library using `np.NaN`
- `libnvrtc.so.13` missing: CUDA runtime toolkit not installed in Docker image. Need `nvidia-cuda-nvrtc` in the Dockerfile
- These are **code/Docker fixes** — document and hand to Go-Gasmoth/Lilithex

### 4. Diarization handler crash loop

**Symptom:** `handler=diarization` logs showing `Application startup failed` repeatedly

**Diagnosis:**
```bash
gcloud logging read \
    'resource.type="cloud_run_revision" AND resource.labels.service_name="neovlab-gpu-worker-east4" AND textPayload:("diarization")' \
    --limit=20 \
    --format="table(timestamp,textPayload)" \
    --freshness=1d
```

**Note:** The Go supervisor starts handlers as subprocesses. If a handler crashes, the supervisor may restart it. Check for repeated startup/crash cycles.

---

## Revision Comparison

When a new revision fails but the old one worked:

```bash
# Compare env vars between two revisions
gcloud run revisions describe neovlab-gpu-worker-east4-00017-7wc \
    --region us-east4 \
    --format="yaml(spec.containers[0].env)"

gcloud run revisions describe neovlab-gpu-worker-east4-00018-fcl \
    --region us-east4 \
    --format="yaml(spec.containers[0].env)"

# Compare images
gcloud run revisions describe neovlab-gpu-worker-east4-00017-7wc \
    --region us-east4 \
    --format="value(spec.containers[0].image)"

gcloud run revisions describe neovlab-gpu-worker-east4-00018-fcl \
    --region us-east4 \
    --format="value(spec.containers[0].image)"
```

---

## Database Diagnostics (Cloud SQL)

```bash
# Check job queue state
gcloud sql connect neovlab-cluster --database=cluster --user=neovlab

# Then run:
# SELECT job_id, job_type, status, error_message FROM job_queue
#  WHERE status IN ('queued','assigned','failed') ORDER BY created_at DESC LIMIT 20;
```

See `runbook-gcp-gpu.md` > "Cluster DB & IO-Cache Diagnostics" for full SQL queries.

---

## Rollback

If the latest revision is broken and a previous one was working:

```bash
# Route 100% traffic back to the known-good revision
gcloud run services update-traffic neovlab-gpu-worker-east4 \
    --region us-east4 \
    --to-revisions=neovlab-gpu-worker-east4-00017-7wc=100
```

---

## Current Known Issues (2026-03-27)

1. **`libnvrtc.so.13` missing** — `torchcodec` fails to load on revision `-00016-wrn` and `-00018-fcl`. The CUDA nvrtc library is not present in the Docker image. Needs Dockerfile fix.
2. **Diarization `np.NaN` crash** — NumPy 2.0 removed `np.NaN`. The diarization handler (pyannote pipeline) fails on startup. Needs dependency pin or code fix.
3. **Recurring 401 auth warnings** — Health check requests hitting the service without authentication. Every ~60 seconds. Likely the C# API health probe not attaching OIDC tokens correctly on retry.
4. **Revision `-00018-fcl` not serving traffic** — Created at 16:00 but traffic still routed to `-00017-7wc`. The new revision likely failed its startup health check.
