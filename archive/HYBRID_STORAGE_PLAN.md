# Hybrid Storage Plan: Filestore SSD + Ephemeral Cache

## Architecture (Tiered Storage)

```
┌──────────────────────────────────────────────────────────────────┐
│ Cloud Run GPU Worker (neovlab-gpu-worker-east4)                 │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ ComfyUI Container                                          │ │
│  │                                                             │ │
│  │  /models/hot/ ────────────────┐  (Filestore SSD)          │ │
│  │    └─ wan/diffusion_pytorch_model.safetensors (91GB)       │ │
│  │       Priority: HOT (most frequently accessed)             │ │
│  │       Speed: <10s load time                                │ │
│  │                                │                            │ │
│  │  /models/archive/ ────────────┤  (GCS FUSE)               │ │
│  │    ├─ wan/                     │  Complete model archive   │ │
│  │    │   └─ diffusion_pytorch_model.safetensors (backup)     │ │
│  │    ├─ sd/                      │                            │ │
│  │    │   ├─ vae/ (254MB)         │                            │ │
│  │    │   ├─ clip/ (7GB)          │                            │ │
│  │    │   └─ checkpoints/ (21GB)  │                            │ │
│  │    ├─ overflow/                │  Models >100GB total      │ │
│  │    └─ cold/                    │  Rarely used models       │ │
│  │       Priority: COLD (fallback/overflow)                   │ │
│  │       Speed: 15-20 min load time                           │ │
│  │                                │                            │ │
│  │  /tmp/cache/ ─────────────────┤  (Ephemeral)              │ │
│  │    ├─ vae/                     │                            │ │
│  │    ├─ clip/                    │                            │ │
│  │    └─ checkpoints/             │                            │ │
│  │       Priority: WARM (populated from GCS on cold start)    │ │
│  │       Speed: ~3s access after cache populated              │ │
│  │                                │                            │ │
│  │  /outputs/ ───────────────────┘  (GCS FUSE)               │ │
│  │    └─ Job artifacts written here, streamed to GCS          │ │
│  │                                                             │ │
│  │  ComfyUI model search order:                               │ │
│  │    1. /tmp/cache/ (ephemeral, fastest)                     │ │
│  │    2. /models/hot/ (Filestore SSD, fast)                   │ │
│  │    3. /models/archive/ (GCS FUSE, fallback)                │ │
│  └────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
         │                                    │
         │ NFS mount                          │ GCS FUSE mount  
         ▼                                    ▼
┌───────────────────────┐   ┌────────────────────────────────────┐
│ Filestore Lite        │   │ GCS Standard Storage               │
│ (us-east4)            │   │ neovnext-neovlab-models            │
│                       │   │                                    │
│ 100GB SSD-backed NFS  │   │ 119GB+ total (unlimited):          │
│ $12/month             │   │  • wan/ (91GB, backup)             │
│                       │   │  • sd/ (28GB, primary)             │
│ hot/                  │   │  • overflow/ (future)              │
│  └─ wan/              │   │  • cold/ (rarely used)             │
│     └─ diffusion_*    │   │                                    │
│        (91GB HOT)     │   │ neovnext-neovlab-io-cache          │
│                       │   │  • outputs/ (job artifacts)        │
│ 9GB free for:         │   │                                    │
│  • Additional WAN     │   │ Total: $2.38-3.00/month            │
│  • Future hot models  │   │ (storage only, no egress)          │
└───────────────────────┘   └────────────────────────────────────┘
```

## Cost Analysis

### Storage Costs (Monthly)
| Component                  | Size    | Rate           | Cost/Month |
|----------------------------|---------|----------------|------------|
| Filestore Lite (SSD)       | 100GB   | $0.12/GB       | **$12.00** |
| GCS Standard (archive)     | 119GB   | $0.020/GB      | **$2.38**  |
| GCS Standard (outputs)     | ~20GB   | $0.020/GB      | **$0.40**  |
| Ephemeral cache (/tmp)     | 32GB    | Free           | **$0.00**  |
| **Total Storage**          |         |                | **$14.78** |

*Note: GCS archive includes wan/ (91GB backup), sd/ (28GB), plus headroom for overflow/cold models*

### Compute Costs (Unchanged)
| Component              | Rate           | Usage Pattern           | Est. Monthly |
|------------------------|----------------|-------------------------|--------------|
| L4 GPU + 16Gi memory   | $3.20/hour     | ~1 hour/day testing     | ~$96         |
| **Total Compute**      |                |                         | **~$96**     |

### Total Monthly Cost
- **Storage**: $14.78/month (↑ from $2.38)
- **Compute**: ~$96/month (unchanged)
- **Total**: ~$110.78/month

### Free Credit Runway
- Free credits: $300
- Burn rate: $110.78/month
- **Runway**: ~2.7 months of active development

