# Cloud Migration Research: Alternative Providers for NeoVNext

> **Date**: 2026-03-27
> **Author**: Lilithex (Supreme Matriarch Succubi Orchestrator)
> **Status**: Research / Decision Pending
> **Audience**: Steve + NeoVNext engineering

---

## Executive Summary

NeoVNext currently runs GPU workloads on **GCP Cloud Run with NVIDIA L4 GPUs** in us-east4, backed by **Cloud SQL PostgreSQL** (db-f1-micro), **GCS buckets** for IO cache, and **Filestore NFS** for ComfyUI model storage. The architecture is well-suited to bursty, scale-to-zero inference but carries vendor lock-in through Cloud Run GPU-specific features (NFS volume mounts, GCS FUSE, Cloud SQL sidecar proxy).

**Key findings:**

1. **GPU-only migration to RunPod** would save approximately 40-50% on GPU compute costs ($0.39/hr vs $0.67/hr for L4-equivalent), but requires replacing the Cloud Run container model with RunPod Pods or Serverless endpoints and self-managing database connectivity.

2. **GPU-only migration to Modal** offers the cleanest developer experience with per-second billing ($0.80/hr L4), cold starts under 1 second, and a Python-native SDK. However, it requires significant rearchitecting of the Go supervisor into Modal's function-based paradigm.

3. **Full-stack migration to AWS** (ECS/Fargate + RDS + S3 + EC2 G6) provides the broadest feature parity and would reduce GPU costs (~$1.32/hr for g6.xlarge but with more CPU/RAM), with mature Postgres (RDS), object storage (S3), and container hosting (ECS/Fargate). Migration is substantial but well-documented.

4. **Full-stack migration to Hetzner** offers dramatic cost reduction for non-GPU workloads (VPS at 1/5 the hyperscaler price) but has limited GPU options (no L4; RTX 4000 SFF Ada at ~$212/mo dedicated) and no managed serverless containers.

**Recommendation**: **Hybrid approach** -- migrate GPU workloads to **RunPod** (Pods with network volumes) while keeping non-GPU services on GCP (or migrating them to a cheaper provider later). This captures the biggest cost savings immediately with the lowest migration risk.

---

## Current GCP Architecture Inventory

### Services in Use

| GCP Service | Purpose | Config | Monthly Cost (est.) |
|---|---|---|---|
| **Cloud Run** (GPU) | GPU worker: Go supervisor + Python ML handlers | 1x L4, 4 vCPU, 16GB RAM, min=0/max=1 | ~$0.67/hr when running; $0 idle |
| **Cloud SQL** (PostgreSQL 16) | Cluster database: job queue, cache manifest, worker registry | db-f1-micro, us-east4 | ~$9.37/mo |
| **GCS Bucket** (io-cache) | Input/output artifact staging between API and GPU worker | Standard tier, us-east4 | ~$0.02/GB/mo |
| **GCS Bucket** (models) | ComfyUI model checkpoints (SDXL base, LoRAs) | Standard tier, us-east4, ~7GB | ~$0.14/mo |
| **Filestore NFS** | Shared model storage mounted at /models/comfyui | Basic HDD, 1TB minimum | ~$190/mo (minimum) |
| **Container Registry** (GCR) | Docker image storage for GPU worker | ~15GB image | ~$0.30/mo |
| **Cloud Logging** | Log aggregation for Cloud Run services | Default | Free tier typically |
| **IAM / Service Accounts** | neovlab-gpu-worker@, neovlab-api-invoker@ | Least-privilege roles | $0 |
| **VPC / Networking** | Default VPC, private ranges egress | Standard | ~$0 (minimal egress) |

**Estimated total monthly cost (light usage, ~20 GPU-hours/mo):** ~$215/mo
- Filestore: $190 (the dominant cost by far)
- Cloud SQL: $9.37
- GPU compute: ~$13.40 (20 hrs x $0.67)
- Storage/misc: ~$2

### GCP SDK / API Integration Points in Codebase

