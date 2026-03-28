# Runbook: GCP to RunPod + Local Postgres Migration

> **Created:** 2026-03-27
> **Status:** DRAFT
> **Scope:** Migrate GPU compute from GCP Cloud Run to RunPod, Cloud SQL to local Docker Postgres, GCS to HTTP file transfer over Tailscale

---

## Overview

### Current Architecture
```
Steve's PC (Windows 11)
  └── C# API (local) ──→ Cloud SQL PostgreSQL (GCP us-east4, $13/mo)
                          ├── job_queue, cache_manifest, worker_registry
                          └── thumbnails, face_detections, transcriptions, etc.
  └── SQLite (local, private API DB)
  └── Uploads inputs to GCS bucket ──→ neovnext-neovlab-io-cache

Cloud Run GPU (GCP us-east4, $2.16/hr active)
  └── Go Supervisor
      ├── Polls job_queue in Cloud SQL
      ├── Reads inputs from GCS FUSE mount (/data/io-cache)
      ├── Writes outputs to GCS FUSE mount
      └── Manages Python handler subprocesses + ComfyUI

Cloud Filestore NFS (GCP us-east4, $45/mo)
  └── /models/comfyui/ (~114 GB of AI model weights)
```

### Target Architecture
```
Steve's PC (Windows 11)
  └── C# API (local)
      ├── Connects to LOCAL Docker Postgres (localhost:5432)
      ├── Exposes file transfer endpoints over Tailscale
      └── SQLite (unchanged, private API DB)
  └── Docker Postgres container (local, persistent volume)
  └── Tailscale node (mesh VPN, free tier)

RunPod On-Demand Pod (L4 GPU, $0.39/hr)
  └── Go Supervisor
      ├── Polls job_queue in Postgres via Tailscale
      ├── Downloads inputs from C# API via Tailscale HTTP
      ├── Writes outputs locally, uploads to API via Tailscale HTTP
      └── Manages Python handler subprocesses + ComfyUI
  └── RunPod Network Volume (/models, ~120 GB, $8.40/mo)
  └── Tailscale node (mesh VPN, free tier)
```

### Cost Comparison

| Component | GCP (Current) | RunPod + Local (Target) |
|-----------|---------------|-------------------------|
| GPU compute (idle) | $0 (scale-to-zero) | $0 (pod stopped) |
| GPU compute (active) | $2.16/hr | $0.39/hr |
| Database | $13/mo (Cloud SQL) | $0 (local Docker) |
| Model storage | $45/mo (Filestore 1 TiB) | $8.40/mo (RunPod NV 120 GB) |
| Object storage | $3/mo (GCS buckets) | $0 (HTTP file transfer) |
| Networking/VPN | $0 (internal) | $0 (Tailscale free tier) |
| **Monthly idle** | **$61/mo** | **$8.40/mo** |
| **Monthly (4 hrs/day GPU)** | **$320/mo** | **$55/mo** |

**Savings: ~$265/mo at typical usage. Idle cost drops from $61 to $8.40.**

---

## Part 1: GCP Scale-Down

### Prerequisites

- `gcloud` CLI authenticated: `gcloud auth login`
- Project selected: `gcloud config set project neovnext`
- Region: `us-east4`

### Step 1: Export Cloud SQL Database

Before deleting anything, export the full database.

```bash
# Option A: pg_dump via Cloud SQL public IP
pg_dump \
  --host=34.21.76.69 \
  --port=5432 \
  --username=neovlab \
  --dbname=cluster \
  --format=custom \
  --file=cluster-backup-$(date +%Y%m%d).dump

# Option B: gcloud SQL export (exports to GCS)
gcloud sql export sql neovlab-cluster \
  gs://neovnext-neovlab-io-cache/backups/cluster-$(date +%Y%m%d).sql \
  --database=cluster
```

**Verify the backup:**
```bash
pg_restore --list cluster-backup-*.dump | head -30
```

### Step 2: Scale Cloud Run GPU to Zero (Kill Switch)

Immediately stop any running GPU instances to stop billing:

```bash
gcloud run services update neovlab-gpu-worker-east4 \
  --region us-east4 \
  --min-instances 0 \
  --max-instances 0 \
  --project neovnext \
  --quiet
```

