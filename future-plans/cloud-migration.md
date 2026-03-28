# Cloud Migration Plan

> **Status**: Approved plan, pending execution
> **Last Updated**: 2026-03-27
> **Decision Owner**: Steve
> **Full Research**: [`docs/cloud-migration-research.md`](../cloud-migration-research.md)

---

## The Problem

NeoVNext's GCP bill is approximately **$218/month** at light usage (roughly 20 GPU-hours/month). The cost breakdown reveals a single dominant line item:

| Item | Monthly Cost | % of Total |
|---|---|---|
| **Filestore NFS** (1TB minimum) | **$190** | **87%** |
| Cloud SQL PostgreSQL (db-f1-micro) | $9.37 | 4% |
| GPU compute (Cloud Run L4, 20 hrs) | $13.40 | 6% |
| Storage, egress, misc | $5 | 3% |

Filestore NFS exists solely to mount ComfyUI model checkpoints (~7GB actual usage) into the Cloud Run GPU worker. We are paying $190/month for a 1TB minimum allocation to store 7GB of files. This is the single highest-leverage optimization available.

---

## Recommended Execution Sequence

### Phase 1: Kill Filestore NFS (saves ~$190/month)

**Effort**: 1-2 days | **Risk**: Low | **Savings**: ~$190/month

The GPU worker already caches primary models in ephemeral container storage. The path forward:

1. Extend ephemeral caching to cover all models needed for active workflows
2. Use the existing GCS models bucket as cold storage (already contains the checkpoints at ~$0.14/month)
3. On worker startup, pull any missing models from GCS into local ephemeral storage
4. Remove Filestore NFS mount from Cloud Run deploy configuration
5. Delete the Filestore instance

**After this phase, the monthly GCP bill drops from ~$218 to ~$23.**

This is the move that matters. Everything after this is optimization at the margins.

### Phase 2: GPU Compute to RunPod (saves ~$6/month at current usage)

**Effort**: 3-5 days | **Risk**: Medium | **Savings**: ~$6/month now, scales better at higher usage

Replace Cloud Run GPU with a RunPod Pod running the same Docker image (no Dockerfile changes needed).

| What Changes | From | To |
|---|---|---|
| GPU hosting | Cloud Run (L4 @ $0.67/hr) | RunPod Pod (L4 @ $0.39/hr) |
| Model storage | GCS cold + ephemeral cache | RunPod Network Volume ($3.50/mo for 50GB) |
| IO artifacts | GCS FUSE mount | HTTP push/pull to RunPod pod |
| Auth | Google OIDC tokens | Bearer token (WORKER_TOKEN, already supported) |
| Scale management | Cloud Run auto-scaling | RunPod API start/stop |
| Database | Cloud SQL (unchanged) | Cloud SQL via public IP + SSL |

**Codebase changes required:**
- New `RunPodGpuPlatformAdapter.cs` (replacing `GoogleCloudGpuPlatformAdapter.cs`)
- Modify `ArtifactSyncService.cs` and `InputStagingService.cs` for HTTP-based artifact transfer
- Update deploy scripts from `gcloud` to RunPod API / `runpodctl`
- Go supervisor: minor config changes (TCP database connection instead of Cloud SQL socket)
- Dockerfile: no changes

**Rollback**: Keep Cloud Run service deployed at max-instances=0. Resume with a single `gcloud` command.

At 20 GPU-hours/month the savings are modest ($6). The real value is better scaling economics: at 100 GPU-hours/month, savings reach ~$28/month, and RunPod's free egress eliminates a variable cost.

### Phase 3: Evaluate Further Optimization (3+ months out)

After running on RunPod for at least one billing cycle, evaluate:

- **If GPU usage grows significantly**: Consider Modal for Python-native serverless GPU with sub-second cold starts. Requires rearchitecting the Go supervisor (8-15 days effort, high risk). Only justified if the developer experience gains outweigh the migration cost.
- **If Cloud SQL costs become material**: Evaluate Neon (serverless Postgres with a free tier) as a Cloud SQL replacement.
- **If egress costs grow**: Consider consolidating all services onto a single provider.

---

## Top 3 Migration Plans at a Glance

| | Plan A: RunPod Hybrid | Plan B: Modal Serverless | Plan C: Full AWS |
|---|---|---|---|
| **Scope** | GPU to RunPod, keep GCP for DB/storage | GPU to Modal, keep GCP for DB | Replace all GCP services |
| **Effort** | 3-5 days | 8-15 days | 15-25 days |
| **Monthly cost** (20 GPU-hrs) | ~$21 | ~$0 (free tier credits) | ~$45 |
| **Monthly cost** (100 GPU-hrs) | ~$55 | ~$82 | ~$135 |
| **Scale-to-zero** | Manual (API stop/start) | Native (automatic) | Manual (ECS desired-count) |
| **Dockerfile changes** | None | N/A (Python decorators) | None |
| **Go supervisor** | Kept (minor config changes) | Eliminated (Modal scheduler) | Kept (minor config changes) |
| **Risk profile** | Low-medium | High (rearchitecture) | Medium (complexity) |
| **Vendor lock-in** | Low (standard Docker) | High (Modal SDK) | Medium (AWS SDK) |
| **Best for** | Current architecture, cost savings now | Python-heavy future, low volume | Enterprise scale, single vendor |

**Recommended**: Plan A (RunPod Hybrid). Lowest effort, lowest risk, preserves the existing Go supervisor architecture, and captures GPU cost savings immediately. Plan B becomes interesting only if GPU usage grows enough to justify the rearchitecture.

---

## What NOT to Do

These options were evaluated and explicitly rejected. If revisiting, check the full research doc for reasoning.

- **Do not migrate to AWS** unless enterprise-scale needs emerge. The complexity overhead is not justified for a desktop app's cloud GPU tier.
- **Do not use Vast.ai** for production. Marketplace reliability is unacceptable for user-facing features.
- **Do not use Hetzner for GPU**. Dedicated monthly pricing does not fit NeoVNext's bursty workload pattern.
- **Do not use Fly.io GPUs**. They deprecated GPU support in August 2025.

---

## Decision Record

| Date | Decision | Rationale |
|---|---|---|
| 2026-03-27 | Kill Filestore NFS first | 87% of monthly cost for <1% of storage capacity used |
| 2026-03-27 | RunPod as GPU target (Plan A) | Lowest effort (3-5 days), lowest risk, 42% GPU cost reduction |
| 2026-03-27 | Keep Cloud SQL on GCP | $9.37/month is reasonable; no migration overhead justified |
| 2026-03-27 | Defer Modal evaluation | High rearchitecture cost not justified at current GPU usage levels |

---

## References

- **Full research document**: [`docs/cloud-migration-research.md`](../cloud-migration-research.md) -- contains provider comparison tables, detailed cost breakdowns, codebase impact analysis, SDK replacement mappings, and deploy command references.
- [RunPod Pricing](https://www.runpod.io/pricing)
- [Modal Pricing](https://modal.com/pricing)
- [GCP Filestore Pricing](https://cloud.google.com/filestore/pricing)
- [GCP Cloud Run GPU Pricing](https://cloud.google.com/run/pricing)