| Component | GCP Dependency | Files |
|---|---|---|
| **C# API** | `Google.Cloud.Storage.V1` NuGet package | `ArtifactSyncService.cs`, `CacheEvictionService.cs`, `InputStagingService.cs` |
| **C# API** | `Google.Apis.Auth` NuGet package (OIDC tokens) | `GoogleCloudGpuPlatformAdapter.cs` |
| **C# API** | `gcloud` CLI (shelled out for Cloud Run scaling) | `GoogleCloudGpuPlatformAdapter.cs` |
| **C# API** | Cloud SQL socket path (`/cloudsql/...`) | `NpgsqlConnectionFactory.cs` |
| **C# API** | GCS config keys (`Gcs:IoCacheBucketName`, `Gcs:IoCacheMountPath`) | `appsettings.*.json` |
| **Go Supervisor** | Cloud Run env vars (`CLOUD_RUN_SERVICE_URL`) | `config.go` |
| **Go Supervisor** | Cloud SQL connection string (`/cloudsql/...` prefix) | `config.go` |
| **Deploy scripts** | `gcloud run deploy`, `gsutil`, `docker push gcr.io/...` | `gpu-worker.ps1`, `deploy-gpu-worker-go.sh` |
| **Dockerfile** | No GCP-specific dependencies (generic CUDA base) | `worker-gpu-go.Dockerfile` |

### Architecture Diagram (Text)

```
[Desktop App (Electron)]
    |
    v
[C# .NET API (local)]
    |-- SQLite (local media DB)
    |-- Npgsql --> [Cloud SQL PostgreSQL] <-- [Go GPU Supervisor]
    |-- GCS SDK --> [GCS io-cache bucket] <-- [GCS FUSE mount in Cloud Run]
    |
    v (OIDC auth)
[Cloud Run GPU Service]
    |-- L4 GPU (nvidia-l4)
    |-- Filestore NFS --> /models/comfyui
    |-- GCS FUSE --> /data/io-cache
    |-- Cloud SQL sidecar --> PostgreSQL
    |
    [Go Supervisor]
        |-- Python handler subprocesses
        |   (transcription, diarization, face_extraction,
        |    visual_tagging, comfyui, face_clustering, voice_face_association)
```

---

## Provider Comparison Table: GPU-Only Alternatives

These providers would replace only the Cloud Run GPU worker. The API, database, and object storage would remain on GCP (or be migrated separately).

| Provider | L4 Available? | L4 Price/hr | Alt GPU | Alt Price/hr | Scale-to-Zero | Docker Support | NFS/Shared Storage | Managed DB |
|---|---|---|---|---|---|---|---|---|
| **GCP Cloud Run** (current) | Yes | $0.67 | -- | -- | Yes (native) | Yes | Filestore NFS mount | Cloud SQL sidecar |
| **RunPod Pods** | Yes | $0.39 | A10 ($0.44) | $0.44 | No (manual stop) | Yes | Network Volumes | No (external) |
| **RunPod Serverless** | Yes | $0.77 | A10 ($0.44) | -- | Yes (native) | Yes (custom handler) | Limited | No |
| **Modal** | Yes | $0.80 | A10 ($1.10) | -- | Yes (native) | Yes (via Image) | Volumes (not NFS) | No (external) |
| **Vast.ai** | Yes | ~$0.20-0.40 | A10 (~$0.25) | -- | No | Yes | No | No |
| **Replicate** | No (managed models) | -- | -- | -- | Yes | Limited | No | No |
| **Fly.io** | No L4 (deprecated GPUs) | -- | A10 ($0.75)* | -- | Yes (auto-stop) | Yes | Volumes | Managed Postgres |
| **Paperspace** (DigitalOcean) | No L4 | -- | A4000 (~$0.45) | -- | No | Yes | Shared drives | No |
| **Vultr GPU** | No L4 | -- | L40S ($1.67) | -- | No | Yes | Block storage | Managed DB |
| **CoreWeave** | No L4 | -- | L40 ($1.25/GPU) | -- | No | Yes (K8s) | Shared FS | No |
| **Lambda Labs** | No L4 | -- | A10 ($0.75) | -- | No | Yes | Shared FS | No |

*Fly.io GPUs deprecated August 2025; no longer recommended.

