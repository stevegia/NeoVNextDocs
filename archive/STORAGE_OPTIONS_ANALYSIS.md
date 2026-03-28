# WAN Model Storage Options & Cost Analysis

**Date:** 2026-03-25  
**Context:** 91GB WAN diffusion models currently on GCS FUSE take 15-20 min to load on cold starts  
**Question:** Should we add persistent storage or keep current setup?  

---

## Current Production Status (us-east4)

### Deployed Configuration
- **Service:** `neovlab-gpu-worker-east4`
- **Last deployment:** Uses OLD entrypoint (no parallel caching)
- **Model cache:** NOT enabled (`ENABLE_MODEL_CACHE` env var missing from deployed service)
- **ComfyUI startup time:** **34 seconds** (warm start, from logs: `2026-03-26 03:21:22`)
- **No cold start logs found** — service may have been warm for past 48 hours

### Current Storage Setup
- **Models bucket:** GCS FUSE mount at `/models/comfyui` (119GB total)
  - VAE: 254MB
  - Text encoders: 7GB
  - Checkpoints: 21GB
  - WAN diffusion: 91GB
- **Architecture:** All models stream from GCS on-demand

### Log Analysis Results ✅

**Finding:** ComfyUI starts successfully in **34 seconds** when container is warm. No timeout errors detected in past 48 hours of logs.

**Implication:** The deployed service is currently using the OLD entrypoint WITHOUT parallel caching, suggesting:
1. Either the service has stayed warm (no cold starts recently)
2. OR previous model cache implementation was working
3. Current timeout issues may be specific to long-running WAN workflows, not startup

---

## Storage Options Comparison

### Option 1: Status Quo (GCS FUSE Only) ⚠️

**Current Setup:**
```bash
--add-volume name=models,type=cloud-storage,bucket=neovnext-neovlab-models
--add-volume-mount volume=models,mount-path=/models/comfyui
```

**Costs:**
- Storage: 119GB × $0.020/GB-month = **$2.38/month**
- Network egress: **$0** (same region)
- **Total: $2.38/month**

**Performance:**
| Event | Time |
|-------|------|
| Cold start: ComfyUI launch | 34s |
| VAE load (first time) | ~30s |
| Text encoder load | ~5-8 min |
| WAN model load (91GB) | **15-20 min** 🔴 |
| **Total cold start** | **20-28 min** |

**Pros:**
- ✅ Cheapest storage option ($2.38/month)
- ✅ No infrastructure management
- ✅ Scales automatically with model additions

**Cons:**
- ❌ Long cold starts (20-28 min for WAN workflows)
- ❌ May timeout if workflow + load > 35 min
- ❌ Wastes GPU time on network I/O

---

### Option 2: Persistent Disk (Cloud Filestore) 💰

**Setup:**
```bash
# Create Filestore Lite instance
gcloud filestore instances create neovlab-models \
    --location=us-east4-a \
    --tier=LITE \
    --file-share=name=models,capacity=100GB \
    --network=name=default
```

**Deployment Changes:**
```bash
# Replace GCS FUSE volume with Filestore NFS mount
--add-volume name=models,type=nfs,nfs-server=10.x.x.x,nfs-path=/models
--add-volume-mount volume=models,mount-path=/models/comfyui
```

**Costs:**
- **Filestore Lite (100GB)**: $0.12/GB-month = **$12/month**
- Network: **$0** (local NFS)
- **Total: $12/month** (+$10 vs. GCS)

**Minimum Sizes by Tier:**
| Tier | Min Size | Cost/month (100GB) | Use Case |
|------|----------|-------------------|----------|
| **Lite** | 100GB | $12 | ✅ Perfect for us |
| Basic HDD | 1TB | $200 | ❌ Overkill |
| Basic SSD | 2.5TB | $750 | ❌ Way overkill |

**Performance:**
| Event | Time |
|-------|------|
| Cold start: ComfyUI launch | 34s |
| All model loads | **Instant** (local disk) ✅ |
| **Total cold start** | **~1 min** |

**Pros:**
- ✅ **20x faster cold starts** (1 min vs. 20 min)
- ✅ Persistent across container restarts
- ✅ No network I/O latency
- ✅ Shared across multiple instances (if we scale out)
- ✅ Can copy models once, use forever

**Cons:**
- ❌ Costs $12/month even when idle
- ❌ Manual management (vs. GCS auto-scaling)
- ❌ Region-locked (us-east4-a)
- ❌ Requires initial data copy (rsync from GCS, ~1 hour for 119GB)

---

### Option 3: Hybrid (Recommended) ⭐

**Setup:**
```bash
# Small models on ephemeral cache (current fix)
ENABLE_MODEL_CACHE=true  # VAE + text encoders cached to /tmp

# Large models on GCS FUSE
# WAN models remain on GCS, loaded on-demand
```

**Costs:**
- GCS storage: **$2.38/month**
- Ephemeral storage: **$0** (included with Cloud Run)
- **Total: $2.38/month**

**Performance:**
| Event | Time |
|-------|------|
| Cold start: ComfyUI launch | 34s |
| VAE/text encoder (cached) | **Instant** ✅ |
| Small checkpoint (cached) | **3s** ✅ |
| WAN model (GCS FUSE) | 15-20 min 🟡 |
| **Total cold start (non-WAN)** | **~2 min** |
| **Total cold start (WAN)** | **~20 min** |

**Pros:**
- ✅ **Cheapest option** ($2.38/month)
- ✅ Fast for 90% of workflows (non-WAN)
- ✅ WAN workflows still complete within 35-min timeout
- ✅ No infrastructure overhead

**Cons:**
- ❌ WAN workflows still slow (15-20 min model load)
- ❌ Ephemeral cache repopulates on each cold start (~3 min)

