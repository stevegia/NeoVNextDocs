# CONTINGENCY PLAN: Alternative GPU Cloud Providers

**Status: CONTINGENCY PLAN ONLY -- This is NOT the current infrastructure.**

The current production system runs on Google Cloud Run with an NVIDIA L4 GPU in us-east4, funded by GCloud credits. This document researches alternatives for when those credits expire.

**Last updated:** 2026-03-26

## Current Setup Summary

| Component | Current | Cost |
|---|---|---|
| GPU Compute | Cloud Run, 1x L4 (24GB), 4 vCPU, 16GB RAM, scale-to-zero | ~$2.16/hr active, $0 idle |
| Model Storage | Cloud Filestore BASIC_HDD, 1 TiB (114GB used), NFS mount | $45/mo fixed |
| Database | Cloud SQL PostgreSQL 16, db-f1-micro | $13/mo |
| Object Storage | GCS bucket for I/O cache, GCS FUSE mount | ~$3/mo |
| Idle baseline | Filestore + Cloud SQL + Storage | ~$61/mo |
| Light usage (4 hrs/day) | All services | ~$320/mo |

**Key architectural constraints:**
- Single Docker container (~8.5GB image) running Go supervisor + Python ML subprocesses + ComfyUI
- NFS-mounted model storage at `/models/comfyui` (~114GB: 91GB Wan diffusion model, 21GB checkpoints, 7GB CLIP/text encoders)
- PostgreSQL access required for job queue (desktop API writes jobs, GPU worker polls)
- GCS FUSE mount at `/data/io-cache` for job I/O
- Scale-to-zero is critical for cost management
- GPU mode switching: shared mode (~12GB VRAM) for SD/SDXL + analysis handlers, exclusive mode (~18GB VRAM) for Wan 14B

---

## Provider Comparison

### 1. RunPod

**Overview:** GPU cloud with both "Pods" (persistent instances) and "Serverless" (scale-to-zero endpoints).

| Aspect | Details |
|---|---|
| GPU Options | A40 (48GB) ~$0.39/hr, A100 80GB ~$1.64/hr, L40S (48GB) ~$0.74/hr, L4 ~$0.24/hr, H100 ~$2.49/hr. Community cloud cheaper. |
| Custom Docker | Yes, full Docker support on Pods. Serverless requires their handler format. |
| Persistent Storage | Network volumes (NFS-like), $0.07/GB/mo for HDD, $0.10/GB/mo for SSD. 114GB = ~$8-11/mo. |
| Network/VPC | No VPC peering. Public internet only. PostgreSQL must be internet-accessible or tunneled. |
| Scale-to-Zero | Serverless: yes. Pods: no (but can be stopped manually). |
| Cold Start | Serverless: 10-60s depending on image size. Pods: minutes to start from stopped. |
| Architecture Changes | **Pods: minimal** -- run our Docker image as-is, point DATABASE_URL to public PG, replace NFS with network volume. **Serverless: major** -- must refactor to their handler API. |

**Verdict:** Pods are a near-drop-in replacement. The A40 at $0.39/hr gives 48GB VRAM (no more mode switching) for less than Cloud Run L4 at $2.16/hr. However, no scale-to-zero on Pods means paying for idle time.

### 2. Lambda Labs

**Overview:** Simple GPU cloud instances, no serverless option.

| Aspect | Details |
|---|---|
| GPU Options | A100 (40GB/80GB), H100, A10 (24GB). Limited L4/L40S availability. |
| Pricing | A10 ~$0.75/hr, A100 40GB ~$1.29/hr, A100 80GB ~$1.99/hr, H100 ~$2.49/hr. |
| Custom Docker | Yes, full root SSH access, run whatever you want. |
| Persistent Storage | Persistent filesystems, up to 10TB. Pricing ~$0.10/GB/mo. |
| Network/VPC | No VPC. Public internet. |
| Scale-to-Zero | No. Instances are billed while running. Can terminate and re-create but lose local state. |
| Cold Start | Instance launch: 1-5 minutes. |
| Architecture Changes | Minimal -- SSH in, docker run our image. Need to handle model persistence across instance restarts. |

**Verdict:** Clean and simple but no scale-to-zero. Only makes sense for sustained usage. Availability of specific GPU types can be limited.

### 3. Vast.ai

**Overview:** GPU marketplace (both on-demand and spot/interruptible). Cheapest option, least reliable.