### Key Observations

1. **RunPod is the clear winner for L4 price/performance** at $0.39/hr on-demand, nearly half the GCP Cloud Run price.
2. **Vast.ai is cheapest** but is a marketplace with variable reliability -- not suitable for production workloads.
3. **Modal offers the best DX** with sub-second cold starts and per-second billing, but requires rearchitecting the Go supervisor.
4. **Most GPU-only providers lack managed Postgres and NFS** -- you would need to keep Cloud SQL and replace Filestore with GCS or bring your own storage.
5. **No provider matches Cloud Run's integrated experience** (GPU + NFS + GCS FUSE + Cloud SQL sidecar + scale-to-zero + OIDC auth in one deploy command).

---

## Provider Comparison Table: Full-Stack Alternatives

These would replace ALL of GCP.

| Provider | GPU Option | Container Hosting | Managed Postgres | Object Storage | NFS Equivalent | Egress (per GB) |
|---|---|---|---|---|---|---|
| **GCP** (current) | L4 @ $0.67/hr | Cloud Run (scale-to-zero) | Cloud SQL ($9.37/mo micro) | GCS ($0.02/GB/mo) | Filestore ($190/mo min) | $0.12 |
| **AWS** | G6 (L4) @ $1.32/hr | ECS Fargate (scale-to-zero) | RDS ($15/mo t4g.micro) | S3 ($0.023/GB/mo) | EFS ($0.30/GB/mo) | $0.09 |
| **Azure** | T4 @ $0.32/sec (~$1.15/hr) in Container Apps | Container Apps (scale-to-zero) | Azure DB for PG ($15/mo burstable) | Blob Storage ($0.018/GB/mo) | Azure Files ($0.06/GB/mo) | $0.087 |
| **Hetzner** | RTX 4000 SFF Ada ($212/mo dedicated) | Docker on VPS (no serverless) | Self-managed on VPS | S3-compatible ($0.005/GB/mo) | No managed NFS | 1TB free, then $1/TB |
| **DigitalOcean** | Paperspace A4000 (~$0.45/hr) | App Platform (scale-to-zero) | Managed PG ($15/mo basic) | Spaces ($5/mo 250GB) | No managed NFS | 1TB free per droplet |
| **Vultr** | L40S @ $1.67/hr | No serverless containers | Managed PG ($15/mo) | Object Storage ($5/mo 250GB) | Block storage | 1TB free |
| **Fly.io** | Deprecated | Fly Machines (scale-to-zero) | Managed PG (via Supabase) | Tigris S3 | Volumes (not shared) | Free (reasonable use) |

### Full-Stack Cost Projection (20 GPU-hours/mo)

| Provider | GPU Compute | Database | Object Storage | File/Model Storage | Container Hosting | Total/mo |
|---|---|---|---|---|---|---|
| **GCP (current)** | $13.40 | $9.37 | $0.50 | $190 (Filestore) | $0 (Cloud Run free tier) | **~$213** |
| **GCP (optimized)** | $13.40 | $9.37 | $0.50 | $0.14 (GCS only, no Filestore) | $0 | **~$23** |
| **RunPod + GCP lite** | $7.80 | $9.37 | $0.50 | $3.50 (RunPod NV 50GB) | $0 | **~$21** |
| **AWS** | $26.40 | $15 | $0.50 | $3.00 (EFS 10GB) | ~$5 (Fargate) | **~$50** |
| **Hetzner + RunPod** | $7.80 (RunPod) | $5 (VPS self-managed) | $0.05 | $3.50 (RunPod NV) | $5 (VPS) | **~$22** |

**Critical insight**: The single largest cost item is **Filestore NFS at $190/mo** (1TB minimum provisioning). Eliminating Filestore by caching models in ephemeral storage (already partially done) and using GCS FUSE alone would drop the GCP bill from ~$213 to ~$23/mo. This optimization should be explored *before* any provider migration.

---

## Detailed Cost Analysis

### Current GCP Spend Breakdown

