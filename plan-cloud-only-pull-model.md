# Plan: Cloud-Only Pull-Model Architecture

**Created:** 2026-03-26  
**Status:** APPROVED  
**Scope:** Remove local Docker worker path, remove CPU stubs, unify on cloud GPU pull-model  

---

## Background

NeoVNext currently has two worker dispatch architectures:

1. **Pull-model (cloud):** Workers poll `job_queue` in Cluster DB, claim jobs with `FOR UPDATE SKIP LOCKED`. This is what's deployed and working (ComfyUI on Cloud Run GPU).
2. **Push-model (local Docker compose):** Backend HTTP-dispatches jobs to worker containers via `Workers:RemoteEndpoints` config. This is dead code — never fully integrated with pull-model, confuses the architecture, and nobody uses it.

Additionally, several CPU worker handlers are stubs (`visual_tagging`, `transcript_analysis`) or inferior fallbacks (`face_extraction` CPU vs GPU version). The system should be GPU-first, cloud-only.

### Current GCP GPU Quota

- **3 GPU Cloud Run instances** available in the project quota (only 1 currently used: ComfyUI on L4)
- **ComfyUI** needs the L4 (big models, 114GB on NFS) — keeps its own Cloud Run service
- **Other GPU workers** (face_extraction, visual_tagging, diarization) each get their own Cloud Run service
- **Deployment strategy (DECIDED):** One Cloud Run service PER worker function
  - Each service = one WORKER_FUNCTION, independently configurable polling times
  - Easier to isolate, scale, or move to a separate cluster later
  - GPU quota constraint: 3 GPU instances → need to be strategic about which functions truly need GPU vs which can run on CPU-only Cloud Run

### Architecture Target

```
Frontend ──→ Backend API ──→ INSERT into job_queue (Cluster DB)
                                     ↑
                    Cloud Run Workers (pull-model, one per function)
                    ├── neovlab-gpu-comfyui        (ComfyUI, L4 GPU)
                    ├── neovlab-gpu-face-extraction (InsightFace, GPU)
                    ├── neovlab-gpu-visual-tagging  (ViT/timm, GPU)
                    ├── neovlab-gpu-diarization     (pyannote, GPU)
                    ├── neovlab-worker-transcription (Whisper, GPU or CPU)
                    └── neovlab-worker-thumbnails    (ffmpeg, CPU-only OK)
                                     ↓
                    claim_next_job(worker_function=X) → process → complete
```

> **GPU Quota Note:** 3 GPU Cloud Run instances available. ComfyUI needs L4.
> The remaining 2 GPU slots cover face_extraction, visual_tagging, diarization.
> Transcription and thumbnails may not need dedicated GPU services.

---

## Phase 1: Clarify & Document the Pull-Model

**Goal:** Make the pull-model architecture crystal clear in docs. Update existing diagrams. Remove references to push-model.

**Parallel-safe:** Yes — no code changes, docs only.

### Tasks

- [ ] **1.1** Update `docs/diagrams/worker-architecture.md`
  - Remove any remaining push-model references
  - Add a "Cloud-Only Architecture" section showing the target state
  - Document: backend ONLY inserts into `job_queue`; workers ONLY poll from it

- [ ] **1.2** Update `docs/diagrams/job-lifecycle.md`
  - Lines 130-138: Remove references to `Workers:Mode`, `Workers:RemoteEndpoints`, `Workers:DispatchIntervalSeconds`
  - Replace with pull-model config: `DATABASE_URL`, `POLL_INTERVAL_SECONDS`, `WORKER_FUNCTION`

- [ ] **1.3** Update `docs/diagrams/gcp-infrastructure.md`
  - Add planned second GPU service (general GPU worker)
  - Update cost estimates for 2-service scenario

- [ ] **1.4** Update `docs/gcp-services-and-costs.md`
  - Remove any Docker Compose-related content
  - Clarify that all workers are Cloud Run GPU

- [ ] **1.5** Update `docs/diagrams/worker-implementation-gaps.md`
  - Mark local Docker sections as "REMOVED"
  - Update status table to reflect cloud-only target

