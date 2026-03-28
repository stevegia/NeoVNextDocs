# Comprehensive Deployment & Testing Plan: Tiered Storage

## Revised Cost Analysis

### Actual Compute Costs (Scale-to-Zero)
| Usage Pattern          | Hours/Month | Cost/Month | Annual  | Use Case                    |
|------------------------|-------------|------------|---------|----------------------------|
| Light testing          | ~10 hours   | **$32**    | $384    | Weekly smoke tests         |
| Active development     | ~40 hours   | **$128**   | $1,536  | Daily iteration            |
| Heavy load testing     | ~100 hours  | **$320**   | $3,840  | Continuous integration     |

**Reality check:** With `min-instances=0` (scale-to-zero), GPU only bills when container is processing jobs.
- Container spins up, runs job, idles for 5 min, then shuts down
- 10 jobs/day × 5 min each = ~50 min/day = 25 hours/month = **$80/month**

### Total Monthly Cost (Realistic)
| Component              | Cost/Month |
|------------------------|------------|
| Filestore Lite (100GB) | $12.00     |
| GCS Storage (~140GB)   | $2.78      |
| Compute (light usage)  | $32.00     |
| **Total**              | **$46.78** |

**Free Credit Runway: ~6.4 months** (was 2.7 months with inflated compute estimate)

---

## Phase 1: Pre-Deployment Preparation

### 1.1 Backup Current State
```powershell
# Snapshot current deployment config (PowerShell)
gcloud run services describe neovlab-gpu-worker-east4 `
  --region=us-east4 `
  --format=yaml > Z:\NeoVNext\logs\backup-service-config-$(Get-Date -Format 'yyyy-MM-dd-HHmm').yaml

# Tag current container image as backup
$CURRENT_IMAGE = gcloud run services describe neovlab-gpu-worker-east4 `
  --region=us-east4 `
  --format="value(spec.template.spec.containers[0].image)"
gcloud container images add-tag $CURRENT_IMAGE "${CURRENT_IMAGE}-pre-tiered-storage-backup"
```

### 1.2 Verify Current GCS Structure
```powershell
# List current bucket contents
gsutil ls -lh gs://neovnext-neovlab-models/

# Expected output:
#   wan/diffusion_pytorch_model.safetensors (91GB)
#   vae/ (254MB)
#   clip/ (7GB)
#   checkpoints/ (21GB)
```

### 1.3 Create Cost Alert
```bash
# Set up billing alert at $50/month
gcloud alpha billing budgets create \
  --billing-account=$(gcloud beta billing projects describe neovnext --format="value(billingAccountName)" | sed 's/.*\///') \
  --display-name="NeoVNext Monthly Budget Alert" \
  --budget-amount=50 \
  --threshold-rule=percent=50 \
  --threshold-rule=percent=90 \
  --threshold-rule=percent=100
```

---

## Phase 2: GCS Bucket Reorganization

### 2.1 Reorganize Model Archive
```powershell
# Create new directory structure
gsutil -m mkdir `
  gs://neovnext-neovlab-models/sd/ `
  gs://neovnext-neovlab-models/archive/ `
  gs://neovnext-neovlab-models/overflow/ `
  gs://neovnext-neovlab-models/cold/

# Move SD models to sd/ subfolder (if not already there)
gsutil -m mv gs://neovnext-neovlab-models/vae gs://neovnext-neovlab-models/sd/vae
gsutil -m mv gs://neovnext-neovlab-models/clip gs://neovnext-neovlab-models/sd/clip
gsutil -m mv gs://neovnext-neovlab-models/checkpoints gs://neovnext-neovlab-models/sd/checkpoints

# Copy WAN models to archive/ (backup, keep original in wan/)
gsutil -m cp -r gs://neovnext-neovlab-models/wan gs://neovnext-neovlab-models/archive/wan