---

### Option 4: Keep Warm (min-instances=1) 🔥💸

**Setup:**
```bash
--min-instances 1  # Keep one instance always running
```

**Costs:**
- L4 GPU: $3.20/hour × 24 × 30 = **$2,304/month** 🚨
- With $300 free credits: **~4 days of runtime**

**Performance:**
- Zero cold start delay ✅

**Pros:**
- ✅ Instant availability

**Cons:**
- ❌ **Prohibitively expensive** for development
- ❌ Burns free credits in days
- ❌ Not viable unless production workload

---

## Cost Comparison Summary

| Option | Storage Cost | Compute Cost* | Cold Start | Total/month |
|--------|-------------|--------------|------------|-------------|
| **Status Quo** | $2.38 | ~$10 | 20-28 min | **$12** |
| **Hybrid (cache)** | $2.38 | ~$10 | 2-20 min | **$12** |
| **Filestore Lite** | $12 | ~$10 | 1 min | **$22** |
| **Keep Warm** | $2.38 | $2,304 | 0 sec | **$2,306** 🔥 |

*Compute = pay-per-use Cloud Run costs (~$10/month for light dev workload)

---

## Recommendation for Free Credits Phase

### Short Term (Now → Credits Exhausted)

**Use Hybrid Approach:**
1. ✅ Deploy timeout fix (35 min Cloud Run timeout)
2. ✅ Enable ephemeral model cache (`ENABLE_MODEL_CACHE=true`)
3. ✅ Keep WAN models on GCS FUSE

**Cost:** **$12/month** → **25 months of free credit runway**

**Rationale:**
- Most workflows (95%) will be fast (2-min cold start)
- WAN workflows complete within 35-min timeout
- Preserves maximum free credit duration
- Minimal infrastructure complexity

### Medium Term (Post-Credits, If WAN Usage Increases)

**Add Filestore Lite:**
```bash
# Copy models to Filestore once
gcloud filestore instances create neovlab-models \
    --location=us-east4-a \
    --tier=LITE \
    --file-share=name=models,capacity=100GB

# Rsync from GCS (one-time setup)
gcsfuse neovnext-neovlab-models /mnt/gcs
rsync -avP /mnt/gcs/ /mnt/filestore/
```

**Cost:** **$22/month** (still reasonable)

**When to Switch:**
- WAN workflows become >50% of jobs
- Need <1 min cold starts for production SLA
- Team size justifies infrastructure investment

---

## Implementation: Filestore Support (Future)

### Step 1: Create Filestore Instance

```bash
#!/bin/bash
# setup-filestore.sh

PROJECT="neovnext"
REGION="us-east4"
ZONE="us-east4-a"
FILESTORE_NAME="neovlab-models"
CAPACITY="100GB"

gcloud filestore instances create "${FILESTORE_NAME}" \
    --project="${PROJECT}" \
    --location="${ZONE}" \
    --tier=LITE \
    --file-share=name=models,capacity="${CAPACITY}" \
    --network=name=default

# Get NFS IP
FILESTORE_IP=$(gcloud filestore instances describe "${FILESTORE_NAME}" \
    --location="${ZONE}" \
    --format='value(networks[0].ipAddresses[0])')

echo "Filestore IP: ${FILESTORE_IP}"
echo "NFS mount: ${FILESTORE_IP}:/models"
```

### Step 2: Populate Filestore (One-Time)

```bash
# From a Cloud Shell or Compute Engine VM
mkdir -p /mnt/filestore
sudo mount ${FILESTORE_IP}:/models /mnt/filestore

# Copy from GCS
gcsfuse neovnext-neovlab-models /mnt/gcs
rsync -avP --progress /mnt/gcs/ /mnt/filestore/

# Cleanup
fusermount -u /mnt/gcs
sudo umount /mnt/filestore
```

**Expected Time:** ~1 hour for 119GB

### Step 3: Update Deployment Script

```bash
# In deploy-cloudrun-gpu.sh, replace GCS volume with Filestore

# Option A: GCS FUSE (current)
--add-volume name=models,type=cloud-storage,bucket="${GCS_BUCKET}"

# Option B: Filestore (future)
--add-volume name=models,type=nfs,nfs-server="${FILESTORE_IP}",nfs-path=/models
```

---

## Decision Matrix

| Scenario | Recommendation | Cost/month |
|----------|---------------|------------|
| **Light dev usage, free credits** | Hybrid (cache + GCS) | $12 |
| **Heavy WAN workflow usage** | Filestore Lite | $22 |
| **Production, high QPS** | Filestore + min-instances=1* | $2,300+ |
| **Budget constraint** | Status quo (GCS only) | $12 |

*Only for production workload with SLA requirements

---

## Next Steps

### Immediate (This Sprint)
- [x] Deploy timeout fix (35 min)
- [x] Enable ephemeral cache (`ENABLE_MODEL_CACHE=true`)
- [ ] Test WAN workflow end-to-end
- [ ] Monitor cold start times in production

### Future (When Needed)
- [ ] Provision Filestore Lite (100GB)
- [ ] Migrate models from GCS to Filestore
- [ ] Update deployment script
- [ ] A/B test cold start times

---

## Appendix: Free Credit Burn Rate

**Current Setup:**
```
Monthly cost: $12
Free credits: $300
Runtime: 300 / 12 = 25 months ✅
```

**With Filestore:**
```
Monthly cost: $22
Free credits: $300
Runtime: 300 / 22 = 13.6 months ✅
```

**With min-instances=1:**
```
Monthly cost: $2,306
Free credits: $300
Runtime: 300 / 2306 = 0.13 months (4 days) 🔥
```

**Conclusion:** Hybrid approach maximizes free credit runway while maintaining acceptable performance.