- [ ] **1.6** Archive or update `docs/diagrams/app-topology.md`
  - Remove 6 CPU worker boxes from topology diagram
  - Show 2 Cloud Run GPU services instead

---

## Phase 2: Remove Stale Local Docker Path

**Goal:** Delete dead code — Docker Compose local worker stack, push-model dispatch, CPU worker containers, stale UI/UX.

**Sequential with Phase 1** (Phase 1 must complete first so docs are coherent).

### 2A: Docker Infrastructure (Delete Files)

- [ ] **2A.1** Delete `docker/docker-compose.yml`
  - Contains: 6 CPU worker services, x-worker-base, backend with push-model env vars
  - Impact: Local `docker compose up` no longer works (intentional)
  
- [ ] **2A.2** Delete `docker/docker-compose.gpu.yml`
  - Contains: Local GPU worker overlay (worker-tagging-gpu, worker-faces-gpu)
  - Impact: Local GPU testing path removed

- [ ] **2A.3** Delete `docker/worker.Dockerfile`
  - CPU worker container build. No longer needed.
  
- [ ] **2A.4** Delete `docker/worker-gpu.Dockerfile`
  - Local GPU worker container build (not Cloud Run).
  
- [ ] **2A.5** Delete `docker/deploy-cloudrun.sh`
  - CPU worker Cloud Run deployment script. Dead path.

- [ ] **2A.6** Delete `src/workers/comfyui_adapter.py`
  - Push-model FastAPI adapter (475 lines). Fully replaced by `handlers/comfyui.py`.
  - Has `/dispatch`, `/jobs/{id}/status`, artifact download — all obsolete in pull-model.

- [ ] **2A.7** Delete `src/workers/host.py`
  - CPU worker entrypoint (WORKER_PROCESSING_TYPE=cpu). Dead path.

- [ ] **2A.8** Update `Start-NeoVLab.ps1`
  - Remove any references to worker services from docker-compose
  - Keep: cluster-db Postgres startup if used for local dev
  - Keep: backend startup, frontend startup

### 2B: Backend API — Remove Push-Model Dispatch

- [ ] **2B.1** Delete push-model service files:
  - `Services/IWorkerClient.cs` (push-model interface)
  - `Services/RemoteWorkerHttpClient.cs` (HTTP dispatch to workers)
  - `Services/RunPodWorkerClient.cs` (RunPod GPU dispatch)
  - `Services/JobDispatcherService.cs` (polls queued jobs, dispatches via HTTP — this is the PUSH dispatcher, not the pull worker)
  - `Services/Orchestration/WorkerRegistrationService.cs` (registers remote workers from config)

- [ ] **2B.2** Delete push-model models:
  - `Models/Worker.cs` (WorkerId, Transport, BaseUrl, IsLocal)
  - `Models/ClusterWorkerRecord.cs` — **KEEP** for healthchecks and smoke tests (backend reads worker_registry for monitoring)
  - `Models/WorkerCapability.cs` (links workers to job types)

- [ ] **2B.3** Delete/modify controller:
  - `Controllers/WorkersController.cs` — DELETE if only used for push-model worker CRUD
  - OR MODIFY to be read-only (expose worker_registry data for monitoring UI)

- [ ] **2B.4** Delete push-model repository:
  - `Repositories/WorkerRegistryRepository.cs` — if it's the C# side only used by push dispatch
  - Note: `worker_registry` table is STILL USED by Python pull-workers for heartbeat. Don't drop the table, just remove C# CRUD for push-model config.

- [ ] **2B.5** Clean up `appsettings.json`:
  - Remove entire `Workers` section (`Mode`, `HeartbeatTimeoutSeconds`, `DispatchIntervalSeconds`, `RemoteEndpoints`)
  - Keep: `Gpu` section, `ConnectionStrings` section

- [ ] **2B.6** Clean up `Program.cs` / DI registration:
  - Remove: `WorkerRegistrationService` hosted service
  - Remove: `JobDispatcherService` hosted service (if push-only)
  - Remove: `IWorkerClient` / `IRemoteWorkerClient` registration
  - Remove: `LocalGpu` platform adapter branch
  - Keep: `GoogleCloudGpuPlatformAdapter`, `GpuBudgetGuardService`

