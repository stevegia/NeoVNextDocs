# Postmortem: Cloud Run Scale-to-Zero Killing Active Jobs

**Date:** 2026-03-26  
**Severity:** P0 — Critical, jobs interrupted mid-execution causing work loss and wasted GPU costs  
**Duration:** March 25-26, 2026 (~12 hours between discovery and fix)  
**Impact:** All long-running ComfyUI jobs (5+ minutes) were terminated by Cloud Run scale-to-zero logic before completion, resulting in 0 successful job completions and wasted GPU compute time.

---

## Timeline

**2026-03-25 23:49:40 UTC** — Job `87516eba` claimed by worker `gpu-comfyui-east4`, status set to "running"  
**2026-03-25 23:54:43 UTC** — Worker received SIGTERM from Cloud Run, initiated graceful shutdown  
**2026-03-26 00:04:39 UTC** — Job lease expired, status still "running", 0 artifacts saved  

**2026-03-26 00:20:00 UTC** — Issue discovered via database query: job stuck in "running" status  
**2026-03-26 00:25:00 UTC** — Verified GCS storage empty for job output directory  
**2026-03-26 00:30:00 UTC** — Log analysis revealed "INFO: Shutting down" at 23:54:43 (5 min after job start)  
**2026-03-26 00:35:00 UTC** — Root cause identified: Cloud Run scale-to-zero triggered during active job  

**2026-03-26 00:45:00 UTC** — First mitigation attempt: Worker-side HTTP keepalive to localhost:8080/health  
**2026-03-26 00:54:14 UTC** — Revision 00052-fll deployed with localhost keepalive  
**2026-03-26 00:59:21 UTC** — **Mitigation failed:** Worker still terminated after 5 minutes despite keepalive pings  

**2026-03-26 01:05:00 UTC** — **Critical insight:** Keepalive pings from 127.0.0.1 (localhost) do NOT count as external HTTP traffic for Cloud Run's scale-to-zero logic  
**2026-03-26 01:15:00 UTC** — Solution designed: External URL discovery + smart keepalive with idle timeout  
**2026-03-26 01:30:00 UTC** — Revision 00053-l26 deployed with external URL keepalive + 10-min idle timeout  
**2026-03-26 01:35:00 UTC** — **Resolution confirmed:** New keepalive mechanism operational  

---

## Root Causes

### 1. Cloud Run Scale-to-Zero Triggers on HTTP Inactivity Only

**The Assumption:** Database heartbeats (updating `worker_registry.last_heartbeat` every 30 seconds) would keep the worker instance alive during job execution.

**The Reality:** Cloud Run's scale-to-zero logic **only counts HTTP requests to the container's ingress port** as activity. Background processes, database connections, and internal asyncio tasks are invisible to Cloud Run's autoscaler.

**Result:** After ~5 minutes of no HTTP traffic, Cloud Run terminated the instance mid-job execution.

### 2. Localhost Keepalive Ineffective

**First Mitigation Attempt:** Added `_job_keepalive_loop()` method that pinged `http://localhost:8080/health` every 60 seconds during job execution.

**Why It Failed:** Cloud Run distinguishes between:
- **External traffic:** HTTP requests from outside the container (counted for scale-to-zero decisions)
- **Internal traffic:** Localhost/127.0.0.1 requests within the container (ignored)

Worker self-pinging via localhost generated HTTP requests visible in logs but **did not prevent scale-to-zero**.

### 3. No Idle Grace Period

The initial keepalive design ran only during active job execution. If the worker completed a job and immediately received another, it would scale to zero between jobs, causing:
- Unnecessary cold starts (4+ min cache population)
- Wasted time re-loading models already in ephemeral storage
- Poor user experience for back-to-back requests

---

## The Fix: Smart External Keepalive with Idle Timeout

### Implementation

**External URL Discovery:**
Workers query Cloud Run's metadata server to discover their own external URL:

```python
def _get_external_url(self) -> str:
    # Query metadata.google.internal for service name and region
    # Construct: https://{service}-{hash}.{region}.run.app
    # Falls back to localhost for local dev
```

**Continuous Keepalive Loop:**
Replaces per-job keepalive with worker-level keepalive that runs continuously:

```python
async def _worker_keepalive_loop(self) -> None:
    while not self._stop_event.is_set():
        await asyncio.sleep(60)
        
        # Check idle timeout
        if self._last_job_completed_at is not None:
            idle_duration = datetime.utcnow() - self._last_job_completed_at
            if idle_duration > timedelta(minutes=self.idle_timeout_minutes):
                logger.info("Worker idle for %s. Stopping keepalive.", idle_duration)
                break
        
        # Ping external URL (not localhost!)
        req = urllib.request.Request(
            f"{self._get_external_url()}/health",
            headers={"Authorization": f"Bearer {self.worker_token}"}
        )
        await asyncio.to_thread(lambda: urllib.request.urlopen(req, timeout=5).read())
```

**Idle Timeout Logic:**
- Job starts → `_last_job_completed_at = None` (reset idle timer)
- Job completes → `_last_job_completed_at = datetime.utcnow()` (mark completion time)
- Keepalive checks: `if idle > threshold: break` (stop pinging, allow scale-to-zero)