**Verify:**
```bash
gcloud run services describe neovlab-gpu-worker-east4 \
  --region us-east4 \
  --format="value(spec.template.spec.containerConcurrency,spec.template.metadata.annotations)"
```

### Step 3: Download Model Weights from GCS (if needed)

If you do not already have local copies of model weights:

```bash
# Download from GCS backup bucket (~114 GB, will take a while)
gsutil -m cp -r gs://neovnext-neovlab-models/ ./models-backup/

# Or, if Filestore is still mounted (from a GCE VM):
# scp -r <vm>:/models/comfyui/ ./models-backup/
```

These will be uploaded to a RunPod Network Volume later.

### Step 4: Delete Cloud Run Service

```bash
gcloud run services delete neovlab-gpu-worker-east4 \
  --region us-east4 \
  --project neovnext \
  --quiet
```

### Step 5: Delete Cloud SQL Instance

```bash
# Delete the database first
gcloud sql databases delete cluster \
  --instance=neovlab-cluster \
  --project neovnext \
  --quiet

# Delete the instance (this stops the $13/mo charge)
gcloud sql instances delete neovlab-cluster \
  --project neovnext \
  --quiet
```

### Step 6: Delete GCS Buckets

```bash
# Delete io-cache bucket (small, ~689 MiB)
gcloud storage rm --recursive gs://neovnext-neovlab-io-cache

# Delete models bucket (~114 GiB — only if you've confirmed local/RunPod copies)
gcloud storage rm --recursive gs://neovnext-neovlab-models

# Delete empty artifacts bucket
gcloud storage rm --recursive gs://neovnext-neovlab-artifacts
```

### Step 7: Delete Cloud Filestore Instance

```bash
gcloud filestore instances delete neovlab-models \
  --zone=us-east4-c \
  --project neovnext \
  --quiet
```

This eliminates the $45/mo Filestore charge.

### Step 8: Clean Up IAM

```bash
# Delete service accounts created for the GPU pipeline
gcloud iam service-accounts delete \
  neovlab-api-invoker@neovnext.iam.gserviceaccount.com \
  --project neovnext --quiet

gcloud iam service-accounts delete \
  neovlab-gpu-worker@neovnext.iam.gserviceaccount.com \
  --project neovnext --quiet
```

### Step 9: Delete Container Registry Images

```bash
# List images
gcloud container images list --repository=gcr.io/neovnext

# Delete the GPU worker image
gcloud container images delete gcr.io/neovnext/neovlab-gpu-worker-comfyui --quiet --force-delete-tags
```

### Step 10: Verification Checklist

After completing all deletions:

- [ ] `gcloud run services list --region us-east4` shows no neovlab services
- [ ] `gcloud sql instances list` shows no neovlab-cluster
- [ ] `gcloud filestore instances list --zone=us-east4-c` shows no neovlab-models
- [ ] `gcloud storage ls` shows no neovnext-neovlab-* buckets
- [ ] `gcloud iam service-accounts list --filter="email:neovlab"` shows no neovlab SAs
- [ ] GCP billing dashboard shows charges dropping to near-zero within 24 hours

### What to Keep in GCP

- **The project itself** (`neovnext`) — keep for DNS, potential future Cloud Run frontend hosting
- **Billing alerts** — keep configured in case of accidental resource creation
- **APIs enabled** — harmless, no cost when not in use

---

## Part 2: New Stack Setup

### Step 1: Local PostgreSQL via Docker

#### 1a. Create docker-compose file

A new `docker/docker-compose.local-infra.yml` is provided in this branch:

```yaml
version: "3.8"
services:
  cluster-db:
    image: postgres:16-alpine
    container_name: neovlab-cluster-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: cluster
      POSTGRES_USER: neovlab
      POSTGRES_PASSWORD: "${CLUSTER_DB_PASSWORD:-neovlab-local}"
    ports:
      # Bind to all interfaces so Tailscale can reach it
      - "0.0.0.0:5432:5432"
    volumes:
      - cluster-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U neovlab -d cluster"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  cluster-data:
```

#### 1b. Start Postgres