- [ ] **2B.7** Update `GpuOptions.cs`:
  - Remove `IsLocalGpuPlatform()` method
  - Remove `LocalGpu` references
  - Simplify: GPU is always GoogleCloud when enabled

- [ ] **2B.8** Update `DefaultPolicySeedService.cs`:
  - Change ALL default execution modes from `"local-docker"` to `"cloud"`
  - Remove CPU-only job types that won't exist anymore
  - Seed only: `comfyui`, `visual_tagging_gpu`, `face_extraction_gpu`, `diarization`, `thumbnails`

- [ ] **2B.9** Update `JobQueueService.cs` `ResolveProcessingType()`:
  - Remove CPU path entirely — everything is GPU now
  - Simplify to always return `"gpu"` (or remove the concept)
  - OR: Keep processing_type but always set to `"gpu"` since all workers are GPU

- [ ] **2B.10** Delete stale test:
  - `tests/backend/Contract/RemoteWorkerContractTests.cs` (already marked obsolete)

### 2C: Frontend UI — Remove Worker Config

- [ ] **2C.1** Delete `SettingsInfrastructureSection.tsx`
  - Worker status dashboard showing CPU/GPU workers, heartbeats, states
  - No longer relevant when all workers are cloud pull-model

- [ ] **2C.2** Update `SettingsPolicySection.tsx`:
  - Remove `"local-docker"` execution mode option
  - Only show `"cloud"` mode (or remove mode selector entirely)
  - Update descriptions to reflect cloud-only architecture

- [ ] **2C.3** Update `src/frontend/src/api/client.ts`:
  - Remove: `Worker` interface, `getWorkers()`, `getWorker()`, `registerWorker()`, `deregisterWorker()`
  - Keep: `ExecutionPolicy`, `ProcessingJob`, GPU-related interfaces and endpoints

- [ ] **2C.4** Update `src/frontend/src/api/events.ts`:
  - Remove: `WorkerStatusChangedEvent`, `onWorkerStatusChanged()`
  - Keep: GPU job progress, budget warning events

- [ ] **2C.5** Find and remove any parent component that renders `SettingsInfrastructureSection`
  - Search for `SettingsInfrastructureSection` imports
  - Remove the section from the settings page layout

- [ ] **2C.6** Update `ProcessingJob.processingType` display:
  - Currently shows "CPU" or "GPU" badges in job list
  - Since everything is GPU now, either remove the badge or change to show the actual `worker_function` (more useful)
  - User wants to see "comfyui" vs work type rather than "cpu" vs "gpu"

### 2D: Database Cleanup

- [ ] **2D.1** DO NOT drop `worker_registry` table from Cluster DB
  - Python pull-workers still use it for heartbeat/state tracking
  - Only remove C# code that manages it for push-model config

- [ ] **2D.2** Update `execution_policies` default values
  - Change all `execution_mode` defaults to `"cloud"`
  - Remove `local-docker` as a valid option

- [ ] **2D.3** Consider simplifying `execution_policies` table
  - If everything is cloud GPU, this table may be overkill
  - Could be simplified to just: `job_type → enabled (boolean)`
  - Defer this to Phase 3 if too complex now

---

## Phase 3: Remove CPU Stubs, All-GPU Architecture

**Goal:** Remove CPU-only handlers, keep only GPU-capable implementations. Restructure work types.

**Sequential after Phase 2** (dead code must be removed first to avoid confusion).

### 3A: Remove CPU Stub Handlers

- [ ] **3A.1** Delete `src/workers/handlers/visual_tagging.py` (stub — returns empty tags)
- [ ] **3A.2** Delete `src/workers/handlers/transcript_analysis.py` (stub — returns first 200 chars)
- [ ] **3A.3** Delete `src/workers/handlers/face_extraction.py` (inferior CPU-only, no embeddings)

### 3B: Promote GPU Handlers

- [ ] **3B.1** Rename `handlers/face_extraction_gpu.py` → `handlers/face_extraction.py`
  - InsightFace buffalo_l, 512-dim embeddings
  - Drop `_gpu` suffix — all handlers are GPU-capable now, the suffix is noise

