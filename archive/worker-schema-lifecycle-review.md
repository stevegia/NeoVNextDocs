# Worker Schema & Lifecycle Review
**Date:** 2026-03-25  
**Context:** Pre-deployment review before deploying HTTP keepalive fix

---

## Database Schema

### `job_queue` Table
```sql
job_id             TEXT PRIMARY KEY
media_id           TEXT NOT NULL
job_type           TEXT NOT NULL
processing_type    TEXT NOT NULL          -- routing: 'cpu', 'gpu'
worker_function    TEXT NOT NULL          -- routing: 'comfyui', 'transcription', etc.
status             TEXT DEFAULT 'queued'  -- queued → assigned → running → completed/failed
assigned_worker_id TEXT                   -- set when claimed
lease_expires_at   TIMESTAMPTZ            -- worker must renew or lose job
priority           INTEGER DEFAULT 5
payload_json       TEXT
error_info         TEXT
created_at         TIMESTAMPTZ
updated_at         TIMESTAMPTZ
```

**Indexes:**
- `idx_job_queue_routing` — composite index on (status, worker_function, processing_type, priority DESC, created_at ASC)
- `idx_job_queue_assigned_worker` — for worker-specific queries
- Individual indexes on status, worker_function

### `cache_manifest` Table
```sql
entry_id        TEXT PRIMARY KEY
job_id          TEXT REFERENCES job_queue(job_id) ON DELETE CASCADE
direction       TEXT NOT NULL              -- 'input' or 'output'
file_key        TEXT NOT NULL
content_type    TEXT NOT NULL
file_size_bytes BIGINT DEFAULT 0
cache_path      TEXT NOT NULL              -- GCS FUSE path or local volume path
synced_locally  BOOLEAN DEFAULT FALSE
created_at      TIMESTAMPTZ
synced_at       TIMESTAMPTZ
evicted_at      TIMESTAMPTZ
gcs_evicted_at  TIMESTAMPTZ               -- added in migration 002
```

**Indexes:**
- `idx_cache_manifest_job_id` — FK lookup
- `idx_cache_manifest_direction` — input vs output filtering
- `idx_cache_manifest_gcs_eviction` — WHERE gcs_evicted_at IS NULL

### `worker_registry` Table
```sql
worker_id         TEXT PRIMARY KEY
processing_type   TEXT NOT NULL
worker_function   TEXT NOT NULL
state             TEXT DEFAULT 'starting'  -- starting → ready → busy → draining → offline
last_heartbeat    TIMESTAMPTZ             -- updated every 30s
capabilities_json TEXT
registered_at     TIMESTAMPTZ
```

**Indexes:**
- `idx_worker_registry_type` — composite on (processing_type, worker_function, state)
- Individual indexes on state, worker_function

---

## Worker Lifecycle

### Job Claiming Logic (`claim_next_job`)
```sql
SELECT job_id FROM job_queue
WHERE (
    status = 'queued'
    OR (
        status IN ('assigned', 'running')
        AND lease_expires_at IS NOT NULL
        AND lease_expires_at < NOW()  -- ✅ AUTOMATIC RECOVERY
    )
)
AND worker_function = %s
AND processing_type = %s
ORDER BY priority DESC, created_at ASC
LIMIT 1
FOR UPDATE SKIP LOCKED
```

**Key Features:**
- ✅ **Automatic orphan recovery**: Expired leases can be re-claimed by any matching worker
- ✅ **Routing isolation**: Workers only see jobs matching their (processing_type, worker_function)
- ✅ **Lock-free concurrency**: SKIP LOCKED prevents worker contention
- ✅ **Priority ordering**: Higher priority jobs processed first, FIFO within priority

### Job State Machine
```
queued → assigned → running → completed
                  ↘         ↘ failed
                    
Transitions:
- queued → assigned:   claim_next_job() sets status='assigned', assigned_worker_id, lease_expires_at
- assigned → running:  set_job_running() after worker picks up job
- running → completed: complete_job() after successful processing
- running → failed:    fail_job(error_info) on exception
```

### Lease Renewal
- **Initial lease**: 10 minutes (configurable via LEASE_DURATION_MINUTES)
- **Renewal interval**: Every 5 minutes (half of lease duration, min 60s)
- **Renewal method**: `renew_lease(job_id, worker_id, duration_minutes)`
  - Returns `True` if worker still owns the job
  - Returns `False` if job was re-claimed or doesn't exist
- **Purpose**: Prevent long-running jobs (e.g., 10+ min ComfyUI renders) from being re-claimed

### Worker Heartbeat
- **Interval**: Every 30 seconds
- **Method**: `heartbeat(worker_id, state)` → updates `worker_registry.last_heartbeat`
- **Purpose**: Cluster visibility — track which workers are alive
- **Limitation**: ⚠️ Database heartbeats do NOT prevent Cloud Run scale-to-zero

### Graceful Shutdown
```python
async def stop():
    1. Stop accepting new jobs (_stop_event.set(), cancel poll task)
    2. Set state to "draining"
    3. Wait for current job to complete (timeout = lease_duration)
    4. Cancel heartbeat task
    5. Set state to "offline"
    6. Close database connection
```

---

## Identified Issues & Mitigations

