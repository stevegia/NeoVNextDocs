# GPU Worker Recovery - Testing Instructions

## Worker Status: ✅ RUNNING

The worker is now successfully starting and polling for jobs. Debug logs confirm:
- Database connection: ✅ Working
- Handler loading: ✅ Working  
- Worker registration: ✅ Working
- Polling loop: ✅ Started

## What Fixed It

**Root Cause**: Python logging from thread pool workers wasn't being flushed to stdout, so Cloud Run never captured the logs. This made it look like the worker was failing, when it was actually running blind.

**Fix Applied**:
1. Added `sys.stdout.flush()` after every log statement in `start()`
2. Added exception handling in FastAPI `@app.on_event("startup")` 
3. Added explicit `logging.basicConfig()` in `host_gpu.py` with `force=True`

## Test the Worker

### 1. Submit a test job from the frontend:
- Go to your NeoVNext UI
- Create a new ComfyUI job
- Job should move from `queued` → `assigned` → `running` → `completed`

### 2. Watch the worker logs:
```powershell
# Watch for job processing
gcloud run services logs read neovlab-gpu-worker-east4 --region=us-east4 --limit=50 | Select-String -Pattern 'job_id|status|Claimed|Completed'
```

### 3. Check the database (optional):
Run the debug SQL queries in `scripts/debug-worker-job-match.sql` to verify:
- Worker appears in `worker_registry` table
- Jobs are being claimed and updated

## If Jobs Still Stuck

If jobs remain in `queued` status:

1. **Check the job record values**:
   ```sql
   SELECT job_id, status, worker_function, processing_type 
   FROM job_queue 
   WHERE job_id = '<your-job-id>';
   ```
   Make sure: `worker_function='comfyui'` and `processing_type='gpu'`

2. **Check worker registration**:
   ```sql
   SELECT * FROM worker_registry WHERE worker_id = 'gpu-comfyui-east4';
   ```
   Should show: `state='ready'`, recent `last_heartbeat`

3. **Check for lease conflicts**:
   ```sql
   SELECT job_id, status, assigned_worker_id, lease_expires_at
   FROM job_queue
   WHERE status IN ('assigned', 'running')
   AND lease_expires_at > NOW();
   ```
   Old leases block new claims — they expire after 30 minutes

## Next Steps

Submit a test job and share the results. If it works, the issue is fully resolved! 🎉