| Item | Unit Cost | Usage | Monthly |
|---|---|---|---|
| Cloud Run GPU (L4) | $0.67/hr | ~20 hrs/mo | $13.40 |
| Cloud Run CPU (4 vCPU) | included in GPU pricing | -- | $0 |
| Cloud Run Memory (16GB) | included in GPU pricing | -- | $0 |
| Cloud SQL db-f1-micro | $9.37/mo flat | Always on | $9.37 |
| Cloud SQL storage (10GB) | included in micro | -- | $0 |
| GCS Standard (io-cache) | $0.02/GB/mo | ~10GB avg | $0.20 |
| GCS Standard (models) | $0.02/GB/mo | ~7GB | $0.14 |
| GCS Operations | $0.005/1K ops | ~5K ops/mo | $0.025 |
| Filestore Basic HDD | $0.19/GiB/mo, 1TB min | 1024 GiB | **$194.56** |
| Container Registry (GCR) | $0.026/GB/mo | ~15GB | $0.39 |
| Egress | $0.12/GB | ~1GB/mo | $0.12 |
| **Total** | | | **~$218** |

### RunPod GPU Worker Equivalent

| Item | Unit Cost | Usage | Monthly |
|---|---|---|---|
| RunPod Pod (L4 24GB) | $0.39/hr | ~20 hrs/mo | $7.80 |
| RunPod Container Disk | $0.10/GB/mo | 50GB | $5.00 |
| RunPod Network Volume | $0.07/GB/mo | 50GB (models) | $3.50 |
| RunPod Egress | $0 (free) | -- | $0 |
| **GPU subtotal** | | | **$16.30** |

Keeping GCP for database + io-cache:
| Cloud SQL micro | $9.37/mo | -- | $9.37 |
| GCS io-cache | $0.02/GB/mo | 10GB | $0.20 |
| **Total (RunPod + GCP lite)** | | | **~$26** |

### Modal Serverless Equivalent

| Item | Unit Cost | Usage | Monthly |
|---|---|---|---|
| Modal L4 compute | $0.80/hr ($0.000222/sec) | ~20 hrs/mo | $16.00 |
| Modal CPU (4 cores) | $0.047/hr | ~20 hrs/mo | $0.94 |
| Modal Memory (16GB) | $0.032/hr | ~20 hrs/mo | $0.64 |
| Modal Volume (models) | ~$0.10/GB/mo (est.) | 50GB | $5.00 |
| Modal free credits | -$30/mo (Starter) | | -$30.00 |
| **GPU subtotal** | | | **~$0** (covered by credits at low usage) |

At low usage (20 hrs/mo), Modal would be essentially free with Starter plan credits. At higher usage (100+ hrs/mo), the $0.80/hr rate becomes significant.

### AWS Full-Stack Equivalent

| Item | Unit Cost | Usage | Monthly |
|---|---|---|---|
| EC2 g6.xlarge (1x L4) | $0.98/hr (us-east-1) | ~20 hrs/mo | $19.60 |
| RDS PostgreSQL (db.t4g.micro) | ~$15/mo | Always on | $15.00 |
| S3 Standard | $0.023/GB/mo | 20GB | $0.46 |
| EFS (Elastic File System) | $0.30/GB/mo | 10GB | $3.00 |
| ECR (Container Registry) | $0.10/GB/mo | 15GB | $1.50 |
| ECS Fargate (API, if needed) | ~$0.04/vCPU-hr | minimal | ~$5.00 |
| Egress | $0.09/GB | 5GB | $0.45 |
| **Total** | | | **~$45** |

---

## Top 3 Migration Plans

---

### Plan A: GPU Workload to RunPod (Recommended)

**Scope**: Replace Cloud Run GPU worker with RunPod Pod. Keep GCP for everything else. Optionally eliminate Filestore.

#### What Changes in Infrastructure

| Component | Before (GCP) | After (RunPod + GCP) |
|---|---|---|
| GPU container | Cloud Run GPU service | RunPod Pod (L4, 12 vCPU, 50GB RAM) |
| Model storage | Filestore NFS at /models/comfyui | RunPod Network Volume at /models/comfyui |
| IO cache | GCS bucket + FUSE mount | RunPod container disk + API push/pull via HTTP |
| Database | Cloud SQL PostgreSQL | Cloud SQL PostgreSQL (unchanged) |
| Container registry | GCR (gcr.io/neovnext/...) | Docker Hub or RunPod registry |
| Auth (OIDC) | Google Identity tokens | Bearer token (already supported via WORKER_TOKEN) |
| Scale management | Cloud Run min/max instances | RunPod API: start/stop pod |