*Note: Actual compute costs vary by usage. 1 hour/day is conservative estimate.*

## Performance Improvements

### Cold Start Timeline (Before)
```
0s ────────────────────────────────────────────> 20-28 min
   │                                             │
   ├─ Symlink setup (5s)                        │
   ├─ Spawn ComfyUI (34s) ────┐                 │
   ├─ Cache SD models (3-5 min) │               │
   ├─ First job starts         │                │
   └─ Load WAN model from GCS (15-20 min) ──────┘
```

### Cold Start Timeline (After)
```
0s ───────────────────────> 5-7 min
   │                        │
   ├─ Mount Filestore (2s) │
   ├─ Symlink setup (5s)   │
   ├─ Spawn ComfyUI (34s) ─┐
   ├─ Cache SD models (3-5 min, parallel)
   ├─ First job starts     │
   └─ Load WAN from Filestore (<10s) ───────────┘
```

### Improvement Summary
- **WAN model load**: 15-20 min → <10s (120x faster)
- **Cold start total**: 20-28 min → 5-7 min (3-4x faster)
- **Warm start**: 34s (unchanged, already using cache)

## Implementation Steps

### 1. Provision Filestore Instance

```bash
# Create Filestore Lite instance (100GB SSD, us-east4)
gcloud filestore instances create neovlab-models-wan \
  --zone=us-east4-a \
  --tier=BASIC_SSD \
  --file-share=name="wan_models",capacity=100GB \
  --network=name="default" \
  --project=neovnext

# Get the NFS mount details
gcloud filestore instances describe neovlab-models-wan \
  --zone=us-east4-a \
  --project=neovnext \
  --format="value(networks[0].ipAddresses[0])"
```

*Output will be IP like `10.x.x.x`*

### 2. Populate Filestore with Hot WAN Models

```powershell
# From your Windows machine with gcloud + gcsfuse

# Mount Filestore (requires NFS client)
# Windows: Install NFS Client via Windows Features
# Then mount:
mount -o anon \\10.x.x.x\wan_models Z:\filestore

# Create hot models directory
mkdir Z:\filestore\hot\wan

# Copy WAN model from GCS to Filestore (hot tier)
gsutil -m cp gs://neovnext-neovlab-models/wan/diffusion_pytorch_model.safetensors Z:\filestore\hot\wan\

# Verify size (should be ~91GB)
dir Z:\filestore\hot\wan\

# Unmount
umount Z:\filestore
```

*Alternative: Use a temporary GCE VM to copy if Windows NFS is problematic.*

*Note: GCS still retains the full model archive at gs://neovnext-neovlab-models/wan/ as backup.*

### 3. Update Deployment Script

Edit `docker/deploy-cloudrun-gpu.sh`:

```bash
# Add after line 60 (environment variables section)
COMFYUI_TIMEOUT_SECONDS=2000
COMFYUI_STARTUP_TIMEOUT_SECONDS=420
FILESTORE_IP=10.x.x.x  # Replace with actual IP from step 1
FILESTORE_SHARE=wan_models
GCS_MODEL_BUCKET=neovnext-neovlab-models
GCS_OUTPUT_BUCKET=neovnext-neovlab-io-cache

# Add volume mounts (after line 77)
gcloud run services update ${SERVICE_NAME} \
  --image ${IMAGE_NAME} \
  --region ${REGION} \
  --platform managed \
  --set-env-vars="$(echo "$ENV_VARS" | tr '\n' ',')" \
  --add-volume=name=hot-models,type=nfs,location=${FILESTORE_IP}/${FILESTORE_SHARE} \
  --add-volume=name=archive-models,type=cloud-storage,bucket=${GCS_MODEL_BUCKET} \
  --add-volume=name=outputs,type=cloud-storage,bucket=${GCS_OUTPUT_BUCKET} \
  --add-volume-mount=volume=hot-models,mount-path=/models/hot \
  --add-volume-mount=volume=archive-models,mount-path=/models/archive \
  --add-volume-mount=volume=outputs,mount-path=/outputs \
  --timeout 2100 \
  --cpu 4 \
  --memory 16Gi \
  --gpu 1 \
  --gpu-type nvidia-l4 \
  --min-instances 0 \
  --max-instances 3 \
  --startup-cpu-boost \
  --startup-probe=initialDelaySeconds=300,periodSeconds=15,failureThreshold=10
```

*Note: Cloud Run GCS FUSE mounts use `type=cloud-storage` natively, no gcsfuse binary needed.*

### 4. Update Entrypoint Script

Edit `docker/comfyui-entrypoint.sh` to configure tiered model lookup:

```bash
# Around line 70 - Update symlink section for tiered storage

echo "[entrypoint] Creating ComfyUI model directory structure (tiered)..."

# Tier 1: Ephemeral cache (fastest, warm start)
mkdir -p /tmp/vae /tmp/clip /tmp/checkpoints
ln -sf /tmp/vae /app/models/vae
ln -sf /tmp/clip /app/models/clip  
ln -sf /tmp/checkpoints /app/models/checkpoints

# Tier 2: Filestore SSD (fast, hot models)
# ComfyUI searches extra_model_paths for diffusion models
mkdir -p /app/models/diffusion_models
ln -sf /models/hot/wan /app/models/diffusion_models/hot

# Tier 3: GCS FUSE archive (fallback, cold/overflow models)
ln -sf /models/archive/wan /app/models/diffusion_models/archive
ln -sf /models/archive/sd /app/models/sd_archive
ln -sf /models/archive/overflow /app/models/diffusion_models/overflow
ln -sf /models/archive/cold /app/models/diffusion_models/cold

echo "[entrypoint] Symlinks created:"
ls -lh /app/models/
ls -lh /app/models/diffusion_models/

# Around line 95 - Update cache population to only cache SD models

populate_cache_if_cold() {
    echo "[entrypoint-cache] Checking cache warmth..."
    
    # Check if cache markers exist
    if [[ -f /tmp/vae/.cached && -f /tmp/clip/.cached && -f /tmp/checkpoints/.cached ]]; then
        echo "[entrypoint-cache] Cache is warm (all markers present). Skipping population."
        return 0
    fi
    
    echo "[entrypoint-cache] Cache is cold. Populating SD models from GCS archive..."
    local start_time=$(date +%s)
    
    # Only cache SD models from GCS archive, NOT WAN (WAN lives on Filestore)
    mkdir -p /tmp/vae /tmp/clip /tmp/checkpoints
    
    if [[ -d /models/archive/sd/vae ]] && [[ ! -f /tmp/vae/.cached ]]; then
        echo "[entrypoint-cache] Copying VAE models..."
        cp -r /models/archive/sd/vae/* /tmp/vae/ 2>/dev/null || true
        touch /tmp/vae/.cached
    fi
    
    if [[ -d /models/archive/sd/clip ]] && [[ ! -f /tmp/clip/.cached ]]; then
        echo "[entrypoint-cache] Copying CLIP models..."
        cp -r /models/archive/sd/clip/* /tmp/clip/ 2>/dev/null || true
        touch /tmp/clip/.cached
    fi
    
    if [[ -d /models/archive/sd/checkpoints ]] && [[ ! -f /tmp/checkpoints/.cached ]]; then
        echo "[entrypoint-cache] Copying checkpoint models..."
        cp -r /models/archive/sd/checkpoints/* /tmp/checkpoints/ 2>/dev/null || true
        touch /tmp/checkpoints/.cached
    fi
    
    local end_time=$(date +%s)
    local duration=$((end_time - start_time))
    echo "[entrypoint-cache] SD model cache populated in ${duration}s"
    echo "[entrypoint-cache] WAN models available on Filestore at /models/hot/wan/"
}
```

**ComfyUI Model Search Order:**
ComfyUI will search for models in this order:
1. `/app/models/vae/` → `/tmp/vae/` (cache, 254MB, ~3s access)
2. `/app/models/clip/` → `/tmp/clip/` (cache, 7GB, ~3s access)
3. `/app/models/checkpoints/` → `/tmp/checkpoints/` (cache, 21GB, ~3s access)
4. `/app/models/diffusion_models/hot/` → `/models/hot/wan/` (Filestore, 91GB, <10s access)
5. `/app/models/diffusion_models/archive/` → `/models/archive/wan/` (GCS FUSE, fallback)
6. `/app/models/diffusion_models/overflow/` → `/models/archive/overflow/` (GCS FUSE, future)
7. `/app/models/diffusion_models/cold/` → `/models/archive/cold/` (GCS FUSE, rarely used)

**Output Handling:**
```bash
# ComfyUI outputs are written to /outputs (GCS FUSE mount)
# This streams directly to gs://neovnext-neovlab-io-cache/outputs/
# No local disk consumption
```

### 5. Organize GCS Bucket Structure

Update bucket structure to support tiered storage:

```
gs://neovnext-neovlab-models/           # Model archive (read-only from worker)
├── wan/                                # WAN models (backed up, hot copy on Filestore)
│   └── diffusion_pytorch_model.safetensors (91GB)
├── sd/                                 # SD models (streamed/cached)
│   ├── vae/
│   ├── clip/
│   └── checkpoints/
├── overflow/                           # Models that don't fit in 100GB Filestore
│   └── (future large models)
└── cold/                               # Rarely used models (cost-optimized storage)
    └── (infrequently accessed models)

gs://neovnext-neovlab-io-cache/        # Job outputs (write from worker)
└── outputs/
    ├── {job-id}/
    │   ├── output_001.png
    │   └── output_002.png
    └── (dynamically written by ComfyUI)
```