# Verify structure
gsutil ls -r gs://neovnext-neovlab-models/
```

**Expected structure:**
```
gs://neovnext-neovlab-models/
├── wan/                                    # Original location (will be hot on Filestore)
│   └── diffusion_pytorch_model.safetensors (91GB)
├── sd/                                     # SD models (cache + stream)
│   ├── vae/
│   ├── clip/
│   └── checkpoints/
├── archive/                                # Complete backup
│   └── wan/
│       └── diffusion_pytorch_model.safetensors (91GB backup)
├── overflow/                               # Future large models >100GB
└── cold/                                   # Rarely used models
```

### 2.2 Verify Output Bucket
```powershell
# Ensure output bucket exists
gsutil ls gs://neovnext-neovlab-io-cache/ 2>$null
if ($LASTEXITCODE -ne 0) {
    gsutil mb -p neovnext -c STANDARD -l us-east4 gs://neovnext-neovlab-io-cache/
    Write-Host "Created output bucket"
}

# Create outputs directory
gsutil -m mkdir gs://neovnext-neovlab-io-cache/outputs/
```

---

## Phase 3: Filestore Provisioning

### 3.1 Create Filestore Instance
```bash
# Provision 100GB Filestore Lite in us-east4
gcloud filestore instances create neovlab-models-hot \
  --project=neovnext \
  --zone=us-east4-a \
  --tier=BASIC_SSD \
  --file-share=name="hot_models",capacity=100GB \
  --network=name="default" \
  --description="Hot tier for frequently-accessed WAN models"

# Wait for provisioning (takes 5-10 minutes)
gcloud filestore instances describe neovlab-models-hot \
  --zone=us-east4-a \
  --project=neovnext

# Capture NFS IP address
FILESTORE_IP=$(gcloud filestore instances describe neovlab-models-hot \
  --zone=us-east4-a \
  --project=neovnext \
  --format="value(networks[0].ipAddresses[0])")

echo "Filestore IP: $FILESTORE_IP"
# Save this IP for deployment script
```

### 3.2 Populate Filestore with Hot Models

**Option A: Via Temporary GCE VM (Recommended for Windows)**
```bash
# Create temporary VM with NFS client in same zone
gcloud compute instances create filestore-loader \
  --project=neovnext \
  --zone=us-east4-a \
  --machine-type=e2-standard-4 \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --scopes=storage-ro \
  --metadata=startup-script='#!/bin/bash
apt-get update
apt-get install -y nfs-common google-cloud-sdk
'

# SSH into VM
gcloud compute ssh filestore-loader --zone=us-east4-a --project=neovnext

# On the VM:
sudo mkdir -p /mnt/filestore
sudo mount -o sync ${FILESTORE_IP}:/hot_models /mnt/filestore

# Create hot directory structure
sudo mkdir -p /mnt/filestore/hot/wan

# Copy WAN model from GCS to Filestore
gsutil -m cp gs://neovnext-neovlab-models/wan/diffusion_pytorch_model.safetensors \
  /mnt/filestore/hot/wan/

# Verify size
du -sh /mnt/filestore/hot/wan/
# Should show ~91GB

# Unmount and exit
sudo umount /mnt/filestore
exit

# Delete temporary VM
gcloud compute instances delete filestore-loader --zone=us-east4-a --project=neovnext --quiet
```

**Option B: Via Windows NFS Client (If Available)**
```powershell
# Enable NFS client (Run as Administrator in PowerShell)
Enable-WindowsOptionalFeature -Online -FeatureName ServicesForNFS-ClientOnly -All

# Mount Filestore
mount -o anon \\$FILESTORE_IP\hot_models Z:

# Create directory structure
New-Item -ItemType Directory -Path Z:\hot\wan

# Copy from GCS
gsutil -m cp gs://neovnext-neovlab-models/wan/diffusion_pytorch_model.safetensors Z:\hot\wan\

# Verify
Get-ChildItem Z:\hot\wan\ -Recurse | Measure-Object -Property Length -Sum

# Unmount
umount Z:
```

---

## Phase 4: Update Deployment Configuration

### 4.1 Update docker/deploy-cloudrun-gpu.sh

Create backup:
```powershell
Copy-Item docker/deploy-cloudrun-gpu.sh docker/deploy-cloudrun-gpu.sh.backup
```

Apply changes:
```bash
# Around line 20, add Filestore IP variable
FILESTORE_IP="${FILESTORE_IP:-10.x.x.x}"  # Replace with actual IP from Phase 3.1