#### What Changes in the Codebase

**Deploy scripts** (high impact):
- Replace `gcloud run deploy` with RunPod API calls or `runpodctl` CLI
- Replace `docker push gcr.io/...` with `docker push` to Docker Hub or RunPod registry
- Replace `gcloud run services update` (scaling) with RunPod pod start/stop API

**C# API** (medium impact):
- `GoogleCloudGpuPlatformAdapter.cs` -- replace with `RunPodGpuPlatformAdapter.cs`
  - Replace OIDC token generation with simple bearer token auth
  - Replace `gcloud run services update` (warm/scale) with RunPod API `POST /pods/{id}/start` and `POST /pods/{id}/stop`
  - Replace Cloud Run service URL resolution with RunPod pod URL
- `ArtifactSyncService.cs` -- replace GCS download with HTTP download from RunPod pod's exposed port or S3-compatible storage
- `InputStagingService.cs` -- replace GCS upload with HTTP upload to RunPod pod
- `CacheEvictionService.cs` -- remove GCS eviction logic (or adapt to new storage)
- Remove `Google.Cloud.Storage.V1` NuGet package (if Filestore also removed)
- Keep `NpgsqlConnectionFactory.cs` (Cloud SQL stays)

**Go supervisor** (low impact):
- Remove `CLOUD_RUN_SERVICE_URL` keepalive logic (RunPod handles pod lifecycle differently)
- Database connection string changes from Cloud SQL socket to direct TCP (RunPod can reach Cloud SQL via public IP + SSL)
- Everything else (handler management, job polling, heartbeats) stays the same

**Dockerfile** (no change):
- `worker-gpu-go.Dockerfile` is generic CUDA -- works as-is on RunPod

#### Data Migration

1. **Models**: Copy from Filestore/GCS to RunPod Network Volume (`runpodctl send` or `rsync`)
2. **Database**: No migration needed (stays on Cloud SQL)
3. **IO cache**: No migration needed (transient data)

#### Risk Assessment

| Risk | Severity | Mitigation |
|---|---|---|
| RunPod pod networking to Cloud SQL | Medium | Use Cloud SQL public IP + SSL; test latency |
| No scale-to-zero (Pods stay running when started) | Medium | Implement stop-after-idle via RunPod API |
| RunPod availability/SLA | Low | RunPod Secure Cloud has 99.9% SLA |
| Network latency (RunPod <-> GCS) | Medium | Replace GCS with RunPod storage; API pushes artifacts via HTTP |

#### Rollback Strategy

Keep Cloud Run service deployed but paused (max-instances=0). Can resume with single `gcloud run services update` command.

#### Estimated Migration Effort: 3-5 days

---

### Plan B: GPU Workload to Modal

**Scope**: Replace Cloud Run GPU worker with Modal serverless functions. Keep GCP for database.

#### What Changes in Infrastructure

| Component | Before (GCP) | After (Modal + GCP) |
|---|---|---|
| GPU container | Cloud Run GPU service | Modal Function with L4 GPU |
| Architecture | Go supervisor + Python handlers | Python-native Modal functions |
| Model storage | Filestore NFS | Modal Volume (persistent) |
| IO cache | GCS bucket + FUSE mount | Modal Volume or S3-compatible |
| Database | Cloud SQL PostgreSQL | Cloud SQL PostgreSQL (unchanged) |

#### What Changes in the Codebase

**This is a significant rearchitecture.** Modal uses a Python-native, decorator-based approach:

```python
# Example: how a handler would look on Modal
import modal

app = modal.App("neovnext-gpu")
image = modal.Image.from_dockerfile("docker/worker-gpu-go.Dockerfile")

@app.function(gpu="L4", image=image, timeout=2100)
def run_handler(job_type: str, job_id: str, inputs: dict):
    # Handler logic here
    ...
```