**Storage Strategy:**
- **hot/**: Filestore SSD (fast, expensive) — most frequently accessed models
- **archive/**: GCS Standard (slow, cheap) — complete backup, fallback, overflow
- **cache/**: Ephemeral /tmp (free, cleared on restart) — warm cache for small models
- **outputs/**: GCS Standard (cheap persistence) — job artifacts

**Model Promotion Workflow (Optional):**
```bash
# If a model in archive/ becomes frequently accessed, copy to Filestore hot/:
gsutil cp gs://neovnext-neovlab-models/overflow/new_diffusion_model.safetensors /tmp/
gcloud compute ssh filestore-mount-vm --zone=us-east4-a
cp /tmp/new_diffusion_model.safetensors /models/hot/wan/
```

### 6. Deploy

```bash
cd docker/
REBUILD_IMAGE=1 ./deploy-cloudrun-gpu.sh neovnext us-east4
```

### 7. Verify

```bash
# Check logs for Filestore mount
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=neovlab-gpu-worker-east4 AND textPayload=~'/models/hot'" \
  --project=neovnext --limit=20

# Check logs for GCS FUSE archive mount
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=neovlab-gpu-worker-east4 AND textPayload=~'/models/archive'" \
  --project=neovnext --limit=20

# Check cold start timing (should see cache + hot tier)
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=neovlab-gpu-worker-east4 AND textPayload=~'entrypoint-cache|hot|archive'" \
  --project=neovnext --limit=50 --format="value(timestamp,textPayload)"

# Verify WAN model accessible from hot tier
gcloud run services proxy neovlab-gpu-worker-east4 --region=us-east4 &
# Then test via local API call that triggers WAN workflow

# Verify outputs written to GCS
gsutil ls gs://neovnext-neovlab-io-cache/outputs/
```

## Rollback Plan

If Filestore causes issues:

```bash
# Remove volume mount
gcloud run services update neovlab-gpu-worker-east4 \
  --region us-east4 \
  --clear-volumes \
  --clear-volume-mounts

# Delete Filestore instance (stops billing)
gcloud filestore instances delete neovlab-models-wan \
  --zone=us-east4-a \
  --project=neovnext
```

Revert entrypoint changes via git:
```bash
git checkout docker/comfyui-entrypoint.sh
REBUILD_IMAGE=1 ./deploy-cloudrun-gpu.sh neovnext us-east4
```

## Decision Matrix

| Metric                  | Current (GCS only) | Tiered (Filestore + GCS) | Improvement |
|-------------------------|--------------------|--------------------------|-------------|
| Storage cost            | $2.38/month        | $14.78/month             | ↑ $12.40    |
| WAN model load time     | 15-20 min          | <10 seconds              | ↓ ~20 min   |
| Cold start (WAN job)    | 20-28 min          | 5-7 min                  | ↓ 15-21 min |
| Cold start (SD job)     | 3-5 min            | 3-5 min                  | No change   |
| Output write speed      | GCS stream         | GCS stream (direct)      | Unchanged   |
| Model overflow capacity | Unlimited (slow)   | 9GB hot + unlimited cold | 9GB fast    |
| Free credit runway      | 25 months          | 2.7 months               | ↓ 22 months |

**Recommendation:** Deploy tiered storage if:
- WAN workflows are >20% of your jobs
- Cold start speed matters more than credit burn rate
- You need to iterate quickly during development (fast feedback loop)

**Alternative:** Stay with GCS-only during initial development, migrate to tiered storage when:
- You're burning <$50/month in credits (still have $250+ runway)
- WAN workflow usage increases
- You're confident in the product direction (worth the storage investment)

## Notes

- **Filestore Lite minimum:** 100GB (cannot provision smaller)
- **Region lock:** Filestore must be in same region as Cloud Run (us-east4)
- **Network:** Filestore uses default VPC network (same as Cloud Run)
- **Backup strategy:** GCS archive is source-of-truth, Filestore is hot cache (can be recreated from GCS)
- **Filestore snapshots:** Available but not needed for models (GCS is backup)
- **Scaling:** All Cloud Run instances share the same Filestore mount (NFS is multi-client)
- **GCS as overflow:** When Filestore fills up (100GB), add new models to `gs://neovnext-neovlab-models/overflow/`
- **Model lifecycle:**
  - Upload new models to GCS archive (cheap, persistent)
  - Copy hot models to Filestore (fast access)
  - Move rarely-used models from Filestore back to GCS `cold/` (reclaim space)
- **Output persistence:** Job artifacts written to GCS via FUSE mount (no ephemeral storage used)
  - Outputs survive container restarts
  - Can be accessed from any worker instance
  - No manual upload/download needed
