# GPU Worker Debug Deployment

## Changes Made

Added comprehensive debug logging to track worker startup:

1. **host_gpu.py**: Added `logging.basicConfig()` to ensure logs go to stdout
2. **host_gpu.py**: Added log before app creation to confirm module load
3. **pull_worker.py**: Added `[DEBUG]` logs at each step of `start()` method

## Deploy the Updated Worker

```powershell
# Navigate to project root
cd Z:\NeoVNext

# Set variables (adjust if needed)
$PROJECT_ID = "neovnext"
$REGION = "us-east4"
$SERVICE_NAME = "neovlab-gpu-worker-east4"
$IMAGE_NAME = "neovlab-gpu-worker"

# Build the GPU worker image
docker build -f docker/worker-gpu.Dockerfile -t gcr.io/$PROJECT_ID/$IMAGE_NAME .latest .

# Push to GCR
docker push gcr.io/$PROJECT_ID/$IMAGE_NAME:latest

# Update the Cloud Run service with the new image
gcloud run services update $SERVICE_NAME `
    --region=$REGION `
    --image=gcr.io/$PROJECT_ID/$IMAGE_NAME:latest

# OR use the deploy script (recommended if it sets all the correct env vars)
cd docker
bash ./deploy-cloudrun-gpu.sh
```

## Check the New Logs

After deployment completes, check logs:

```powershell
# Watch logs in real-time
gcloud run services logs tail neovlab-gpu-worker-east4 --region=us-east4

# Or get recent logs
gcloud run services logs read neovlab-gpu-worker-east4 --region=us-east4 --limit=50 | Select-String -Pattern '\[DEBUG\]|Worker.*started|ERROR'
```

## What to Look For

The debug logs will show:
- `GPU worker host module loaded` - confirms module import
- `Creating pull-worker app` - confirms app creation
- `[DEBUG] PullWorkerRuntime.start() called` - confirms startup event fired
- `[DEBUG] io_cache_dir created` - confirms first step succeeds
- `[DEBUG] DATABASE_URL configured: YES` - confirms env var is set
- `[DEBUG] Creating ClusterDbClient...` - before DB client creation
- `[DEBUG] ClusterDbClient created successfully` - DB client works
- `[DEBUG] Loading handler...` - before handler load
- `[DEBUG] Handler loaded successfully` - handler loaded
- `[DEBUG] Registering worker in database...` - before DB registration
- `[DEBUG] Worker registered successfully` - DB write succeeds
- `[DEBUG] State set to 'ready'` - state transition works
- `Worker {id} started in pull mode for gpu/comfyui` - FULL SUCCESS

**Where the logs stop will tell us exactly what's failing.**