**Major changes:**
- Go supervisor would be eliminated or significantly reduced (Modal handles scheduling)
- Python handlers would be wrapped in Modal function decorators
- C# API would call Modal via HTTP webhook or Modal's Python client
- All deploy scripts replaced with `modal deploy`
- ComfyUI integration would need rethinking (Modal has no persistent background process model)

**This is the highest-effort migration option** but yields the best long-term developer experience for pure Python ML workloads.

#### Risk Assessment

| Risk | Severity | Mitigation |
|---|---|---|
| ComfyUI incompatibility (needs long-running server) | High | Use Modal's `modal.Cls` for persistent containers |
| Loss of Go supervisor benefits | Medium | Modal's scheduler may be superior for this use case |
| Vendor lock-in to Modal's SDK | High | Abstract handler logic; keep Modal as thin wrapper |
| Cold start for large models | Medium | Modal Volumes + snapshot caching |

#### Rollback Strategy

Keep Cloud Run service paused. Modal has no infrastructure to tear down (serverless).

#### Estimated Migration Effort: 8-15 days

---

### Plan C: Full Migration to AWS

**Scope**: Replace all GCP services with AWS equivalents.

#### What Changes in Infrastructure

| Component | GCP (current) | AWS (target) |
|---|---|---|
| GPU worker | Cloud Run GPU (L4) | ECS on EC2 G6 (L4) or SageMaker Inference |
| Database | Cloud SQL PostgreSQL | RDS PostgreSQL (db.t4g.micro) |
| Object storage | GCS buckets | S3 buckets |
| File storage | Filestore NFS | EFS (Elastic File System) |
| Container registry | GCR | ECR (Elastic Container Registry) |
| Networking | GCP VPC | AWS VPC |
| IAM | GCP Service Accounts | AWS IAM Roles |
| Logging | Cloud Logging | CloudWatch Logs |
| Container hosting (API) | N/A (local) | ECS Fargate (if needed later) |

#### What Changes in the Codebase

**C# API** (high impact):
- Replace `Google.Cloud.Storage.V1` with `AWSSDK.S3`
- Replace `Google.Apis.Auth` with AWS SDK auth
- New `AwsGpuPlatformAdapter.cs`:
  - ECS task management instead of Cloud Run scaling
  - AWS IAM auth instead of OIDC tokens
- `NpgsqlConnectionFactory.cs`: standard TCP connection string to RDS
- All GCS config keys become S3 config keys

**Go supervisor** (medium impact):
- Database URL becomes standard `postgresql://host:5432/db` (no `/cloudsql/` prefix)
- IO cache mount path stays the same (just backed by EFS or S3 FUSE instead of GCS FUSE)
- Remove Cloud Run keepalive (ECS handles task lifecycle)

**Deploy scripts** (complete rewrite):
- Replace all `gcloud` commands with `aws` CLI equivalents
- Replace `docker push gcr.io/...` with `docker push {account}.dkr.ecr.{region}.amazonaws.com/...`
- New ECS task definition, service, and cluster configuration
- Terraform or CloudFormation recommended

**Dockerfile** (no change):
- Generic CUDA base image works on any provider

#### Data Migration

1. **Database**: `pg_dump` from Cloud SQL, `pg_restore` to RDS
2. **GCS to S3**: `gsutil` or `rclone` to copy bucket contents
3. **Filestore to EFS**: `rsync` from Filestore mount to EFS mount
4. **Container images**: Pull from GCR, push to ECR

#### Risk Assessment

| Risk | Severity | Mitigation |
|---|---|---|
| ECS lacks Cloud Run's GPU-aware scale-to-zero | High | Use ECS auto-scaling with custom metrics; accept idle costs |
| Migration window (both systems running) | Medium | Run parallel for 1 week; gradual cutover |
| AWS complexity (IAM, VPC, Security Groups) | Medium | Use Terraform modules; follow AWS well-architected |
| Higher GPU cost ($1.32/hr vs $0.67/hr) | Medium | Use Spot instances ($0.40-0.60/hr) or Savings Plans |