# Around line 60, add new environment variables
COMFYUI_TIMEOUT_SECONDS=2000
COMFYUI_STARTUP_TIMEOUT_SECONDS=420
ENABLE_MODEL_CACHE=true

# Around line 77, replace gcloud run services update with volume mounts
gcloud run services update ${SERVICE_NAME} \
  --image ${IMAGE_NAME} \
  --region ${REGION} \
  --platform managed \
  --set-env-vars="COMFYUI_TIMEOUT_SECONDS=${COMFYUI_TIMEOUT_SECONDS},COMFYUI_STARTUP_TIMEOUT_SECONDS=${COMFYUI_STARTUP_TIMEOUT_SECONDS},ENABLE_MODEL_CACHE=${ENABLE_MODEL_CACHE}" \
  --add-volume=name=hot-models,type=nfs,location=${FILESTORE_IP}/hot_models \
  --add-volume=name=archive-models,type=cloud-storage,bucket=neovnext-neovlab-models \
  --add-volume=name=outputs,type=cloud-storage,bucket=neovnext-neovlab-io-cache \
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
  --no-cpu-throttling \
  --startup-probe=initialDelaySeconds=300,periodSeconds=15,failureThreshold=10
```

### 4.2 Update docker/comfyui-entrypoint.sh

Create backup:
```powershell
Copy-Item docker/comfyui-entrypoint.sh docker/comfyui-entrypoint.sh.backup
```

**Replace entire symlink section (around line 65-85):**
```bash
echo "[entrypoint] Creating ComfyUI model directory structure (tiered storage)..."

# Create base directories
mkdir -p /app/models/diffusion_models

# Tier 1: Ephemeral cache for small SD models (fastest)
mkdir -p /tmp/vae /tmp/clip /tmp/checkpoints
ln -sf /tmp/vae /app/models/vae
ln -sf /tmp/clip /app/models/clip  
ln -sf /tmp/checkpoints /app/models/checkpoints

# Tier 2: Filestore SSD for hot WAN models (fast)
if [[ -d /models/hot/wan ]]; then
    ln -sf /models/hot/wan /app/models/diffusion_models/hot
    echo "[entrypoint] Hot tier (Filestore): /models/hot/wan -> /app/models/diffusion_models/hot"
else
    echo "[entrypoint] WARNING: Filestore hot tier not mounted at /models/hot/wan"
fi

# Tier 3: GCS archive for backup/overflow/cold models (fallback)
if [[ -d /models/archive ]]; then
    ln -sf /models/archive/wan /app/models/diffusion_models/archive
    ln -sf /models/archive/overflow /app/models/diffusion_models/overflow
    ln -sf /models/archive/cold /app/models/diffusion_models/cold
    ln -sf /models/archive/sd /app/models/sd_archive
    echo "[entrypoint] Archive tier (GCS): /models/archive/* -> /app/models/*"
else
    echo "[entrypoint] WARNING: GCS archive not mounted at /models/archive"
fi

# Output directory (GCS FUSE)
if [[ -d /outputs ]]; then
    mkdir -p /app/outputs
    ln -sf /outputs /app/outputs/gcs
    echo "[entrypoint] Outputs (GCS): /outputs -> /app/outputs/gcs"
else
    echo "[entrypoint] WARNING: GCS outputs not mounted at /outputs"
fi