| Aspect | Details |
|---|---|
| GPU Options | Everything: RTX 4090 (24GB), A40 (48GB), A100, L40S, H100. Wide selection. |
| Pricing | RTX 4090 ~$0.15-0.25/hr, A40 ~$0.20-0.35/hr, A100 80GB ~$0.80-1.20/hr. Spot pricing fluctuates. |
| Custom Docker | Yes, full Docker support. |
| Persistent Storage | Disk attached to instance. Lost when instance is destroyed unless using their "template" volumes. |
| Network/VPC | No VPC. Public internet only. Machines are in various datacenters globally. |
| Scale-to-Zero | No. Spot instances can be interrupted. On-demand runs until stopped. |
| Cold Start | Instance creation: 2-10 minutes. Docker pull can add 5-10 min for our 8.5GB image. |
| Architecture Changes | Moderate -- need a script to provision instances on demand, download models on startup (or use a persistent disk), expose PostgreSQL publicly. Reliability is lower. |

**Verdict:** Extremely cheap for development and testing. An A40 (48GB) for $0.25/hr eliminates mode switching entirely. Not recommended for production without a management layer due to reliability concerns.

### 4. CoreWeave

**Overview:** Kubernetes-native GPU cloud. Enterprise-focused.

| Aspect | Details |
|---|---|
| GPU Options | A40, A100, L40S, H100, L4. Broad selection. |
| Pricing | A40 ~$0.74/hr, L40S ~$0.99/hr, A100 80GB ~$2.21/hr. Committed pricing lower. |
| Custom Docker | Yes, via Kubernetes pods. |
| Persistent Storage | Block and shared filesystems. NFS-compatible volumes available. |
| Network/VPC | Full VPC, private networking, VPN connectivity. Best enterprise networking. |
| Scale-to-Zero | Via Kubernetes HPA or KEDA. Requires setup. |
| Cold Start | Pod scheduling + image pull: 1-5 minutes depending on node availability. |
| Architecture Changes | Moderate -- need to write Kubernetes manifests (Deployment, PVC, Service). Our Docker image runs as-is inside the pod. |

**Verdict:** Best networking and reliability, but requires Kubernetes expertise and has minimum spend requirements. Overkill for a single-worker setup.

### 5. Modal

**Overview:** Serverless GPU compute with Python-first API. Code runs in their managed containers.

| Aspect | Details |
|---|---|
| GPU Options | T4, A10G, A100, L4, L40S, H100. |
| Pricing | A10G ~$1.10/hr, A100 ~$3.72/hr, L4 ~$0.59/hr. Includes CPU/RAM. |
| Custom Docker | Partial -- you define a Modal "Image" with install steps but cannot run arbitrary Docker images directly. |
| Persistent Storage | Modal Volumes (network storage), $0.63/GB/mo. 114GB = ~$72/mo (expensive). |
| Network/VPC | No VPC. Can make outbound connections. No inbound except via Modal endpoints. |
| Scale-to-Zero | Yes, native. This is their core value prop. |
| Cold Start | ~10-30s with warm containers, up to 2 minutes cold. |
| Architecture Changes | **Major refactor.** Must convert our Go supervisor + Python subprocesses into Modal's Python decorator pattern. ComfyUI would need to be wrapped as a Modal function. |

**Verdict:** Excellent developer experience and true scale-to-zero, but requires a fundamental rewrite of the supervisor architecture. Storage is expensive. Not a good fit for our existing Docker-based approach.

### 6. Banana.dev / Replicate

| Aspect | Details |
|---|---|
| **Banana.dev** | Effectively dead/pivoted. Not recommended. |
| **Replicate** | Run models via API. Can create custom "Cog" containers. A40 ~$0.65/hr, A100 ~$1.15/hr. Scale-to-zero supported. |
| Custom Docker | Replicate: must use their Cog container format. Not a standard Docker. |
| Persistent Storage | No persistent volumes. Models must be baked into the image or downloaded at startup. |
| Architecture Changes | **Major refactor.** Must split each handler into a separate Cog model. |

**Verdict:** Wrong abstraction layer. These are for serving individual models, not running a multi-handler supervisor.

### 7. Paperspace (by DigitalOcean)

| Aspect | Details |
|---|---|
| GPU Options | A4000 (16GB), A5000 (24GB), A6000 (48GB), A100. |
| Pricing | A4000 ~$0.76/hr, A5000 ~$0.87/hr, A6000 ~$1.89/hr, A100 ~$3.09/hr. |
| Custom Docker | Yes on "Core" machines (full VMs). |
| Persistent Storage | Persistent shared drives, ~$0.29/GB/mo. 114GB = ~$33/mo. |
| Scale-to-Zero | No. |
| Architecture Changes | Minimal on Core machines -- run our Docker image as-is. |