#### Rollback Strategy

Keep GCP resources running during migration window. DNS/config switch to revert.

#### Estimated Migration Effort: 15-25 days

---

## Command Reference: GCP to Alternative Mappings

### Deploy Commands

| Operation | GCP (current) | RunPod (Plan A) | AWS (Plan C) |
|---|---|---|---|
| **Build + push image** | `docker build -t gcr.io/proj/img:tag .`<br>`docker push gcr.io/proj/img:tag` | `docker build -t dockerhub/img:tag .`<br>`docker push dockerhub/img:tag` | `docker build -t acct.dkr.ecr.region.amazonaws.com/img:tag .`<br>`aws ecr get-login-password \| docker login ...`<br>`docker push acct.dkr.ecr.region.amazonaws.com/img:tag` |
| **Deploy service** | `gcloud run deploy svc --image img --gpu 1 --gpu-type nvidia-l4 ...` | `runpodctl create pod --gpu-type L4 --image img --volume-id vol_xxx ...` or RunPod API `POST /pods` | `aws ecs update-service --cluster c --service s --task-definition td:rev` |
| **Scale to zero** | `gcloud run services update svc --max-instances 0` | RunPod API `POST /pods/{id}/stop` | `aws ecs update-service --desired-count 0` |
| **Resume** | `gcloud run services update svc --max-instances 1` | RunPod API `POST /pods/{id}/start` | `aws ecs update-service --desired-count 1` |
| **View logs** | `gcloud logging read "resource.labels.service_name=svc"` | RunPod dashboard or API `GET /pods/{id}/logs` | `aws logs get-log-events --log-group /ecs/svc` |
| **Rollback** | `gcloud run services update-traffic svc --to-revisions=rev=100` | Stop pod, create new with old image | `aws ecs update-service --task-definition td:prev-rev` |

### Database Commands

| Operation | GCP Cloud SQL | RunPod (external) | AWS RDS |
|---|---|---|---|
| **Create instance** | `gcloud sql instances create name --tier=db-f1-micro --region=us-east4` | N/A (use GCP Cloud SQL or Neon/Supabase) | `aws rds create-db-instance --db-instance-identifier name --db-instance-class db.t4g.micro --engine postgres` |
| **Connect** | `gcloud sql connect name --database=db --user=user` | `psql postgresql://user:pass@host:5432/db` | `psql postgresql://user:pass@name.region.rds.amazonaws.com:5432/db` |
| **Import SQL** | `gcloud sql import sql name gs://bucket/file.sql --database=db` | `psql -f file.sql` | `psql -h endpoint -f file.sql` |
| **Backup** | `gcloud sql backups create --instance=name` | N/A | `aws rds create-db-snapshot --db-instance-identifier name` |

### Storage Commands

| Operation | GCP (gsutil/gcloud storage) | RunPod | AWS (aws s3) |
|---|---|---|---|
| **Create bucket** | `gsutil mb -l us-east4 gs://bucket` | N/A (use Network Volume) | `aws s3 mb s3://bucket --region us-east-1` |
| **Upload file** | `gsutil cp file gs://bucket/path` | `runpodctl send file` or HTTP upload | `aws s3 cp file s3://bucket/path` |
| **Download file** | `gsutil cp gs://bucket/path file` | `runpodctl receive` or HTTP download | `aws s3 cp s3://bucket/path file` |
| **List contents** | `gsutil ls gs://bucket/` | RunPod dashboard | `aws s3 ls s3://bucket/` |
| **Delete bucket** | `gsutil rm -r gs://bucket` | Delete Network Volume in dashboard | `aws s3 rb s3://bucket --force` |

### C# SDK Replacements

| GCP SDK | Purpose | RunPod Replacement | AWS Replacement |
|---|---|---|---|
| `Google.Cloud.Storage.V1` (StorageClient) | GCS object upload/download | `HttpClient` to RunPod storage API or S3-compatible endpoint | `AWSSDK.S3` (AmazonS3Client) |
| `Google.Apis.Auth` (GoogleCredential) | OIDC identity tokens for Cloud Run auth | Simple bearer token (WORKER_TOKEN) | `AWSSDK.Core` (AWS SigV4 auth) |
| `gcloud` CLI (shelled out) | Cloud Run scaling, auth tokens | RunPod REST API via HttpClient | `aws` CLI or AWS SDK |