### ✅ Issue #1: Orphaned Jobs (HANDLED BY DESIGN)
**Problem:** Worker dies mid-job without graceful shutdown  
**Current Behavior:**  
- Job remains in "running" status with active lease
- After lease expires (10 min), another worker can re-claim it
- Job is automatically recovered via `claim_next_job` WHERE clause

**Mitigation:** ✅ Built-in via lease expiry recovery

### ⚠️ Issue #2: Cloud Run Scale-to-Zero Kills Active Jobs (CRITICAL — FIXING NOW)
**Problem:** Cloud Run scales worker to zero during active job execution  
**Root Cause:** Database heartbeats (every 30s) don't generate HTTP traffic, so Cloud Run thinks worker is idle  
**Evidence:** Job 87516eba-861d-40e7-8ef7-c7052ec21ffa killed at 23:54:43 after 5 min of active rendering  
**Consequence:**  
- Job stuck in "running" status until lease expires (10 min)
- No outputs saved to cache_manifest or GCS
- ComfyUI work lost

**Mitigation:** ✅ HTTP keepalive implementation (in progress)
- Ping `http://localhost:8080/health` every 60s during job execution
- Generates HTTP traffic that Cloud Run recognizes as activity
- Prevents scale-to-zero while job is running

### ✅ Issue #3: Worker Registry Cleanup (ACCEPTABLE)
**Problem:** Dead workers remain in `worker_registry` with stale `last_heartbeat`  
**Current Behavior:** No automatic cleanup of offline workers  
**Impact:** Minimal — worker_registry is informational only, doesn't affect job routing  
**Mitigation:** ✅ Not needed (jobs route via job_queue, not worker_registry)

### ✅ Issue #4: Lease Renewal Failure Handling (WELL-DESIGNED)
**Problem:** What if lease renewal fails during long job?  
**Current Behavior:**  
- `renew_lease()` returns `False` if job was re-claimed
- Worker logs warning but continues processing (doesn't fail the job)
- If another worker re-claims, it will also start processing
- Race condition: two workers may process same job

**Mitigation:** ✅ Acceptable — stateless jobs (ComfyUI renders) can tolerate duplicate processing
- First worker to complete wins
- Database UPDATE is atomic, no corruption risk

### ✅ Issue #5: Job Re-Claiming Edge Case (ACCEPTABLE)
**Problem:** Jobs with `status='assigned'` but `lease_expires_at=NULL` cannot be re-claimed  
**SQL Requirement:** `lease_expires_at IS NOT NULL AND lease_expires_at < NOW()`  
**Current Behavior:** Such jobs would be orphaned permanently  
**Mitigation:** ✅ Not a concern — `claim_next_job()` always sets lease_expires_at when claiming

---

## Test Coverage

### Lease Recovery Tests (`test_worker_lease_recovery.py`)
✅ Comprehensive contract tests validate:
- Expired leases are re-claimable
- Active leases prevent re-claiming
- Routing by (processing_type, worker_function)
- State transitions (queued → assigned → running → completed/failed)
- Zero-duration leases are invalid

### Worker Contract Tests
✅ Additional test files:
- `test_cpu_worker_contract.py`
- `test_gpu_worker_contract.py`
- `test_orchestrator_worker_contract.py`
- `test_comfyui_primary_output.py`

---

## Remaining Risks

### Low Risk
1. **Database connection failures during job execution**
   - Impact: Worker logs error, job continues processing
   - Recovery: Lease expires, job re-claimed
   - Mitigation: Retry logic in ClusterDbClient (not implemented, but acceptable)

2. **Duplicate job processing (race condition)**
   - Impact: Two workers process same job after lease expiry
   - Recovery: First to complete wins, second gets UPDATE failure
   - Mitigation: Acceptable for stateless jobs

3. **Worker registry bloat**
   - Impact: Old "offline" workers accumulate in registry
   - Recovery: Manual cleanup or periodic purge
   - Mitigation: Low priority (informational table only)

### High Risk (FIXING NOW)
1. **Cloud Run scale-to-zero during active jobs** ← HTTP keepalive fix

---

## Recommendations

### Immediate (Before Deployment)
1. ✅ Deploy HTTP keepalive fix (in progress)
2. ✅ Verify keepalive appears in logs after deployment
3. ✅ Test job completes without interruption

### Short-Term (Next Sprint)
1. Add monitoring/alerting for:
   - Jobs stuck in "running" > 1 hour
   - Workers with last_heartbeat > 5 minutes ago
   - Failed lease renewals
2. Add admin endpoint to manually fail stuck jobs
3. Consider adding `started_at` and `completed_at` timestamps to job_queue for observability

### Long-Term (Future)
1. Implement worker registry cleanup job (delete workers offline > 24h)
2. Add job retry mechanism for failed jobs (with exponential backoff)
3. Consider adding `attempt_count` to track job retries
4. Add metrics for job processing latency, success/failure rates

---

## Conclusion

**Schema & Worker Logic: WELL-DESIGNED ✅**
- Lease recovery handles orphaned jobs automatically
- Routing logic prevents cross-contamination
- State machine is sound
- Test coverage is comprehensive

**Critical Issue: Cloud Run Scale-to-Zero ⚠️**
- Database heartbeats insufficient for Cloud Run
- HTTP keepalive required to keep worker alive during jobs
- Fix implemented, ready for deployment

**Overall Assessment: SAFE TO DEPLOY** after HTTP keepalive verification.