```powershell
cd Z:\NeoVNext\docker
docker compose -f docker-compose.local-infra.yml up -d
```

#### 1c. Restore the database from backup

```bash
# Restore from the pg_dump taken in Part 1
pg_restore \
  --host=localhost \
  --port=5432 \
  --username=neovlab \
  --dbname=cluster \
  --clean \
  --if-exists \
  cluster-backup-*.dump
```

Or if starting fresh, apply the schema migrations:

```bash
# Connect and run all migration files in order
psql -h localhost -U neovlab -d cluster \
  -f src/data/migrations/cluster/001_cluster_schema.sql \
  -f src/data/migrations/cluster/002_gcs_eviction.sql \
  -f src/data/migrations/cluster/003_thumbnails.sql \
  -f src/data/migrations/cluster/004_staged_inputs.sql \
  -f src/data/migrations/cluster/005_face_detections.sql \
  -f src/data/migrations/cluster/006_transcriptions.sql \
  -f src/data/migrations/cluster/007_speaker_segments.sql \
  -f src/data/migrations/cluster/008_handler_config.sql \
  -f src/data/migrations/cluster/009_pipeline_jobs_persons.sql
```

#### 1d. Update C# API connection string

In `appsettings.json` (or user secrets):
```json
{
  "ConnectionStrings": {
    "ClusterConnection": "Host=localhost;Port=5432;Database=cluster;Username=neovlab;Password=neovlab-local"
  }
}
```

#### 1e. Verify

```bash
psql -h localhost -U neovlab -d cluster -c "SELECT tablename FROM pg_tables WHERE schemaname='public';"
```

Expected tables: `job_queue`, `cache_manifest`, `worker_registry`, `handler_config`, `thumbnails`, `face_detections`, `transcriptions`, `speaker_segments`, `pipeline_jobs`, `persons`, `person_appearances`, `__migrations_applied`.

---

### Step 2: Tailscale Mesh VPN Setup

#### 2a. Install Tailscale on Steve's PC

1. Download from https://tailscale.com/download/windows
2. Install and sign in with your account (GitHub/Google/etc.)
3. Note your machine's Tailscale IP: `tailscale ip -4` (e.g., `100.x.y.z`)

#### 2b. Configure Windows Firewall for Postgres over Tailscale

Create a firewall rule allowing Postgres traffic *only* from the Tailscale interface:

```powershell
# Allow Postgres (5432) only on Tailscale interface
New-NetFirewallRule `
  -DisplayName "PostgreSQL via Tailscale" `
  -Direction Inbound `
  -Protocol TCP `
  -LocalPort 5432 `
  -InterfaceAlias "Tailscale" `
  -Action Allow
```

The Postgres container already binds to `0.0.0.0:5432`, so Tailscale traffic will reach it. The firewall rule ensures only Tailscale-originating connections are accepted.

#### 2c. Configure C# API to accept Tailscale connections

The API already listens on all interfaces. Ensure the worker token authentication is enabled so RunPod requests are authenticated.

#### 2d. Tailscale on RunPod (installed in Docker image)

The RunPod Dockerfile installs Tailscale and starts it with a pre-authenticated key. See "Step 4: RunPod Setup" below.

#### 2e. Verify Tailscale connectivity

From the RunPod pod (once set up):
```bash
# Verify Tailscale is connected
tailscale status

# Test Postgres connectivity
psql -h <steve-tailscale-ip> -U neovlab -d cluster -c "SELECT 1;"

# Test API connectivity
curl http://<steve-tailscale-ip>:5000/health
```

---

### Step 3: RunPod Network Volume Setup

#### 3a. Create a Network Volume

1. Go to https://www.runpod.io/console/user/storage
2. Create a new Network Volume:
   - **Name:** `neovlab-models`
   - **Region:** Pick the datacenter you'll use for pods (e.g., US-East, EU-RO)
   - **Size:** 150 GB (gives headroom for future models)
   - **Cost:** ~$10.50/mo

#### 3b. Upload Model Weights

Start a temporary CPU pod with the network volume attached, then upload:

```bash
# From Steve's machine, upload via rsync over Tailscale (or via RunPod's SCP)
# Or use runpodctl:
runpodctl send ./models-backup/comfyui/ --pod <pod-id>

# On the pod, organize files:
mkdir -p /runpod-volume/comfyui/checkpoints
mkdir -p /runpod-volume/comfyui/clip_vision
mkdir -p /runpod-volume/comfyui/text_encoders
mkdir -p /runpod-volume/comfyui/vae
mkdir -p /runpod-volume/comfyui/loras
mkdir -p /runpod-volume/comfyui/diffusion_models/wan

# Move files into place (preserving the directory structure from Filestore)
mv ./models-backup/comfyui/* /runpod-volume/comfyui/
```

The network volume mounts at `/runpod-volume` by default. The Dockerfile will symlink `/models/comfyui` to `/runpod-volume/comfyui`.

---

### Step 4: RunPod Pod Setup

#### 4a. Build and Push Docker Image

The updated Dockerfile (`docker/worker-gpu-go-runpod.Dockerfile`) builds for RunPod:

```bash
# Build the image
docker build -f docker/worker-gpu-go-runpod.Dockerfile -t neovlab-gpu-worker:latest .

# Tag for Docker Hub (or RunPod registry)
docker tag neovlab-gpu-worker:latest <your-dockerhub>/neovlab-gpu-worker:latest

# Push
docker push <your-dockerhub>/neovlab-gpu-worker:latest
```

#### 4b. Create RunPod Pod Template

1. Go to https://www.runpod.io/console/user/templates
2. Create a new template:
   - **Image:** `<your-dockerhub>/neovlab-gpu-worker:latest`
   - **GPU Type:** NVIDIA L4 (24 GB)
   - **Container Disk:** 20 GB
   - **Volume:** Attach `neovlab-models` network volume at `/runpod-volume`
   - **Environment Variables:**
     ```
     DATABASE_URL=postgresql://neovlab:<password>@<steve-tailscale-ip>:5432/cluster
     NEOVLAB_API_BASE_URL=http://<steve-tailscale-ip>:5000
     WORKER_TOKEN=<your-worker-token>
     TAILSCALE_AUTHKEY=tskey-auth-<your-key>
     ENABLED_HANDLERS=comfyui,face_extraction,visual_tagging,transcription,diarization
     IO_CACHE_DIR=/data/io-cache
     COMFYUI_INPUT_DIR=/comfyui/input
     MODELS_DIR=/runpod-volume/comfyui
     HF_TOKEN=<huggingface-token>
     ```
   - **Expose HTTP Ports:** 8080 (health endpoint)

#### 4c. Launch a Pod

1. Go to "Pods" -> "Deploy"
2. Select your template
3. Select GPU type (L4)
4. Deploy

#### 4d. Verify

```bash
# SSH into the pod or use RunPod terminal
tailscale status  # Should show connected to Steve's network

# Check supervisor health
curl localhost:8080/health

# Check logs
journalctl -u supervisor -f  # or check container logs
```

---

### Step 5: Go Supervisor Configuration Changes

The Go supervisor requires these environment variable changes:

| Variable | GCP Value | RunPod Value |
|----------|-----------|--------------|
| `DATABASE_URL` | `postgresql://neovlab:...@/cluster?host=/cloudsql/neovnext:us-east4:neovlab-cluster` | `postgresql://neovlab:<pass>@<tailscale-ip>:5432/cluster` |
| `CLOUD_RUN_SERVICE_URL` | `https://neovlab-gpu-worker-east4-...run.app` | (removed -- no keepalive needed) |
| `IO_CACHE_DIR` | `/data/io-cache` (GCS FUSE) | `/data/io-cache` (local disk) |
| `NEOVLAB_API_BASE_URL` | (not needed -- GCS handled I/O) | `http://<tailscale-ip>:5000` |
| `WORKER_TOKEN` | (same) | (same) |

The `CLOUD_RUN_SERVICE_URL` env var triggers the keepalive loop. When empty (RunPod), the keepalive loop is skipped -- the pod stays up until manually stopped.

New env var `NEOVLAB_API_BASE_URL` tells the supervisor where to fetch/push files via HTTP instead of relying on GCS FUSE.

---

### Step 6: C# API Configuration Changes