- [ ] **3B.2** Rename `handlers/visual_tagging_gpu.py` → `handlers/visual_tagging.py`
  - ViT/timm, GPU-accelerated
  - Drop `_gpu` suffix — same reasoning

- [ ] **3B.3** Keep `handlers/comfyui.py` (already production)
- [ ] **3B.4** Keep `handlers/thumbnails.py` (ffmpeg — works on CPU, fine on GPU box too)
- [ ] **3B.5** Keep `handlers/transcription.py` (Whisper — can use GPU if available)
- [ ] **3B.6** Keep `handlers/diarization.py` (pyannote — already uses CUDA if available)
- [ ] **3B.7** Leave `handlers/compute_file_hashes.py` alone — it's part of the API file discovery/ingest pipeline, NOT the worker system
  - Not in HANDLER_MAP, not in Docker configs, not called by any worker code
  - Reads API SQLite DB directly (different concern from cluster DB pull-model)
  - Handle separately if/when file ingest pipeline is redesigned

### 3C: Update HANDLER_MAP

- [ ] **3C.1** Update `handlers/__init__.py` HANDLER_MAP:
  ```python
  HANDLER_MAP = {
      "comfyui": "handlers.comfyui",           # dedicated L4 GPU service
      "thumbnails": "handlers.thumbnails",       # CPU-only (ffmpeg)
      "transcription": "handlers.transcription", # GPU-accelerated (Whisper)
      "diarization": "handlers.diarization",     # GPU-accelerated (pyannote)
      "visual_tagging": "handlers.visual_tagging",       # renamed from visual_tagging_gpu
      "face_extraction": "handlers.face_extraction",     # renamed from face_extraction_gpu
  }
  ```
  - Remove: `visual_tagging` (CPU stub), `transcript_analysis` (CPU stub), `face_extraction` (CPU fallback)
  - Remove: `face_extraction_gpu`, `visual_tagging_gpu` keys (replaced by renamed files)
  - Note: `transcript_analysis` removed entirely — LLM integration planned for Phase 7

### 3D: Restructure Work Type Display

- [ ] **3D.1** Rethink `processing_type` field
  - Currently: `"cpu"` or `"gpu"` — this distinction dies
  - New concept: `"comfyui"` (dedicated service) vs `"general"` (shared GPU service)?
  - Or just use `worker_function` as the primary identifier

- [ ] **3D.2** Update `JobQueueService.ResolveProcessingType()` in backend
  - Either: always return `"gpu"`
  - Or: return `"comfyui"` vs `"general-gpu"` based on job type
  - This determines which Cloud Run service picks up the job

- [ ] **3D.3** Update frontend job list display
  - Show `worker_function` (e.g., "comfyui", "face_extraction") instead of "CPU"/"GPU" badge
  - Show whether it ran on the ComfyUI service vs general GPU service

### 3E: GPU Deployment Strategy (DECIDED: One Service Per Function)

> **Decision:** Each worker function gets its own Cloud Run service with independent configuration.
> Rationale: Independent polling times, easier to scale/isolate, straightforward to move to its own cluster later.

- [ ] **3E.1** Plan GPU quota allocation across services:

  | Service | GPU Required? | Notes |
  |---------|:---:|---|
  | `neovlab-gpu-comfyui` | **Yes** (L4) | Already deployed. 114GB models on NFS. |
  | `neovlab-gpu-face-extraction` | **Yes** (T4/L4) | InsightFace + ONNX Runtime GPU |
  | `neovlab-gpu-visual-tagging` | **Yes** (T4/L4) | ViT + timm, GPU-accelerated |
  | `neovlab-gpu-diarization` | Preferred | pyannote uses CUDA if available |
  | `neovlab-worker-transcription` | Optional | Whisper is fast on GPU but works on CPU |
  | `neovlab-worker-thumbnails` | **No** | ffmpeg only, CPU Cloud Run is fine |

  With 3-GPU quota: ComfyUI (1) + face_extraction (1) + visual_tagging (1) = 3.
  Diarization may need to share with one of those or wait for quota increase.
  Transcription and thumbnails can use CPU-only Cloud Run (no GPU quota needed).