**Configuration:**
```bash
--set-env-vars="WORKER_IDLE_TIMEOUT_MINUTES=10"  # Default: 10 min
```

### Why This Works

| Aspect | Localhost Keepalive (Failed) | External URL Keepalive (Success) |
|--------|------------------------------|----------------------------------|
| Traffic source | 127.0.0.1 (internal) | External Cloud Run URL |
| Counted by Cloud Run? | ❌ No | ✅ Yes |
| Prevents scale-to-zero? | ❌ No | ✅ Yes |
| Runs when? | Only during job execution | Continuously until idle timeout |
| Grace period? | ❌ None | ✅ Configurable (default 10 min) |
| Cost when idle? | $0 (scales immediately) | ~$0.53/hour for grace period only |

**Cost Savings vs. min-instances=1:**
- `min-instances=1`: $3.20/hour × 24 hours = **$76.80/day**
- Smart keepalive (10 min idle): $3.20/hour × (job duration + 10 min grace) = **~$3-5/day** for typical workload

---

## Verification

### Log Signatures

**Expected on startup:**
```
Worker keepalive started: pinging https://neovlab-gpu-worker-east4-....run.app/health every 60s (idle timeout: 10 min)
```

**During execution (every 60s):**
```
Worker keepalive ping succeeded (next ping in 60s).
```

**After job completion:**
```
Job completed at 2026-03-26T01:45:23. Idle timeout will trigger in 10 minutes.
```

**After idle timeout:**
```
Worker idle for 0:10:03 (> 10 min threshold). Stopping keepalive to allow scale-to-zero.
```

### Database Validation

Jobs no longer stuck in "running" status:
```sql
SELECT job_id, status, assigned_worker_id, created_at, lease_expires_at
FROM job_queue
WHERE status = 'running' AND lease_expires_at < NOW();
```

Should return 0 rows after fix deployment.

---

## Lessons Learned

### 1. Cloud Platform Assumptions Are Dangerous

**Assumption:** "Heartbeats keep the worker alive"  
**Reality:** Only HTTP ingress traffic matters for Cloud Run autoscaling

**Action:** Document infrastructure behavior explicitly. Added "Worker Keepalive and Scale-to-Zero" section to [runbook-gcp-gpu.md](./runbook-gcp-gpu.md).

### 2. Localhost vs. External Traffic

**Mistake:** Treating localhost HTTP requests as equivalent to external traffic  
**Learning:** Cloud Run distinguishes traffic sources for autoscaling decisions

**Action:** Added to debugging memory: "Cloud Run only counts external HTTP requests for scale-to-zero, not localhost traffic."

### 3. Graceful Degradation > Instant Optimization

**Original design:** Scale to zero immediately after job completion (optimal cost)  
**Better design:** Grace period allows batched jobs to avoid cold starts (optimal UX + cost balance)

**Action:** Made idle timeout configurable (`WORKER_IDLE_TIMEOUT_MINUTES`) so it can be tuned per workload.

### 4. Incremental Validation Crucial

**What worked:**
1. First fix attempt (localhost keepalive) deployed quickly
2. Monitored logs, discovered it didn't work
3. Diagnosed *why* (localhost vs. external traffic)
4. Iterated to correct solution

**What would have failed:**
- Designing a "perfect" solution upfront without testing assumptions
- Committing to `min-instances=1` without exploring alternatives

**Action:** Incremental deployment and log-driven debugging saved ~$70/day in unnecessary min-instances costs.

---

## Action Items

- [x] Implement external URL keepalive with idle timeout
- [x] Deploy to Cloud Run GPU worker (revision 00053-l26)
- [x] Update documentation ([runbook-gcp-gpu.md](./runbook-gcp-gpu.md))
- [x] Add troubleshooting entries for keepalive issues
- [ ] Add Prometheus metrics for keepalive success/failure rate
- [ ] Add alerting if jobs stuck in "running" status > lease duration
- [ ] Consider Cloud Run Jobs for true batch workloads (future architecture iteration)

---

## Related Documentation

- [GCP Cloud Run GPU Runbook](./runbook-gcp-gpu.md) — Section 2.4 covers keepalive architecture
- [Worker Schema & Lifecycle Review](./worker-schema-lifecycle-review.md) — Analysis of worker state machine and lease recovery
- [Copilot Instructions: GCP CLI Patterns](.github/copilot-instructions.md) — Operational debugging patterns

---

## Final Notes

This incident highlighted a fundamental mismatch between pull-model worker architecture (which uses database polling) and Cloud Run's HTTP-centric autoscaling. The external URL keepalive is an effective mitigation, but the **architecturally correct long-term solution** may be migrating to **Cloud Run Jobs** for batch GPU workloads, where execution is tied to job lifecycle rather than HTTP traffic patterns.

For now, the smart keepalive provides optimal cost/reliability balance: workers stay alive during jobs and brief idle periods, then scale to zero to save costs. This is significantly better than `min-instances=1` ($76/day) while avoiding job interruptions.