Update `appsettings.json`:

```json
{
  "ConnectionStrings": {
    "ClusterConnection": "Host=localhost;Port=5432;Database=cluster;Username=neovlab;Password=neovlab-local"
  },
  "Gcs": {
    "IoCacheBucketName": "",
    "IoCacheMountPath": ""
  },
  "Gpu": {
    "Enabled": true,
    "Platform": "RunPod",
    "RunPod": {
      "ApiKey": "<runpod-api-key>",
      "PodId": "<pod-id>",
      "GpuType": "NVIDIA L4"
    }
  }
}
```

Key changes:
- `ClusterConnection` points to localhost Docker Postgres
- `Gcs:IoCacheBucketName` set to empty string (disables GCS uploads in `InputStagingService`)
- `Gpu:Platform` changed from `GoogleCloud` to `RunPod`
- New `RunPod` section for pod management

---

### Step 7: Testing and Verification

#### Smoke Test 1: Database connectivity
```bash
# From Steve's PC
psql -h localhost -U neovlab -d cluster -c "SELECT COUNT(*) FROM job_queue;"
```

#### Smoke Test 2: API starts and connects to Postgres
```powershell
cd Z:\NeoVNext\src\backend\NeoVLab.Api
dotnet run
# Check logs for "Cluster DB migrations" success message
```

#### Smoke Test 3: RunPod pod can reach Postgres
```bash
# On the RunPod pod
psql -h <steve-tailscale-ip> -U neovlab -d cluster -c "SELECT COUNT(*) FROM job_queue;"
```

#### Smoke Test 4: Submit a job from API, verify worker picks it up
1. Submit a test job via the API (e.g., thumbnail generation)
2. Watch RunPod pod logs for "claimed job" message
3. Verify job transitions: queued -> assigned -> running -> completed
4. Verify artifact sync back to API

#### Smoke Test 5: ComfyUI generation
1. Submit a simple SD image generation job
2. Verify ComfyUI starts on the pod
3. Verify the generated image is synced back to the API

---

### Step 8: Cost Breakdown (Final)

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| RunPod Network Volume (150 GB) | $10.50/mo | Model weights, persistent |
| RunPod L4 Pod (on-demand) | $0.39/hr x usage | Stop when not needed |
| Local Docker Postgres | $0 | Runs on Steve's PC |
| Tailscale | $0 | Free tier (personal) |
| GCP Project (kept alive) | $0 | No active resources |
| **Idle monthly** | **$10.50/mo** | vs $61/mo on GCP |
| **Light use (2 hrs/day)** | **$34/mo** | vs $190/mo on GCP |
| **Heavy use (4 hrs/day)** | **$57/mo** | vs $320/mo on GCP |

---

## Rollback Plan

If the RunPod migration fails or is unsatisfactory:

1. **Restore Cloud SQL:** Create a new instance and restore from the backup taken in Part 1
2. **Restore Filestore:** Create a new instance and upload models from GCS backup bucket
3. **Redeploy Cloud Run:** Use the existing Docker image in GCR (if not deleted) or rebuild
4. **Revert code:** This migration is on a separate branch -- simply revert to master

The GCP project is kept alive specifically to make rollback possible.

---

## Security Considerations

1. **Postgres is NOT exposed to the internet.** It binds to `0.0.0.0:5432` on Steve's machine, but Windows Firewall only allows connections from the Tailscale interface. All traffic is encrypted by Tailscale's WireGuard tunnel.

2. **Worker authentication.** The `WORKER_TOKEN` env var authenticates the Go supervisor's API requests. The same token mechanism used in Cloud Run continues to work.

3. **Tailscale auth keys.** The RunPod pod uses a pre-authenticated Tailscale key (`TAILSCALE_AUTHKEY`). Generate these with expiry and tag them as `tag:runpod-worker` for ACL management.

4. **Tailscale ACLs.** Configure Tailscale ACLs to restrict the RunPod node to only reach Steve's machine on ports 5432 (Postgres) and 5000 (API). No other access needed.

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["tag:runpod-worker"],
      "dst": ["autogroup:owner:5432", "autogroup:owner:5000"]
    }
  ]
}
```