- [ ] **3E.2** Create deploy scripts (one per service):
  - `docker/deploy-gpu-face-extraction.sh`
  - `docker/deploy-gpu-visual-tagging.sh`
  - `docker/deploy-gpu-diarization.sh`
  - `docker/deploy-worker-transcription.sh`
  - `docker/deploy-worker-thumbnails.sh`
  - Each script sets `WORKER_FUNCTION=<name>` and appropriate GPU/CPU config

- [ ] **3E.3** Create shared Dockerfile: `docker/worker-general.Dockerfile`
  - Based on CUDA base image (GPU services) or Python base (CPU services)
  - Install ALL handler dependencies (insightface, onnxruntime-gpu, timm, whisper, pyannote.audio, ffmpeg)
  - Single image, deployed as multiple services with different WORKER_FUNCTION env var
  - Much smaller than ComfyUI image (no 114GB models)

- [ ] **3E.4** Configure independent polling intervals:
  - Each service has its own `POLL_INTERVAL_SECONDS` env var
  - ComfyUI: 3s (already set, latency-sensitive)
  - face_extraction/visual_tagging: 5-10s (batch-oriented)
  - diarization: 10s (long-running jobs)
  - transcription: 5s (moderate latency)
  - thumbnails: 3s (quick jobs)

---

## Phase 4: Diarization — Make It Work

**Goal:** Get pyannote diarization running as a cloud pull-model worker.

**Parallel-safe:** Yes — independent of Phases 2-3 as long as it follows the pull-model pattern established by ComfyUI.

### Tasks

- [ ] **4.1** Validate diarization handler locally
  - Test `handlers/diarization.py` with sample audio file
  - Ensure HUGGINGFACE_TOKEN auth works
  - Verify pyannote model downloads correctly

- [ ] **4.2** Add voice embedding extraction
  - `speaker_segments.voice_embedding` (192-dim) exists in schema but isn't populated
  - Extend diarization handler to extract embeddings from pyannote output
  - Write embeddings to output JSON

- [ ] **4.3** Integrate with pull-worker pattern
  - Verify diarization handler works inside `pull_worker.py` `_process_job()` flow
  - Test: `WORKER_FUNCTION=diarization` → claims diarization job → processes → outputs
  - Verify artifact upload to io-cache

- [ ] **4.4** Add backend support for diarization results
  - Backend parses diarization JSON output
  - Writes to `speaker_segments` table in cluster DB
  - Exposes via API endpoint for frontend

- [ ] **4.5** Write tests
  - Unit test for diarization handler
  - Integration test: submit diarization job → worker claims → processes → backend reads result

- [ ] **4.6** Cloud deployment
  - Include diarization in the general GPU worker container
  - Test cold start with pyannote model download
  - Verify HUGGINGFACE_TOKEN is available as Cloud Run secret

### Phase 4 Prerequisite

Diarization slots into the same pull-model as ComfyUI. The pattern:
1. Backend inserts row into `job_queue` (worker_function='diarization', processing_type='gpu')
2. GPU worker polls `claim_next_job()` matching worker_function='diarization'
3. Worker loads pyannote, processes audio, writes output to io-cache
4. Worker marks job complete in `job_queue`
5. Backend polls/detects completion, downloads artifacts, writes to `speaker_segments` table

This is identical to how ComfyUI works, just with a different handler.

---

## Phase 5: Firm Up Thumbnails

**Goal:** Ensure thumbnails handler is production-ready in the cloud pull-model.

**Parallel-safe:** Yes — independent work.

### Tasks

- [ ] **5.1** Validate thumbnails handler in pull-model flow
  - `WORKER_FUNCTION=thumbnails` → claims thumbnail job → ffmpeg extraction → artifacts
  - Verify: artifact paths, io-cache writes, job completion

- [ ] **5.2** Backend integration
  - Verify backend writes to `thumbnails` table in cluster DB on job completion
  - API endpoint serves thumbnail gallery for media items

- [ ] **5.3** Add test coverage
  - Unit test for thumbnails handler
  - Test with various video formats/durations

