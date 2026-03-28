# Legacy Doc Audit — spec 005-storage-architecture

**Audited**: 2026-03-25  
**Feature**: 005-storage-architecture (dual-DB, pull-model workers, io-cache)  

---

## Summary

| Doc | Status | Required Update |
|-----|--------|-----------------|
| `README.md` | Updated (T100) | Done — execution modes, Docker Compose, worker registration |
| `docs/runbook-gcp-gpu.md` | Updated (T118) | Done — log locations table added |
| `docs/runbook-comfyui-cloud-run-debugging.md` | Partial | Add observability API quick-check, note cache_manifest TTL |
| `docs/gcp-provisioning-and-teardown.md` | Partial | Add cost table for Cloud SQL + Cloud Run resources |
| `docs/runbook-db-cutover.md` | New (T111) | Created — full cutover and recovery guide |
| `docs/postmortem-comfyui-stuck-queued.md` | No change needed | References pull model correctly |
| `docs/wan-smoke-test.md` | No change needed | Test-specific doc, no architectural model references |

---

## Detailed Findings

### README.md — Updated
- **Old model**: stdio workers, HTTP dispatch, static worker endpoints, `PUT /api/workers` manual registration
- **New model**: Docker Compose with cluster DB container, pull-model polling, self-registration
- **Action taken**: Updated "Remote Simulation Mode" section → "Local-Docker Mode"; updated execution modes table; updated cloud deployment registration text

### docs/runbook-gcp-gpu.md — Updated
- **Old model**: No log location guidance
- **New model**: Cloud Run structured logs, local file logger via `FileLoggerProvider`, observability endpoints
- **Action taken**: Added "Log Locations" table with local file, Cloud Run, and Cloud SQL log references

### docs/runbook-comfyui-cloud-run-debugging.md — Needs T104 update
- **Current state**: Already uses pull-model terminology; has job_queue and cache_manifest SQL debugging
- **Missing**: Reference to observability API endpoints for quick queue-depth / error-rate check without direct SQL access
- **Pending action (T104)**: Add observability API quick-check to Quick Diagnostic Checklist section

### docs/gcp-provisioning-and-teardown.md — Needs T105 update
- **Current state**: Correct provisioning steps for pull-model Cloud SQL + Cloud Run
- **Missing**: Cost table showing approximate resource costs for Cloud SQL db-f1-micro, Cloud Run CPU/GPU workers, GCS buckets
- **Pending action (T105)**: Add "Resource Cost Reference" table

### docs/runbook-db-cutover.md — New (T111)
- Created fresh for spec 005 to document cluster DB schema cutover and config population via `scripts/populate-cluster-config.py`

---

## Terminology Reference (spec 005 canonical terms)

| Old term | New term |
|----------|----------|
| HTTP dispatch | pull-model / `SELECT FOR UPDATE SKIP LOCKED` |
| Worker registration via `PUT /api/workers` | Self-registration via cluster DB heartbeat |
| Remote Simulation Mode | local-docker mode |
| local-stdio / local-http | local-docker (Docker Compose) |
| Job dispatcher (push) | Job queue (cluster DB, workers poll) |
| Export backup script | `scripts/populate-cluster-config.py` (post-deploy populate) |