**Verdict:** Straightforward but not price-competitive. A6000 (48GB) at $1.89/hr is more expensive than RunPod's A40 at $0.39/hr for the same VRAM class.

---

## Comparison Table

| Provider | L4 (24GB) | A40 (48GB) | A100 80GB | Scale-to-Zero | Custom Docker | Storage (114GB) | VPC | Arch Change |
|---|---|---|---|---|---|---|---|---|
| **GCP Cloud Run (current)** | $2.16/hr* | N/A | N/A | Yes | Yes | $45/mo | Yes | None |
| **RunPod Pods** | $0.24/hr | $0.39/hr | $1.64/hr | No | Yes | ~$10/mo | No | Minimal |
| **RunPod Serverless** | ~$0.24/hr | N/A | N/A | Yes | Partial | N/A | No | Major |
| **Lambda Labs** | N/A | N/A | $1.29/hr | No | Yes | ~$11/mo | No | Minimal |
| **Vast.ai** | ~$0.20/hr | ~$0.28/hr | ~$0.90/hr | No | Yes | Included | No | Moderate |
| **CoreWeave** | Available | $0.74/hr | $2.21/hr | Via K8s | Yes | Available | Yes | Moderate |
| **Modal** | $0.59/hr | N/A | $3.72/hr | Yes | Partial | ~$72/mo | No | Major |
| **Replicate** | N/A | $0.65/hr | $1.15/hr | Yes | Cog only | None | No | Major |
| **Paperspace** | N/A | N/A | $3.09/hr | No | Yes | ~$33/mo | Partial | Minimal |

*GCP Cloud Run $2.16/hr includes CPU + RAM + GPU bundled pricing

---

## VRAM Upgrade Analysis: Eliminating Mode Switching

With 48GB+ VRAM, all handlers could run simultaneously with Wan 14B, eliminating GPU mode transitions.

| GPU | VRAM | Run Everything? | Cheapest Provider | Cost |
|---|---|---|---|---|
| L4 | 24GB | No -- need mode switching | RunPod | $0.24/hr |
| RTX 4090 | 24GB | No -- same constraint | Vast.ai | $0.18/hr |
| A40 | 48GB | **Yes** -- ~30GB total | RunPod | $0.39/hr |
| L40S | 48GB | **Yes** -- faster than A40 | RunPod | $0.74/hr |
| A100 80GB | 80GB | **Yes** -- plenty of room | RunPod | $1.64/hr |

**Recommendation:** An A40 (48GB) is the sweet spot. At $0.39/hr on RunPod, it costs less than our current L4 on Cloud Run ($2.16/hr) while providing double the VRAM.

---

## Recommended Migration Path

### Option A: RunPod Pods (Recommended -- Lowest Effort)

**Estimated savings: 60-80% vs current Cloud Run**

1. Make PostgreSQL accessible from the internet (Cloud SQL authorized networks + SSL)
2. Create a RunPod Network Volume (~120GB) and upload models
3. Deploy our existing Docker image as a RunPod Pod with an A40 GPU
4. Modify `DATABASE_URL` to point to Cloud SQL public IP
5. Replace GCS FUSE I/O cache with local disk or RunPod S3-compatible storage
6. Update the API's `IGpuPlatformAdapter` to manage RunPod pods via their API
7. Remove GPU mode switching code (no longer needed with 48GB VRAM)

**Cost estimate (4 hrs/day):** ~$70/mo (vs ~$320/mo current)

### Option B: Vast.ai (Lowest Cost, More Work)

**Estimated savings: 80-90% vs current Cloud Run**

Same DB exposure, but need instance orchestration and model download scripts.

**Cost estimate (4 hrs/day):** ~$50/mo

### Option C: RunPod Serverless (Best Scale-to-Zero, Most Refactoring)

Only worth considering if usage becomes very sporadic (< 1 hr/day).

---

## Migration Priority

1. **Immediate (before credits expire):** Expose Cloud SQL via public IP with SSL + authorized networks
2. **First migration:** RunPod Pods (Option A)
3. **If insufficient:** Fall back to Vast.ai (Option B)
4. **Keep GCP as backup:** Filestore + Cloud SQL at ~$58/mo while validating new provider