---

## Recommendation

### Immediate (This Sprint): Eliminate Filestore

Before any provider migration, **eliminate the Filestore NFS dependency**. This alone saves ~$190/mo:

1. The worker already caches primary models in ephemeral storage (14GB of 32GB free)
2. Extend caching to cover all models needed for active workflows
3. Use GCS as cold storage for models (already have the bucket)
4. Remove Filestore mount from Cloud Run deploy config

### Short-Term (Next 2 Sprints): GPU to RunPod (Plan A)

1. Create `RunPodGpuPlatformAdapter.cs` implementing `IGpuPlatformAdapter`
2. Set up RunPod Pod with L4 GPU + Network Volume for models
3. Modify artifact sync to use HTTP push/pull instead of GCS FUSE
4. Update deploy scripts to use RunPod API
5. Keep Cloud SQL on GCP (direct TCP connection from RunPod with SSL)

**Expected savings**: ~$6/mo on GPU compute (small at 20 hrs/mo), but better scaling economics at higher usage. The real win is eliminating Filestore ($190/mo).

### Medium-Term (Evaluate After 3 Months on RunPod)

- If GPU usage grows significantly, evaluate Modal for the Python-native developer experience
- If non-GPU GCP costs grow, evaluate migrating Cloud SQL to Neon (serverless Postgres, potentially free tier)
- If egress becomes a cost factor, evaluate moving everything to a single provider

### What NOT to Do

- **Do not migrate to AWS** unless you anticipate enterprise-scale needs. The complexity overhead is not justified for a desktop app's cloud GPU tier.
- **Do not use Vast.ai** for production. The marketplace model introduces reliability risks unacceptable for user-facing features.
- **Do not migrate to Hetzner** for GPU. Their GPU offerings are datacenter-class (dedicated monthly), not the bursty on-demand model NeoVNext needs.
- **Do not use Fly.io GPUs**. They deprecated their GPU offering in 2025.

---

## Appendix: Sources

- [RunPod Pricing](https://www.runpod.io/pricing)
- [RunPod L4 GPU](https://www.runpod.io/gpu-models/l4)
- [Modal Pricing](https://modal.com/pricing)
- [Modal L4 GPU Cost Analysis](https://modal.com/blog/nvidia-l4-price-article)
- [CoreWeave Pricing](https://www.coreweave.com/pricing)
- [Lambda AI Pricing](https://lambda.ai/pricing)
- [Vast.ai L4 GPU](https://vast.ai/pricing/gpu/L4)
- [Fly.io Pricing](https://fly.io/docs/about/pricing/)
- [GCP Cloud Run Pricing](https://cloud.google.com/run/pricing)
- [GCP Cloud SQL Pricing](https://cloud.google.com/sql/pricing)
- [GCP Filestore Pricing](https://cloud.google.com/filestore/pricing)
- [GCP GPU Pricing](https://cloud.google.com/compute/gpus-pricing)
- [AWS EC2 G6 Instances](https://aws.amazon.com/ec2/instance-types/g6/)
- [Azure Container Apps GPU](https://learn.microsoft.com/en-us/azure/container-apps/gpu-serverless-overview)
- [Hetzner GPU Servers](https://www.hetzner.com/dedicated-rootserver/gex44/)
- [L4 Cloud Pricing Comparison (GetDeploying)](https://getdeploying.com/gpus/nvidia-l4)
- [Cloud GPU Providers Comparison 2026](https://www.gpu.fm/blog/cloud-gpu-providers-comparison-2026)
- [Cloud Egress Costs Comparison](https://holori.com/egress-costs-comparison/)
- [Paperspace/DigitalOcean Pricing](https://docs.digitalocean.com/products/paperspace/pricing/)
- [PostgreSQL Hosting Pricing Comparison](https://www.bytebase.com/blog/postgres-hosting-options-pricing-comparison/)
