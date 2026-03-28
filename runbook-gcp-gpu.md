# GCP Cloud Run GPU Runbook

> Frugal MVP testing guide. Goal: prove GPU workers work on Cloud Run, then shut everything down.
> Expected cost: **$0 - $2** for a test session using free-tier credits.

---

## Prerequisites

| Item | How to check |
|------|-------------|
| Google Cloud account with billing enabled | [console.cloud.google.com](https://console.cloud.google.com) |
| Free credits ($300 trial or promo) | Billing > Overview > Credits |
| `gcloud` CLI installed | `gcloud --version` |
| Docker Desktop running | `docker info` |
| NeoVNext repo cloned | `ls src/backend` |

---

## Phase 1: One-Time GCP Setup (~10 min)

### 1.1 Authenticate and set project

```bash
gcloud auth login
gcloud config set project YOUR_PROJECT_ID

# Verify
gcloud config get-value project
```

### 1.2 Enable required APIs

```bash
gcloud services enable \
    run.googleapis.com \
    cloudbuild.googleapis.com \
    storage.googleapis.com \
    artifactregistry.googleapis.com \
    sqladmin.googleapis.com
```

### 1.3 Create Cloud SQL cluster database

The pull-model GPU workers use a PostgreSQL database (`cluster`) to claim jobs
and register themselves. This must exist before any worker can start.

```bash
# Create the Cloud SQL instance (micro tier, ~$8/month or free-tier eligible)
gcloud sql instances create neovlab-cluster \
    --database-version=POSTGRES_16 \
    --tier=db-f1-micro \
    --region=us-east4 \
    --edition=ENTERPRISE \
    --project=YOUR_PROJECT_ID

# Create the database and user
gcloud sql databases create cluster --instance=neovlab-cluster --project=YOUR_PROJECT_ID
gcloud sql users create neovlab --instance=neovlab-cluster --password=neovlab --project=YOUR_PROJECT_ID
```

### 1.4 Apply the cluster schema

Two migration files must be applied in order:
- `src/data/migrations/cluster/001_cluster_schema.sql` — creates `job_queue`, `cache_manifest`, `worker_registry`
- `src/data/migrations/cluster/002_gcs_eviction.sql` — adds `gcs_evicted_at` column to `cache_manifest` (required for GCS eviction tracking)

```bash
# Grant the SQL service account read access to the GCS bucket (one-time)
SA=$(gcloud sql instances describe neovlab-cluster --project=YOUR_PROJECT_ID \
    --format="value(serviceAccountEmailAddress)")
gcloud storage buckets add-iam-policy-binding gs://YOUR_PROJECT_ID-neovlab-io-cache \
    --member="serviceAccount:$SA" --role="roles/storage.objectViewer"

# Upload and apply 001 (base schema)
gcloud storage cp src/data/migrations/cluster/001_cluster_schema.sql \
    gs://YOUR_PROJECT_ID-neovlab-io-cache/migrations/001_cluster_schema.sql
gcloud sql import sql neovlab-cluster \
    gs://YOUR_PROJECT_ID-neovlab-io-cache/migrations/001_cluster_schema.sql \
    --database=cluster --user=neovlab --project=YOUR_PROJECT_ID --quiet

# Upload and apply 002 (GCS eviction column)
gcloud storage cp src/data/migrations/cluster/002_gcs_eviction.sql \
    gs://YOUR_PROJECT_ID-neovlab-io-cache/migrations/002_gcs_eviction.sql
gcloud sql import sql neovlab-cluster \
    gs://YOUR_PROJECT_ID-neovlab-io-cache/migrations/002_gcs_eviction.sql \
    --database=cluster --user=neovlab --project=YOUR_PROJECT_ID --quiet
```

> **Verify**: `gcloud sql connect neovlab-cluster --database=cluster --user=neovlab`
> then `\dt` should show `job_queue`, `cache_manifest`, `worker_registry`.
> `\d cache_manifest` should include column `gcs_evicted_at timestamptz`.

### 1.5 Set a billing alert (safety net)

```bash
# Quick way via the script:
BILLING_ACCOUNT=your-billing-account-id \
BUDGET_AMOUNT=8.00 \
bash docker/configure-billing-alert.sh YOUR_PROJECT_ID

# Or manually: Cloud Console > Billing > Budgets & alerts > Create
# Amount: $8, alerts at 50%, 80%, 100%
# Filter: Cloud Run only
```

### 1.6 Create GCS buckets

```bash
# I/O cache (for schema uploads and job input/output files)
gsutil mb -l us-east4 gs://YOUR_PROJECT_ID-neovlab-io-cache

# Model storage (for ComfyUI checkpoints) -- skip if not testing ComfyUI
gsutil mb -l us-east4 gs://YOUR_PROJECT_ID-neovlab-models

# Populate with SDXL base model (~7 GB, ~5 min on fast connection)
MODEL_VOLUME_ROOT=./comfyui-models bash docker/populate-model-volume.sh --gcs gs://YOUR_PROJECT_ID-neovlab-models
```

> **Cost**: GCS Standard ~$0.02/GB/month. 7 GB = ~$0.14/month. Delete when done testing.

---

## Phase 1.7: Understanding Model Storage and Caching

### Storage Architecture

The GPU worker uses a hybrid approach for model storage:

**GCS FUSE Mounts (Network Storage):**
- Models stored in `gs://YOUR_PROJECT-neovlab-models/`
- Mounted at `/models/comfyui` in the container
- **Problem:** Streaming multi-GB models over the network on every cold start is slow (~5+ min for large diffusion models)

**Ephemeral Storage Cache (Free, Fast):**
- Cloud Run provides up to **32GB of ephemeral storage at no cost**
- Located at `/tmp` in the container
- Persists for the container lifetime (lost on scale-to-zero)
- Perfect for caching frequently-used models

### Current Caching Strategy

The worker automatically caches small, frequently-accessed models on startup:

| Model Type | Size | Cached? | Load Time | Location |
|------------|------|---------|-----------|----------|
| VAE | ~254MB | ✅ Yes | ~3s | /tmp/models/vae |
| Text Encoders | ~7GB | ✅ Yes | ~110s | /tmp/models/text_encoders |
| Primary Checkpoint (ponyRealism) | ~7GB | ✅ Yes | ~110s | /tmp/models/checkpoints |
| Other Checkpoints (animagine, ponyDiffusion) | ~14GB | ❌ No | ~3-5min | GCS FUSE (on-demand) |
| Diffusion Models (WAN) | ~91GB | ❌ No | ~5-15min | GCS FUSE (on-demand) |

**Total cached:** ~14GB (~44% of free ephemeral storage)  
**Cache population time:** ~3m 40s (221s) on cold start  
**ComfyUI startup:** ~38s  
**Total cold start:** ~4m 19s (259s)  
**Cost:** $0 (ephemeral storage is free)

**Why ponyRealism?** This is the most commonly used checkpoint in smoke tests and typical workflows.
Other checkpoints (animagineXL, ponyDiffusionV6XL) stream from GCS on first use.

### Performance Impact

**Before caching (all models on GCS FUSE):**
- Cold start: 5+ minutes stuck loading models over network
- Jobs frequently timed out waiting for models

**After caching (VAE + text encoders + primary checkpoint cached):**
- Cold start: ~4m 19s total (3m 40s cache population + 38s ComfyUI startup)
- VAE loads instantly from cache
- Text encoders load instantly from cache (was 5+ minutes streaming from GCS)
- Primary checkpoint (ponyRealism) loads instantly from cache (was 5-10 minutes streaming from GCS)
- ComfyUI startup: 38s (model initialization, database migrations)
- Other checkpoints still stream from GCS on first use (~5-10 min)

### Disabling the Cache

If you need to disable caching (testing, debugging, etc.):

```bash
gcloud run services update neovlab-gpu-worker-east4 \
    --region us-east4 \
    --update-env-vars ENABLE_MODEL_CACHE=false
```

The entrypoint script (`docker/comfyui-entrypoint.sh`) checks the `ENABLE_MODEL_CACHE` env var.

---

## Phase 2: Deploy GPU Worker (~5 min)

### 2.1 Build and deploy

```bash
cd /path/to/NeoVNext

# Default deployment target: neovnext / us-east4 / service: neovlab-gpu-worker-east4
# REBUILD_IMAGE=1 builds + pushes the image first; REBUILD_IMAGE=0 (default) is a safe re-deploy.
REBUILD_IMAGE=1 bash docker/deploy-gpu-worker.sh neovnext us-east4 east4
```

To rebuild and push the image from Windows (PowerShell) before deploying:
```powershell
docker build -t gcr.io/neovnext/neovlab-gpu-worker-comfyui:latest -f docker/worker-comfyui.Dockerfile .
docker push gcr.io/neovnext/neovlab-gpu-worker-comfyui:latest
# Then deploy (safe mode, image already pushed):
$env:REBUILD_IMAGE = "0"; bash docker/deploy-gpu-worker.sh neovnext us-east4 east4
```

The script outputs:
- **Service URL** (e.g., `https://neovlab-gpu-worker-xyz.run.app`)
- **Worker Token** (64-char hex — save this)

### 2.2 Cloud Run Configuration Details

The deployment configures the following Cloud Run settings optimized for model caching:

**Resource Limits:**
- 4 vCPU
- 16GB RAM
- 1× NVIDIA L4 GPU (22GB VRAM)
- Concurrency: 1 (one job per instance)

**Startup Probe (Critical for Model Caching):**
- Type: TCP socket on port 8080
- Initial Delay: 240s (4 minutes) — allows time for cache population
- Period: 10s — check every 10 seconds after initial delay
- Failure Threshold: 6 — allows 60s of retries (6 × 10s)
- **Total startup window: 300s (5 minutes)** — covers 4m19s cache + startup time

**Why these settings?**
- Cloud Run caps `initialDelaySeconds` at 240s (4 min) and `periodSeconds` at 240s
- Our cache takes ~3m 40s + ComfyUI startup 38s = ~4m 19s total
- The 240s initial delay + 60s retry window (failureThreshold=6) = 300s total
- This gives us 41s buffer for slowdowns without failed deployments

**Request Timeout:**
- 1200s (20 minutes) — allows time for slow GCS streaming if cache disabled or secondary checkpoints needed

### 2.3 Configure the backend

```bash
cd src/backend/NeoVLab.Api

# Store secrets (NEVER commit these)
dotnet user-secrets set "Gpu:Enabled" "true"
dotnet user-secrets set "Gpu:Platform" "GoogleCloud"
dotnet user-secrets set "Gpu:GoogleCloud:ProjectId" "neovnext"
dotnet user-secrets set "Gpu:GoogleCloud:Region" "us-east4"
dotnet user-secrets set "Gpu:GoogleCloud:ServiceName" "neovlab-gpu-worker-east4"
dotnet user-secrets set "Gpu:GoogleCloud:WorkerToken" "THE_TOKEN_FROM_DEPLOY"

# Budget: $5 cap (500 cents). Adjust as needed.
dotnet user-secrets set "Gpu:BudgetCents" "500"
```

### 2.4 Worker Keepalive and Scale-to-Zero

**Problem:** Cloud Run scales services to zero after inactivity. Since workers use database heartbeats (not HTTP requests) to maintain job leases, Cloud Run's scale-to-zero logic sees no HTTP traffic and terminates the instance **mid-job execution**, causing job failures.

**Solution:** Workers implement a **smart keepalive mechanism** that:
1. Pings the worker's own **external URL** every 60 seconds (generates real HTTP traffic Cloud Run recognizes)
2. Continues for a **configurable idle timeout** after job completion (default: 10 minutes)
3. **Automatically stops** after idle period to allow cost-saving scale-to-zero

**Configuration:**
```bash
# Set idle timeout (minutes) before allowing scale-to-zero
--set-env-vars="WORKER_IDLE_TIMEOUT_MINUTES=10"  # Default: 10 min
```

**What happens:**
```
Job arrives → Worker scales up → Keepalive starts
Job executing → Keepalive pings every 60s → Worker stays alive
Job completes → Idle timer starts (10 min)
10 min idle → Keepalive stops → Worker scales to zero
```

**Monitoring keepalive in logs:**
```bash
# Check keepalive started
gcloud logging read 'resource.labels.service_name=neovlab-gpu-worker-east4' \
    --limit=20 --format='value(textPayload)' | grep "keepalive"

# Expected output:
# Worker keepalive started: pinging https://neovlab-gpu-worker-east4-....run.app/health every 60s (idle timeout: 10 min)
# Worker keepalive ping succeeded (next ping in 60s).
# Worker idle for 0:10:03 (> 10 min threshold). Stopping keepalive to allow scale-to-zero.
```

**Key behaviors:**
- External URL discovered via Cloud Run metadata server (no manual configuration)
- Includes worker auth token for authenticated /health endpoint
- Non-fatal failures (logged as warnings, don't kill the worker)
- Resets idle timer when new job starts
- Falls back to localhost for local development

**Cost implications:**
- **Without keepalive:** Workers killed mid-job, wasted GPU time + job failures
- **With keepalive + idle timeout:** Workers stay alive during jobs + brief grace period, then scale to zero → optimal cost/reliability balance
- Setting `WORKER_IDLE_TIMEOUT_MINUTES=0` would allow immediate scale-to-zero after jobs (may cause cold starts if jobs arrive frequently)

### 2.5 Verify deployment

```bash
# Check service exists and is set to scale-to-zero
gcloud run services describe neovlab-gpu-worker-east4 \
    --region us-east4 \
    --format "table(status.url, spec.template.spec.containers[0].resources.limits)"

# Should show:
#   min-instances: 0  (scale to zero = no idle cost)
#   max-instances: 1
#   gpu: 1 nvidia-l4
```

---

## Phase 3: Test It (~5 min)

### 3.1 Start the local stack

Cloud mode connects the API to Cloud SQL and enables GCS io-cache.

```powershell
# Start API (CloudDev profile) + frontend
.\Start-NeoVLab.ps1 start all -Cloud

# Check status / stop
.\Start-NeoVLab.ps1 status
.\Start-NeoVLab.ps1 stop
```

API runs at `http://localhost:6000`, frontend at `http://localhost:6100`.

> The `-Cloud` flag launches the API with `--launch-profile CloudDev`, which sets
> `ASPNETCORE_ENVIRONMENT=CloudDev` so the API reads Cloud SQL connection strings
> and enables GCS artifact fallback.

### 3.2 Verify GPU status from the API

```bash
# Get auth token
TOKEN=$(curl -s http://localhost:6000/api/auth/token | jq -r .token)

# Check GPU status
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:6000/api/gpu/status | jq .

# Expected: { "enabled": true, "platform": "GoogleCloud", "healthy": true/false, ... }
```

> First request triggers a Cloud Run cold start (~15-30s). This is normal.

### 3.3 Submit a test job

From the frontend at `http://localhost:6100`:
1. Go to **Settings** > verify GPU shows as enabled
2. Go to **Jobs** > submit a GPU-eligible job (visual tagging or ComfyUI)
3. Watch the job lifecycle: queued -> dispatched -> running -> completed

Or via API:
```bash
curl -X POST http://localhost:6000/api/jobs \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"mediaId": "YOUR_MEDIA_ID", "jobType": "visual_tagging_gpu"}'
```

### 3.4 Check budget tracking

```bash
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:6000/api/budget | jq .
```

---

## Phase 4: Shut It Down

### Option A: Pause (keeps deployment, zero cost when idle)

Cloud Run with `min-instances: 0` already costs $0 when idle. But if you want to
be extra safe and prevent ANY cold-start charges from accidental requests:

```bash
# Sets max-instances to 0 -- nothing can start
bash docker/gpu-pause.sh

# Or directly:
gcloud run services update neovlab-gpu-worker-east4 \
    --region us-east4 \
    --max-instances 0
```

**Resume later:**
```bash
bash docker/gpu-resume.sh
```

### Option B: Full teardown (delete everything)

```bash
# Delete the Cloud Run service
gcloud run services delete neovlab-gpu-worker-east4 --region us-east4 --quiet

# Delete the Cloud SQL instance (stops billing immediately)
gcloud sql instances delete neovlab-cluster --project=YOUR_PROJECT_ID --quiet

# Delete the container image from GCR
gcloud container images delete gcr.io/YOUR_PROJECT_ID/neovlab-gpu-worker-comfyui --quiet --force-delete-tags

# Delete GCS buckets
gsutil rm -r gs://YOUR_PROJECT_ID-neovlab-models
gsutil rm -r gs://YOUR_PROJECT_ID-neovlab-io-cache

# Disable GPU in backend
cd src/backend/NeoVLab.Api
dotnet user-secrets set "Gpu:Enabled" "false"
```

### Option C: Disable in app only (keep cloud resources paused)

```bash
dotnet user-secrets set "Gpu:Enabled" "false"
# Restart backend -- all GPU jobs route to CPU fallback automatically
```

---

## Cost Cheat Sheet

| Resource | When running | When idle (min=0) | When paused (max=0) | When deleted |
|----------|-------------|-------------------|---------------------|-------------|
| Cloud Run GPU (L4) | ~$0.80/hr | $0 | $0 | $0 |
| Cloud SQL (db-f1-micro) | ~$8/mo | ~$8/mo | ~$8/mo | $0 |
| GCS buckets (7 GB models) | $0.14/mo | $0.14/mo | $0.14/mo | $0 |
| Container image in GCR | ~$0.01/mo | ~$0.01/mo | ~$0.01/mo | $0 |
| **Ephemeral storage (32GB)** | **$0** | **$0** | **$0** | **$0** |
| **Total idle** | | **~$8.15/mo** | **~$8.15/mo** | **$0** |

> **Note on ephemeral storage:** Cloud Run provides up to 32GB of ephemeral storage (in-memory tmpfs)
> at no cost. The worker uses ~7GB to cache frequently-accessed models, significantly improving
> cold start performance without adding any cost.

**Typical test session** (deploy, run 5-10 jobs, tear down): **$0.50 - $1.50**

---

## IAM & Security

### Service Accounts

Each Cloud Run service runs under a dedicated, least-privilege service account
instead of the default Compute Engine SA (which typically has broad `Editor` access).

| Service Account | Used By | Roles |
|-----------------|---------|-------|
| `neovlab-api-invoker@<PROJECT>.iam` | Local API (CloudDev profile) | `roles/run.invoker` on GPU worker service |
| `neovlab-gpu-worker@<PROJECT>.iam` | GPU worker Cloud Run service | `roles/run.developer`, `roles/cloudsql.client`, `roles/storage.objectUser` |

**Local development:** The CloudDev launch profile sets `GOOGLE_APPLICATION_CREDENTIALS`
to a key file for the invoker SA. This key is gitignored (`*-key.json`).

### Cloud SQL SSL

SSL is enforced on the Cloud SQL instance (`requireSsl=true`). Connection strings use
`SslMode=Require;Trust Server Certificate=true`.

### OIDC Authentication Flow

The API authenticates to Cloud Run GPU workers via OIDC identity tokens.
`GoogleCloudGpuPlatformAdapter` tries three methods in order:

1. **ADC** (`GoogleCredential.GetApplicationDefaultAsync`) -- works with SA key files
2. **gcloud with `--audiences`** -- works for service accounts via CLI
3. **gcloud without `--audiences`** -- fallback for user accounts (`gcloud auth login`)

If you get 403 errors calling the GPU worker, check:
- The calling identity has `roles/run.invoker` on the target service
- `GOOGLE_APPLICATION_CREDENTIALS` points to a valid key file (CloudDev profile)
- Or `gcloud auth login` is active with an account that has `run.invoker`

---

## Safety Controls Summary

| Control | What it does | Command |
|---------|-------------|---------|
| **Least-privilege SAs** | Each service has minimum IAM roles | See IAM & Security section above |
| **Cloud SQL SSL** | Encrypted DB connections required | `requireSsl=true` on instance |
| **Scale-to-zero** | No cost when no requests | Default (`min-instances: 0`) |
| **App budget cap** | Blocks new GPU dispatch at $5/mo | `Gpu:BudgetCents = 500` in user-secrets |
| **80% budget warning** | SignalR event to frontend | Automatic at 80% of cap |
| **Pause script** | Sets max-instances=0 instantly | `bash docker/gpu-pause.sh` |
| **Resume script** | Restores max-instances=1 | `bash docker/gpu-resume.sh` |
| **API kill switch** | Pause from the app UI/API | `POST /api/gpu/pause` |
| **GCP billing alert** | Email at 80%/100% of $8 | `bash docker/configure-billing-alert.sh` |
| **Full teardown** | Delete all cloud resources | See Phase 4 Option B above |

---

## Quick Reference Card

```
# DEPLOY (rebuild image + push + deploy)
REBUILD_IMAGE=1 bash docker/deploy-gpu-worker.sh neovnext us-east4 east4

# SAFE RE-DEPLOY (image already pushed)
bash docker/deploy-gpu-worker.sh neovnext us-east4 east4

# START LOCAL SERVICES IN CLOUD MODE (API w/ CloudDev profile + frontend)
.\Start-NeoVLab.ps1 start all -Cloud

# PAUSE (stop all GPU, keep deployment)
bash docker/gpu-pause.sh

# RESUME
bash docker/gpu-resume.sh

# CHECK STATUS
curl -H "Authorization: Bearer $TOKEN" http://localhost:6000/api/gpu/status

# CHECK SPEND
curl -H "Authorization: Bearer $TOKEN" http://localhost:6000/api/budget

# NUKE EVERYTHING
gcloud run services delete neovlab-gpu-worker-east4 --region us-east4 --quiet
gcloud sql instances delete neovlab-cluster --quiet
gsutil rm -r gs://neovnext-neovlab-models
gsutil rm -r gs://neovnext-neovlab-io-cache
dotnet user-secrets set "Gpu:Enabled" "false"
```

---

## Cluster DB & IO-Cache Diagnostics

Use these queries when jobs are stuck or artifacts are missing.

### Check job state

```bash
# Connect to cluster DB
gcloud sql connect neovlab-cluster --database=cluster --user=neovlab --project=YOUR_PROJECT_ID
```

```sql
-- Jobs stuck in queued or assigned
SELECT job_id, job_type, status, created_at, assigned_at
  FROM job_queue
 WHERE status IN ('queued', 'assigned')
 ORDER BY created_at DESC
 LIMIT 20;

-- Recent failures
SELECT job_id, job_type, status, error_message, updated_at
  FROM job_queue
 WHERE status = 'failed'
 ORDER BY updated_at DESC
 LIMIT 10;
```

### Check IO-cache manifest

```sql
-- Cache entries for a specific job
SELECT cache_path, direction, size_bytes, synced_locally, evicted_at
  FROM cache_manifest
 WHERE job_id = '<job_id>';

-- Jobs with output not yet synced to local storage
SELECT job_id, cache_path, synced_locally, synced_at
  FROM cache_manifest
 WHERE direction = 'output'
   AND synced_locally = FALSE
   AND evicted_at IS NULL;
```

### API observability endpoints (no SQL required)

```bash
# Real-time queue depth
curl -H "Authorization: Bearer <token>" https://<api-url>/api/observability/queue-depth

# Rolling error rate (last 15 minutes)
curl -H "Authorization: Bearer <token>" https://<api-url>/api/observability/error-rate?windowMinutes=15

# GCS sync diagnostics (CloudDev/Development environments only)
# Shows: pending jobs, manifest stats (total/syncedLocally/gcsEvicted/missedLocal), GCS config
curl http://localhost:6000/api/debug/sync-status
```

### Enable verbose GPU worker logging

To get detailed artifact download/upload logs from the Cloud Run worker:

```bash
gcloud run services update neovlab-gpu-worker-east4 \
    --region us-east4 \
    --set-env-vars ARTIFACT_DEBUG=true

# Tail logs
gcloud logging tail "resource.type=cloud_run_revision AND resource.labels.service_name=neovlab-gpu-worker-east4" \
    --format="value(textPayload)"

# Disable when done
gcloud run services update neovlab-gpu-worker-east4 \
    --region us-east4 \
    --set-env-vars ARTIFACT_DEBUG=false
```

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Cold start > 60s | Normal for first request. Subsequent requests within idle window are fast. |
| **Job status stuck "running", worker scaled to zero** | **Keepalive issue.** Check: 1) Worker logs for "Worker keepalive started" message, 2) External URL discovery succeeded (not localhost fallback), 3) Worker token configured correctly. See section 2.4 for keepalive details. |
| **Jobs interrupted after ~5 minutes** | **Scale-to-zero killed worker during execution.** Verify keepalive pings appearing in logs every 60s. If seeing localhost URL, worker cannot reach metadata server — check Cloud Run service configuration. |
| **Worker never scales to zero when idle** | Keepalive not stopping after idle timeout. Check `WORKER_IDLE_TIMEOUT_MINUTES` env var (default 10 min). Look for "Stopping keepalive to allow scale-to-zero" message in logs. |
| `GPU_BUDGET_EXCEEDED` (402) | Budget cap hit. Increase via `dotnet user-secrets set "Gpu:BudgetCents" "1000"` or wait for monthly reset. |
| `MODEL_NOT_FOUND` (422) | ComfyUI model not in GCS bucket. Run `populate-model-volume.sh --gcs`. |
| `relation "worker_registry" does not exist` | Cluster schema not applied to Cloud SQL. Re-run Phase 1.4 (upload SQL to GCS and import). |
| `relation "job_queue" does not exist` | Same as above -- apply cluster schema. |
| Worker starts but jobs stay `queued` | Check: 1) Cloud SQL schema applied, 2) worker can reach DB, 3) `CLUSTER_DB_*` env vars correct on Cloud Run service.  |
| `could not connect to server: Connection timed out` | Cloud Run service needs Cloud SQL connector or authorized networks on the SQL instance. Workers use `CLUSTER_DB_HOST` env var. |
| **403 calling GPU worker** | OIDC auth failure. Check: 1) `GOOGLE_APPLICATION_CREDENTIALS` set in CloudDev profile, 2) SA has `run.invoker` on target service, 3) Try `gcloud auth print-identity-token` to verify token generation. See IAM & Security section. |
| **"Invalid account type for --audiences"** | Using `gcloud auth login` (user account). The adapter auto-falls back to no-audiences mode. If still failing, set `GOOGLE_APPLICATION_CREDENTIALS` to an invoker SA key file. |
| Health check fails | Cloud Run auto-starts on request. Wait 30s and retry. |
| Unexpected charges | Run `bash docker/gpu-pause.sh` immediately, then check Cloud Console billing. |

---

## Log Locations

| Location | Path / Command | Notes |
|----------|----------------|-------|
| Local API file log | logs/neovlab-api.log (relative to API content root) | Written by the built-in FileLoggerProvider; controlled by Logging:File:Enabled in ppsettings.json |
| Local API console | stdout / VS Code Debug Console | Default when running dotnet run or via the desktop launcher |
| Cloud Run API logs | gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=neovlab-api" --limit=50 --format=json | Structured JSON; filter by severity or jsonPayload.message |
| Cloud Run GPU worker logs | gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=neovlab-gpu-worker-east4" --limit=50 --format=json | Includes ComfyUI stdout forwarded by the entrypoint script |
| Cloud SQL (cluster DB) logs | GCP Console > SQL > neovlab-cluster > Logs, or gcloud logging read "resource.type=cloudsql_database" | Query logs; useful for diagnosing schema / connection errors |
| Cluster DB diagnostics (spec 005) | GET /api/observability/queue-depth and GET /api/observability/error-rate | Returns live queue depth and rolling error-rate from the cluster DB |