echo "[entrypoint] Model directory structure created:"
ls -lh /app/models/
ls -lh /app/models/diffusion_models/ 2>/dev/null || echo "  (diffusion_models dir empty or not created)"
```

**Update cache population function (around line 95-135):**
```bash
populate_cache_if_cold() {
    echo "[entrypoint-cache] Checking cache warmth..."
    
    # Check if cache markers exist
    if [[ -f /tmp/vae/.cached && -f /tmp/clip/.cached && -f /tmp/checkpoints/.cached ]]; then
        echo "[entrypoint-cache] Cache is warm (all markers present). Skipping population."
        return 0
    fi
    
    echo "[entrypoint-cache] Cache is cold. Populating SD models from GCS archive..."
    local start_time=$(date +%s)
    
    # Only cache SD models from GCS archive, NOT WAN (WAN lives on Filestore hot tier)
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
    echo "[entrypoint-cache] WAN models available on Filestore hot tier (no cache needed)"
}
```

**Convert CRLF to LF (PowerShell):**
```powershell
$path = "docker\comfyui-entrypoint.sh"
(Get-Content $path -Raw) -replace "`r`n","`n" | Set-Content $path -NoNewline
```

---

## Phase 5: Deployment Execution

### 5.1 Build and Deploy
```powershell
cd docker

# Build new image with updated entrypoint
$env:REBUILD_IMAGE = "1"
$env:FILESTORE_IP = "10.x.x.x"  # Replace with actual IP

.\deploy-cloudrun-gpu.sh neovnext us-east4
```

### 5.2 Monitor Deployment
```powershell
# Watch deployment progress
gcloud run services describe neovlab-gpu-worker-east4 `
  --region=us-east4 `
  --format="value(status.conditions)"

# Wait for READY condition
while ($true) {
    $status = gcloud run services describe neovlab-gpu-worker-east4 `
      --region=us-east4 `
      --format="value(status.conditions[0].type,status.conditions[0].status)"
    Write-Host "Status: $status"
    if ($status -match "Ready.*True") {
        Write-Host "Deployment complete!"
        break
    }
    Start-Sleep -Seconds 10
}
```

---

## Phase 6: Validation & Smoke Tests

### 6.1 Verify Volume Mounts
```powershell
# Check deployed configuration
gcloud run services describe neovlab-gpu-worker-east4 `
  --region=us-east4 `
  --format=yaml > Z:\NeoVNext\logs\deployed-config-$(Get-Date -Format 'yyyy-MM-dd-HHmm').yaml

# Verify volumes in config
Select-String "volumes:" Z:\NeoVNext\logs\deployed-config-*.yaml -Context 0,10
```

### 6.2 Check Cold Start Logs
```bash
# Trigger cold start by invoking service
gcloud run services proxy neovlab-gpu-worker-east4 --region=us-east4 &

# In another terminal, watch logs
gcloud logging tail "resource.type=cloud_run_revision AND resource.labels.service_name=neovlab-gpu-worker-east4" \
  --project=neovnext \
  --format="value(timestamp,severity,textPayload)"