- [ ] **5.4** Include in general GPU worker container
  - ffmpeg is available in most CUDA base images
  - Thumbnails don't need GPU, but run on the GPU box fine

---

## Phase 6: Test Coverage

**Goal:** Add test coverage for all handlers and pull-model infrastructure.

**After:** Phases 2-5 complete.

### Tasks

- [ ] **6.1** Pull-model infrastructure tests
  - Test `PullWorkerRuntime` startup/shutdown lifecycle
  - Test `ClusterDbClient.claim_next_job()` with concurrent workers
  - Test lease expiry and reclaim logic
  - Test heartbeat loop

- [ ] **6.2** Handler unit tests
  - `comfyui` — mock HTTP calls to ComfyUI server
  - `face_extraction_gpu` — test with sample image/video
  - `visual_tagging_gpu` — test with sample image
  - `transcription` — test with sample audio
  - `diarization` — test with sample audio
  - `thumbnails` — test with sample video

- [ ] **6.3** Integration tests
  - End-to-end: submit job via API → worker claims → processes → backend reads result
  - Test with each handler type

---

## Phase 7: LLM Integration (Future)

**Goal:** Add transcript analysis via LLM API (OpenAI or equivalent). Replaces the deleted `transcript_analysis` CPU stub.

**Placeholder** — scope TBD when we get there.

### Tasks

- [ ] **7.1** Design transcript analysis LLM pipeline
  - Define: summarization, entity extraction, topic classification, sentiment?
  - Choose LLM provider (OpenAI API, Gemini, Anthropic, local Ollama)
  - Define output schema for `transcript_analysis` results

- [ ] **7.2** Implement `handlers/transcript_analysis.py` (new, LLM-backed)
  - Reads transcript from prior transcription job output
  - Calls LLM API for analysis
  - Returns structured JSON

