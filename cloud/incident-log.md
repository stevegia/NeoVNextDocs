# Cloud Incident Log

Incidents are recorded in reverse chronological order. Each entry documents what was observed, investigated, root cause hypothesis, and any code-layer changes required.

---

## INC-002 — Go Supervisor Keepalive Auth Failure + Orphaned Job on SIGTERM

**Date:** 2026-03-27
**Severity:** P1 — Active job lost, job record orphaned in database
**Service:** neovlab-gpu-worker-east4 (revision 00017-7wc)
**Job ID:** b7945fdb-6c83-4616-86a3-858aa648766a
**Worker function:** comfyui
**Status in DB at time of investigation:** `running` (orphaned — no worker alive)

---

### What Was Observed

Job `b7945fdb` was submitted and picked up by the GPU worker. The container cold-started, ComfyUI initialized successfully, and model loading began (WAN 2.1, 14B fp8 scaled). Approximately 4 minutes into execution the container received SIGTERM and exited. The job record in `job_queue` was never updated from `running` to `failed`.

Database state at investigation time (19:40 UTC):
- `status`: `running`
- `assigned_worker_id`: `gpu-worker-east4`
- `updated_at`: `2026-03-27 19:31:57 UTC` (last lease renewal)
- `error_info`: NULL

No active Cloud Run instances. The container is gone.

---

### Timeline

| Time (UTC) | Event |
|---|---|
| 18:35:51 | Job created in `job_queue`, `status = running`, assigned to `gpu-worker-east4` |
| 19:27:18 | Container TCP probe succeeded — Cloud Run instance started |
| 19:27:19 | ComfyUI loaded: `13917.38 MB usable` — L4 GPU confirmed active |
| 19:27:39–19:28:21 | WAN 2.1 UNET models loading (fp8 quantized, 14B) |
| 19:28:54 | First keepalive ping: HTTP **403** — unauthenticated request rejected |
| 19:29:54–19:31:54 | Every keepalive tick returns 403 — keepalive loop is broken |
| 19:31:57 | Lease renewed in DB (last successful DB operation) |
| 19:32:20 | `shutdown signal received: terminated` — Cloud Run SIGTERM delivered |
| 19:32:20 | `job failed: context canceled` logged — but DB write did **not** complete |
| 19:32:22 | `GPU Supervisor stopped` — container exits |

---

### What Was Investigated

**Service status:** `neovlab-gpu-worker-east4` is `Ready`. Revision `00018-fcl` had been created (new deployment) but was not yet serving traffic — `00017-7wc` was serving 100%. The SIGTERM at 19:32:20 was consistent with a revision rollout draining the old revision.

**Keepalive 403s:** The supervisor's keepalive loop is pinging the Cloud Run service URL (the external ingress) but attaching no identity token. The Cloud Run service requires IAM authentication. Every ping returns 403 for approximately 4 minutes before SIGTERM arrives. This means the keepalive — which was implemented to prevent scale-to-zero — is silently failing on every tick.

**Prior art:** This is a recurrence of the pattern documented in `postmortem-scale-to-zero-kills-jobs.md`. That incident established that:
1. Localhost keepalive does not count for Cloud Run autoscaling.
2. External URL keepalive must include a valid Bearer token.

The Python pull_worker had this solved. The Go supervisor (Go-Gasmoth's code) regressed on the auth token when the keepalive was ported.

**Orphaned record:** The Go supervisor logs `job failed: context canceled` at shutdown, then proceeds to stop all handlers. The DB write for the status update appears to be racing with context cancellation — the context is already canceled when the write is attempted, so the write is dropped.

---

### Root Causes

**RC-1: Keepalive missing OIDC identity token (Go supervisor regression)**

The Go supervisor's keepalive pings `https://neovlab-gpu-worker-east4-....run.app` (the Cloud Run ingress URL) with no Authorization header. Cloud Run IAM rejects it with 403. The keepalive loop continues ticking and logging 403 without escalating or failing. This means the mechanism intended to prevent scale-to-zero is doing nothing.

The Python predecessor attached `worker_token` as a Bearer header. The Go port dropped this.

**RC-2: Job status not written on SIGTERM shutdown**

When SIGTERM is received, the supervisor cancels the root context. The job failure handler logs the error (`job failed: context canceled`) but the DB write to set `status = failed` is attempted on the already-canceled context — so the write is dropped and the record stays `running` forever. This is an orphan-creation bug.

---

### Code Changes Required

**Route to: Go-Gasmoth** (Go supervisor and Python handlers in `src/gpu-supervisor/`)

**Change 1: Keepalive OIDC token (HIGH priority)**

The Go supervisor's keepalive loop must attach a GCP OIDC identity token to outbound pings. The token target audience should be the Cloud Run service URL. In Go, the canonical approach is:

```go
// Use metadata.google.internal to fetch an identity token
// audience = the Cloud Run service URL (not localhost)
// GET http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/identity?audience=<URL>
// Attach result as: Authorization: Bearer <token>
```

Token should be refreshed before expiry (identity tokens have a 1-hour TTL — refresh every 45 minutes is safe).

The prior Python implementation is in `postmortem-scale-to-zero-kills-jobs.md` lines 78-96 for reference.

**Change 2: Graceful shutdown must use a background context for the status write (HIGH priority)**

In the SIGTERM shutdown path, the root context is canceled before the DB write completes. The job status update (`status = failed`, `error = "SIGTERM received during execution"`) must be performed on a **new background context with a short deadline** (e.g., 10 seconds), not the already-canceled root context. This ensures the DB write completes before the process exits.

Pattern:
```go
// In shutdown handler, after detecting context cancellation:
shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
db.UpdateJobStatus(shutdownCtx, jobID, "failed", "SIGTERM received during execution")
```

---

### Immediate Mitigation (Infrastructure)

The orphaned job must be manually reset. Run against Cloud SQL `cluster` database:

```sql
UPDATE job_queue
SET status = 'pending',
    assigned_worker_id = NULL,
    started_at = NULL,
    updated_at = NOW()
WHERE job_id = 'b7945fdb-6c83-4616-86a3-858aa648766a'
  AND status = 'running';
```

Verify no other orphans exist (jobs stuck `running` with expired leases):

```sql
SELECT job_id, worker_function, assigned_worker_id, updated_at
FROM job_queue
WHERE status = 'running'
  AND updated_at < NOW() - INTERVAL '10 minutes';
```

---

### Related Documentation

- `docs/postmortem-scale-to-zero-kills-jobs.md` — prior incident with identical keepalive class of failure
- `docs/runbook-gcp-gpu.md` — GPU worker operational runbook

---

## INC-001 — Scale-to-Zero Kills Active ComfyUI Jobs

**Date:** 2026-03-26
**See:** `docs/postmortem-scale-to-zero-kills-jobs.md` for full writeup.
**Resolution:** External URL keepalive with idle timeout deployed in Python pull_worker (revision 00053-l26). Fix was not carried forward when Go supervisor replaced pull_worker.