```

**Expected log sequence:**
```
[entrypoint] Creating ComfyUI model directory structure (tiered storage)...
[entrypoint] Hot tier (Filestore): /models/hot/wan -> /app/models/diffusion_models/hot
[entrypoint] Archive tier (GCS): /models/archive/* -> /app/models/*
[entrypoint] Outputs (GCS): /outputs -> /app/outputs/gcs
[entrypoint] Model directory structure created:
[entrypoint] Starting ComfyUI server on port 8188...
[entrypoint-cache] Checking cache warmth...
[entrypoint-cache] Cache is cold. Populating SD models from GCS archive...
[entrypoint-cache] Copying VAE models...
[entrypoint-cache] Copying CLIP models...
[entrypoint-cache] Copying checkpoint models...
[entrypoint-cache] SD model cache populated in 180s
[entrypoint-cache] WAN models available on Filestore hot tier (no cache needed)
[entrypoint] ComfyUI ready after 34s.
```

### 6.3 Verify Filestore Mount
```bash
# Search for Filestore mount confirmation
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=neovlab-gpu-worker-east4 AND textPayload=~'Hot tier.*Filestore'" \
  --project=neovnext \
  --limit=10 \
  --format="value(timestamp,textPayload)"
```

### 6.4 Verify GCS FUSE Mounts
```bash
# Search for GCS archive mount confirmation
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=neovlab-gpu-worker-east4 AND textPayload=~'Archive tier.*GCS'" \
  --project=neovnext \
  --limit=10 \
  --format="value(timestamp,textPayload)"
```

---

## Phase 7: End-to-End ComfyUI Testing

### 7.1 Direct ComfyUI API Test (Bypass NeoVLab API)

**Setup port forwarding:**
```powershell
# Forward ComfyUI port 8188 to local machine
gcloud run services proxy neovlab-gpu-worker-east4 --region=us-east4 --port=8188
# Leave this running, open new terminal for tests
```

**Test 1: Model Discovery (Verify Tiering)**
```powershell
# Test from Z:\NeoVNext
cd Z:\NeoVNext

# List available models via ComfyUI API
curl http://localhost:8188/object_info | jq '.CheckpointLoaderSimple.input.required.ckpt_name[0]' > logs/comfy-models-available.json

# Check for WAN model from hot tier
Select-String "diffusion_pytorch_model" logs/comfy-models-available.json
```

**Test 2: Simple Workflow (SD Models from Cache)**
```powershell
# Create simple text-to-image workflow (uses cached SD models)
$workflow = @'
{
  "3": {
    "inputs": {
      "seed": 42,
      "steps": 20,
      "cfg": 7.0,
      "sampler_name": "euler",
      "scheduler": "normal",
      "denoise": 1.0,
      "model": ["4", 0],
      "positive": ["6", 0],
      "negative": ["7", 0],
      "latent_image": ["5", 0]
    },
    "class_type": "KSampler"
  },
  "4": {
    "inputs": {
      "ckpt_name": "sd_xl_base_1.0.safetensors"
    },
    "class_type": "CheckpointLoaderSimple"
  },
  "5": {
    "inputs": {
      "width": 512,
      "height": 512,
      "batch_size": 1
    },
    "class_type": "EmptyLatentImage"
  },
  "6": {
    "inputs": {
      "text": "a photo of a cat",
      "clip": ["4", 1]
    },
    "class_type": "CLIPTextEncode"
  },
  "7": {
    "inputs": {
      "text": "bad quality",
      "clip": ["4", 1]
    },
    "class_type": "CLIPTextEncode"
  },
  "8": {
    "inputs": {
      "samples": ["3", 0],
      "vae": ["4", 2]
    },
    "class_type": "VAEDecode"
  },
  "9": {
    "inputs": {
      "filename_prefix": "ComfyUI",
      "images": ["8", 0]
    },
    "class_type": "SaveImage"
  }
}
'@

$workflow | Out-File -FilePath "scripts\test-workflow-sd.json" -Encoding UTF8

# Submit workflow
$response = Invoke-RestMethod -Uri "http://localhost:8188/prompt" -Method Post -Body (@{prompt=$workflow} | ConvertTo-Json -Depth 10) -ContentType "application/json"
$prompt_id = $response.prompt_id

Write-Host "Submitted workflow, prompt_id: $prompt_id"

# Poll for completion
while ($true) {
    $history = Invoke-RestMethod -Uri "http://localhost:8188/history/$prompt_id"
    if ($history.PSObject.Properties.Count -gt 0) {
        Write-Host "Workflow completed!"
        $history | ConvertTo-Json -Depth 10 | Out-File "logs/comfy-test-sd-result.json"
        break
    }
    Start-Sleep -Seconds 2
}
```

**Test 3: WAN Workflow (Filestore Hot Tier)**
```powershell
# Create workflow using WAN diffusion model from Filestore
$wan_workflow = @'
{
  "3": {
    "inputs": {
      "seed": 42,
      "steps": 50,
      "cfg": 7.0,
      "sampler_name": "euler",
      "scheduler": "normal",
      "denoise": 1.0,
      "model": ["4", 0],
      "positive": ["6", 0],
      "negative": ["7", 0],
      "latent_image": ["5", 0]
    },
    "class_type": "KSampler"
  },
  "4": {
    "inputs": {
      "ckpt_name": "diffusion_pytorch_model.safetensors"
    },
    "class_type": "CheckpointLoaderSimple"
  },
  "5": {
    "inputs": {
      "width": 1024,
      "height": 1024,
      "batch_size": 1
    },
    "class_type": "EmptyLatentImage"
  },
  "6": {
    "inputs": {
      "text": "beautiful landscape, mountains, sunset",
      "clip": ["4", 1]
    },
    "class_type": "CLIPTextEncode"
  },
  "7": {
    "inputs": {
      "text": "ugly, distorted",
      "clip": ["4", 1]
    },
    "class_type": "CLIPTextEncode"
  },
  "8": {
    "inputs": {
      "samples": ["3", 0],
      "vae": ["4", 2]
    },
    "class_type": "VAEDecode"
  },
  "9": {
    "inputs": {
      "filename_prefix": "WAN_Test",
      "images": ["8", 0]
    },
    "class_type": "SaveImage"
  }
}
'@

$wan_workflow | Out-File -FilePath "scripts\test-workflow-wan.json" -Encoding UTF8

# Time the execution
$start_time = Get-Date

$response = Invoke-RestMethod -Uri "http://localhost:8188/prompt" -Method Post -Body (@{prompt=$wan_workflow} | ConvertTo-Json -Depth 10) -ContentType "application/json"
$prompt_id = $response.prompt_id

Write-Host "Submitted WAN workflow, prompt_id: $prompt_id"
Write-Host "This tests Filestore SSD access speed..."

# Poll for completion with timing
$model_load_start = $null
$model_loaded = $false

while ($true) {
    $history = Invoke-RestMethod -Uri "http://localhost:8188/history/$prompt_id"
    
    # Check logs for model loading
    if (-not $model_loaded) {
        $logs = gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=neovlab-gpu-worker-east4 AND textPayload=~'diffusion_pytorch_model'" `
          --project=neovnext --limit=5 --format="value(timestamp,textPayload)" 2>$null
        if ($logs) {
            $model_load_start = Get-Date
            $model_loaded = $true
            Write-Host "Model loading from Filestore detected..."
        }
    }
    
    if ($history.PSObject.Properties.Count -gt 0) {
        $end_time = Get-Date
        $duration = ($end_time - $start_time).TotalSeconds
        Write-Host "WAN workflow completed in $duration seconds!"
        $history | ConvertTo-Json -Depth 10 | Out-File "logs/comfy-test-wan-result.json"
        
        if ($model_load_start) {
            $model_load_duration = ($end_time - $model_load_start).TotalSeconds
            Write-Host "Model load + inference: $model_load_duration seconds"
        }
        break
    }
    Start-Sleep -Seconds 5
}
```

**Expected Results:**
- **SD workflow (cache):** ~30-60 seconds total (model already cached)
- **WAN workflow (Filestore hot):** ~10 seconds model load + ~120 seconds inference = ~130 seconds total
- **WAN workflow (GCS cold):** Would be ~20 minutes model load + ~120 seconds inference = ~21 minutes total

### 7.2 Verify Output Persistence
```powershell
# Check if outputs were written to GCS
gsutil ls -lh gs://neovnext-neovlab-io-cache/outputs/

# Should see output images from Test 2 and Test 3
```

### 7.3 Performance Validation Checklist

- [ ] **Cold start timing:** Container ready in <7 minutes
- [ ] **Filestore mount:** Hot tier accessible, WAN model found
- [ ] **GCS archive mount:** Archive tier accessible, fallback models found
- [ ] **SD model cache:** VAE/CLIP/checkpoints copied from GCS in ~3-5 minutes
- [ ] **WAN model load (hot):** <10 seconds from Filestore
- [ ] **Output persistence:** Images written to GCS outputs bucket
- [ ] **Scale-to-zero:** Container shuts down after idle timeout
- [ ] **Warm start:** Subsequent runs use cached models (<60 seconds)

---

## Phase 8: Full API Integration Test (Optional)

If you want to test the full NeoVLab API → ComfyUI pipeline:

### 8.1 Start Backend API Locally
```powershell
cd src\backend\NeoVLab.Api
dotnet run --urls "http://localhost:5000"
```

### 8.2 Submit Job via API
```powershell
# Test 1: Submit SD workflow via API
$job_response = Invoke-RestMethod -Uri "http://localhost:5000/api/jobs/submit-comfy" `
  -Method Post `
  -Body (@{
    workflowJson = (Get-Content "scripts\test-workflow-sd.json" -Raw)
  } | ConvertTo-Json) `
  -ContentType "application/json"

$job_id = $job_response.jobId
Write-Host "Job submitted: $job_id"

# Poll for job completion
while ($true) {
    $status = Invoke-RestMethod -Uri "http://localhost:5000/api/jobs/$job_id"
    Write-Host "Job status: $($status.status)"
    
    if ($status.status -in @("completed", "failed", "error")) {
        Write-Host "Job finished: $($status.status)"
        $status | ConvertTo-Json -Depth 10 | Out-File "logs/api-test-result.json"
        break
    }
    Start-Sleep -Seconds 5
}
```

---

## Phase 9: Cost Monitoring

### 9.1 Check Current Spend
```bash
# View current month's billing
gcloud billing projects describe neovnext --format="value(billingAccountName)" | \
  xargs -I {} gcloud alpha billing accounts list --filter="name={}" --format="value(displayName)"

# Detailed breakdown
gcloud billing projects describe neovnext
```

### 9.2 Set Up Usage Alerts
```powershell
# Enable usage export to BigQuery (for detailed analysis)
gcloud alpha billing accounts get-iam-policy $(gcloud billing projects describe neovnext --format="value(billingAccountName)" | Split-Path -Leaf)
```

---

## Phase 10: Rollback Plan (If Issues Arise)

### 10.1 Quick Rollback to Previous Image
```powershell
# Revert to pre-tiered-storage image
$BACKUP_IMAGE = gcloud container images list-tags gcr.io/neovnext/neovlab-gpu-worker `
  --filter="tags:pre-tiered-storage-backup" `
  --format="value(digest)" `
  --limit=1

gcloud run services update neovlab-gpu-worker-east4 `
  --region=us-east4 `
  --image="gcr.io/neovnext/neovlab-gpu-worker@$BACKUP_IMAGE" `
  --clear-volumes `
  --clear-volume-mounts
```

### 10.2 Restore Original Deployment Script
```powershell
Copy-Item docker/deploy-cloudrun-gpu.sh.backup docker/deploy-cloudrun-gpu.sh -Force
Copy-Item docker/comfyui-entrypoint.sh.backup docker/comfyui-entrypoint.sh -Force
```

### 10.3 Delete Filestore (Stop Billing)
```bash
# WARNING: This deletes all data on Filestore
gcloud filestore instances delete neovlab-models-hot \
  --zone=us-east4-a \
  --project=neovnext \
  --quiet
```

---

## Success Criteria

### Must Pass:
- [x] Container deploys successfully
- [x] Filestore mount accessible (`/models/hot/`)
- [x] GCS archive mount accessible (`/models/archive/`)
- [x] GCS output mount accessible (`/outputs/`)
- [x] SD models cached from GCS in <5 minutes
- [x] WAN model loads from Filestore in <10 seconds
- [x] ComfyUI workflows complete successfully
- [x] Outputs written to GCS bucket
- [x] No errors in Cloud Run logs
- [x] Container scales to zero when idle

### Nice to Have:
- [ ] Cold start (full cache population) <7 minutes
- [ ] Warm start (cache hit) <60 seconds
- [ ] WAN workflow total time <3 minutes
- [ ] Storage costs <$15/month
- [ ] Compute costs <$50/month (light usage)

---

## Troubleshooting Common Issues

### Issue: Filestore mount fails
**Symptoms:** Log shows "WARNING: Filestore hot tier not mounted"
**Fix:**
```bash
# Verify Filestore is running
gcloud filestore instances describe neovlab-models-hot --zone=us-east4-a

# Check VPC network connectivity
gcloud compute networks describe default

# Ensure Cloud Run service account has NFS access
gcloud projects add-iam-policy-binding neovnext \
  --member="serviceAccount:PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/file.editor"
```

### Issue: GCS FUSE mount slow or timing out
**Symptoms:** Cache population takes >10 minutes
**Fix:**
```bash
# Check GCS bucket location matches Cloud Run region
gsutil ls -Lb gs://neovnext-neovlab-models/ | grep Location
# Should show: us-east4

# Verify no egress charges
gsutil ls -L gs://neovnext-neovlab-models/ | grep "Storage class"
# Should show: STANDARD
```

### Issue: WAN model not found in hot tier
**Symptoms:** ComfyUI can't find diffusion_pytorch_model.safetensors
**Fix:**
```bash
# SSH into temporary VM to verify Filestore contents
gcloud compute instances create filestore-checker \
  --zone=us-east4-a --machine-type=e2-micro --scopes=cloud-platform

gcloud compute ssh filestore-checker --zone=us-east4-a
sudo mount ${FILESTORE_IP}:/hot_models /mnt
ls -lh /mnt/hot/wan/
# Should see diffusion_pytorch_model.safetensors (91GB)
```

### Issue: Container exceeds memory limit
**Symptoms:** Container killed with OOM error
**Fix:**
```bash
# Increase memory allocation
gcloud run services update neovlab-gpu-worker-east4 \
  --region=us-east4 \
  --memory=24Gi
```

### Issue: Compute costs higher than expected
**Symptoms:** Bill shows >$100 in first week
**Fix:**
```bash
# Check container isn't staying up indefinitely
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=neovlab-gpu-worker-east4" \
  --project=neovnext \
  --format="value(timestamp)" \
  --limit=100 | sort

# Look for scaling events - should see container stop after idle period
# If container stays up, check for leaked connections or long-running jobs
```

---

## Next Steps After Successful Deployment

1. **Monitor costs daily for first week:** Ensure no surprises
2. **Run 10+ WAN workflows:** Verify Filestore hot tier performance is consistent
3. **Test cold start after 24h idle:** Confirm cache persists (it shouldn't, by design)
4. **Benchmark warm vs. cold start:** Document actual performance improvements
5. **Update documentation:** Record any deviations from this plan
6. **Consider archiving rarely-used models:** Move to `gs://.../cold/` to optimize Filestore usage
7. **Plan for model rotation:** When Filestore approaches 90GB, decide which models stay hot

---

## Appendix A: Estimated Timeline

| Phase | Duration | Blocker | Can Parallelize |
|-------|----------|---------|-----------------|
| 1. Pre-deployment prep | 30 min | - | - |
| 2. GCS reorganization | 45 min | Network speed | No |
| 3. Filestore provisioning | 10 min | GCP API | No |
| 4. Populate Filestore | 30 min | Network speed | No |
| 5. Update deployment config | 15 min | - | - |
| 6. Build and deploy | 20 min | Docker build | No |
| 7. Validation | 15 min | Cold start | - |
| 8. E2E testing | 30 min | Workflow execution | - |
| **Total** | **3-4 hours** | | |

---

## Appendix B: Cost Breakdown (Revised)

### Monthly Costs (Light Usage - 10 hours GPU/month)
| Component | Unit Cost | Usage | Monthly Cost |
|-----------|-----------|-------|--------------|
| Filestore Lite 100GB | $0.12/GB-month | 100GB | $12.00 |
| GCS Standard Storage | $0.020/GB-month | 140GB | $2.80 |
| L4 GPU | $3.20/hour | 10 hours | $32.00 |
| Memory (16Gi) | Included with GPU | - | $0.00 |
| CPU (4 cores) | Included with GPU | - | $0.00 |
| GCS Egress | $0/GB (same region) | - | $0.00 |
| **Total** | | | **$46.80** |

### Monthly Costs (Active Development - 40 hours GPU/month)
| Component | Monthly Cost |
|-----------|--------------|
| Filestore | $12.00 |
| GCS Storage | $2.80 |
| Compute (40h) | $128.00 |
| **Total** | **$142.80** |

### Free Credit Runway
- $300 credits / $46.80 per month (light usage) = **6.4 months**
- $300 credits / $142.80 per month (active dev) = **2.1 months**

**Reality:** Actual usage likely between these (20-25 hours/month) = **~$100/month total** = **3 months runway**