- [ ] **7.3** Add to HANDLER_MAP and deploy as its own Cloud Run service
  - `neovlab-worker-transcript-analysis` (CPU-only, LLM API calls don't need GPU)

---

## Execution Order

```
Week 1 (Parallel tracks):
├── Phase 1: Docs cleanup (1-2 days)
├── Phase 4: Diarization R&D (independent, ongoing)
└── Phase 5: Thumbnails validation (1 day)

Week 2 (Sequential):
├── Phase 2A: Delete Docker files + comfyui_adapter.py + host.py (30 min)
├── Phase 2B: Backend push-model removal (1-2 days)
├── Phase 2C: Frontend UI cleanup (half day)
└── Phase 2D: Database cleanup (1 hour)

Week 3:
├── Phase 3A-3C: Remove CPU stubs, rename GPU handlers (1 day)
├── Phase 3D: Work type display refactor (1 day)
└── Phase 3E: Per-function Cloud Run deploy scripts (2-3 days)

Week 4:
├── Phase 4 (continued): Diarization deployment
├── Phase 5 (continued): Thumbnails deployment
└── Phase 6: Test coverage

Future:
├── Phase 7: LLM integration (transcript analysis)
└── Phase 4: Diarization (may extend — pyannote model tuning)
```

---

## Files Affected Summary

### DELETE (20 files)

| File | Phase |
|------|-------|
| `docker/docker-compose.yml` | 2A |
| `docker/docker-compose.gpu.yml` | 2A |
| `docker/worker.Dockerfile` | 2A |
| `docker/worker-gpu.Dockerfile` | 2A |
| `docker/deploy-cloudrun.sh` | 2A |
| `src/workers/comfyui_adapter.py` | 2A |
| `src/workers/host.py` | 2A |
| `Services/IWorkerClient.cs` | 2B |
| `Services/RemoteWorkerHttpClient.cs` | 2B |
| `Services/RunPodWorkerClient.cs` | 2B |
| `Services/JobDispatcherService.cs` | 2B |
| `Services/Orchestration/WorkerRegistrationService.cs` | 2B |
| `Models/Worker.cs` | 2B |
| `Models/WorkerCapability.cs` | 2B |
| `Controllers/WorkersController.cs` | 2B |
| `Repositories/WorkerRegistryRepository.cs` | 2B |
| `tests/backend/Contract/RemoteWorkerContractTests.cs` | 2B |
| `SettingsInfrastructureSection.tsx` | 2C |
| `handlers/visual_tagging.py` (CPU stub) | 3A |
| `handlers/transcript_analysis.py` (CPU stub) | 3A |
| `handlers/face_extraction.py` (CPU fallback) | 3A |

### MODIFY (15+ files)

| File | Phase | Changes |
|------|-------|---------|
| `Start-NeoVLab.ps1` | 2A | Remove worker service refs |
| `appsettings.json` | 2B | Remove `Workers` section |
| `Program.cs` | 2B | Remove push-model DI registrations |
| `GpuOptions.cs` | 2B | Remove LocalGpu |
| `DefaultPolicySeedService.cs` | 2B | Cloud-only defaults |
| `JobQueueService.cs` | 2B/3D | Remove CPU processing_type |
| `SettingsPolicySection.tsx` | 2C | Remove local-docker option |
| `api/client.ts` | 2C | Remove worker interfaces/endpoints |
| `api/events.ts` | 2C | Remove worker events |
| `handlers/__init__.py` | 3C | Remove CPU entries from HANDLER_MAP |
| `pull_worker.py` | - | No changes needed (already one-per-function) |
| `cluster_db.py` | - | No changes needed (already filters by single worker_function) |
| `host_gpu.py` | 3E | Verify it works as general GPU entrypoint |
| Job list components | 3D | Show worker_function instead of CPU/GPU |
| Various docs/ | 1 | Update diagrams |

### CREATE (5+ files)

| File | Phase | Purpose |
|------|-------|---------|
| `docker/worker-general.Dockerfile` | 3E | General GPU/CPU worker container |
| `docker/deploy-gpu-face-extraction.sh` | 3E | Deploy face extraction GPU service |
| `docker/deploy-gpu-visual-tagging.sh` | 3E | Deploy visual tagging GPU service |
| `docker/deploy-gpu-diarization.sh` | 3E | Deploy diarization GPU service |
| `docker/deploy-worker-transcription.sh` | 3E | Deploy transcription service |
| `docker/deploy-worker-thumbnails.sh` | 3E | Deploy thumbnails service |
| Test files | 6 | Handler unit/integration tests |

---

## Resolved Questions

| # | Question | Decision |
|---|---------|----------|
| 1 | GPU quota — what's available? | 3 GPU Cloud Run instances in us-east4. ComfyUI uses 1 L4. 2 remaining for face_extraction + visual_tagging. Diarization may share or await quota increase. Transcription + thumbnails can run CPU-only. |
| 2 | Keep `_gpu` suffix on handlers? | **Rename** — drop the suffix. Everything is GPU-capable now. |
| 3 | Multi-function worker or one-per-function? | **One per function.** Independent polling times, easier to scale/isolate/migrate later. |
| 4 | Transcript analysis — drop or plan LLM? | Remove stub now. **LLM integration = Phase 7** (future). |
| 5 | `compute_file_hashes.py` — entangled? | **No entanglement.** Not in HANDLER_MAP, not called by worker system. It's an orphaned API ingest utility. Leave it alone during Docker removal. |
| 6 | `comfyui_adapter.py` — dead code? | **Yes, toss it.** Replaced by `handlers/comfyui.py`. Push-model FastAPI adapter, fully superseded. |
| 7 | `ClusterWorkerRecord.cs` — push-only? | **Keep** for healthchecks and smoke tests. Backend reads `worker_registry` for monitoring. |

---

## Success Criteria

- [ ] `docker-compose.yml` and all local worker infrastructure deleted
- [ ] No references to push-model HTTP dispatch in backend
- [ ] No CPU-only worker stubs remain
- [ ] Frontend settings page shows cloud-only configuration
- [ ] All workers use pull-model from cluster DB
- [ ] Diarization handler validated and deployable
- [ ] Thumbnails handler validated in pull-model flow
- [ ] Tests exist for all active handlers
- [ ] Only 2-3 GPU Cloud Run services needed (ComfyUI L4 + face_extraction + visual_tagging)
- [ ] Non-GPU handlers deployed as CPU-only Cloud Run services
- [ ] Each worker function has its own independently-configurable Cloud Run service
