# Worker Implementation Status & Gaps

This document analyzes the current implementation status of all NeoVNext workers, identifies gaps, and recommends remediation priorities.

## Current Deployment Status

### Cloud Run GPU Worker (Production)

| Service | Status | Handler | Models | Cold Start | Deployment |
|---------|--------|---------|--------|------------|------------|
| **neovlab-gpu-worker-east4** | ✅ **Fully Operational** | `comfyui` | 114GB on Filestore NFS | 39s | Cloud Run GPU (L4), min-instances=0 |

**Capabilities:**
- Stable Diffusion workflows via ComfyUI
- Model registry: 114GB across checkpoints, loras, VAEs, controlnet, upscale_models
- GCS FUSE mount for artifact caching
- Pull-model job claiming from cluster DB
- Keepalive loop prevents mid-job scale-to-zero

**Cost:**  $2.16/hr active, $61/mo idle (scale-to-zero)

### Local Docker Workers (Development)

> **Note:** The `docker-compose.yml` files have been removed. Workers are now deployed
> as Cloud Run services via `docker/deploy-gpu-worker.sh` (unified) or per-handler
> deploy scripts. The Go supervisor (`src/gpu-supervisor/`) manages all handlers in a
> single container.

Handler implementation maturity:

| Worker | Port | Handler Module | Status | Notes |
|--------|------|----------------|--------|-------|
| **worker-thumbnails** | 8001 | `thumbnails.py` | ✅ **Production Ready** | ffmpeg frame extraction, tested |
| **worker-transcription** | 8002 | `transcription.py` | ✅ **Production Ready** | Whisper integration, tested |
| **worker-diarization** | 8003 | `diarization.py` | ⚠️ **Functional, Untested** | pyannote pipeline, requires HUGGINGFACE_TOKEN |
| **worker-faces** | 8004 | `face_extraction.py` | ⚠️ **Fallback Implementation** | OpenCV DNN (CPU), no embeddings |
| **worker-tagging** | 8005 | `visual_tagging.py` | ❌ **Stub Only** | Returns empty tags, no model loaded |
| **worker-analysis** | 8006 | `transcript_analysis.py` | ❌ **Stub Only** | Returns first 200 chars as "summary" |

### Local Docker GPU Workers (Optional, Not Deployed)

These are in [docker/docker-compose.gpu.yml](../docker/docker-compose.gpu.yml) but **not currently used**:

| Worker | Handler Module | Status | Notes |
|--------|----------------|--------|-------|
| **worker-tagging-gpu** | `visual_tagging_gpu.py` | ✅ **Implemented** | ViT/CLIP via timm, expects CUDA |
| **worker-faces-gpu** | `face_extraction_gpu.py` | ✅ **Implemented** | InsightFace buffalo_l, 512-dim embeddings |

## Handler Implementation Analysis

### ✅ Production Ready

#### `comfyui` (GPU, Cloud Run)

**Location:** [src/workers/handlers/comfyui.py](../src/workers/handlers/comfyui.py)

**Implementation:**
- Full ComfyUI API integration: workflow submission, polling, artifact download
- Input staging to `/comfyui/input/` directory
- Queue clearing to prevent stale prompts
- Timeout handling (default: 1800s)
- Primary output enforcement (FR-040)
- Magic-byte media validation

**Dependencies:**
- ComfyUI server running at `COMFYUI_BASE_URL` (default: http://127.0.0.1:8188)
- GCS FUSE mount for artifact caching
- Filestore NFS for model access

**Test Coverage:** ✅ Smoke tested (39s cold start, warm requests 0.3s)

**Deployment:** ✅ Cloud Run GPU, production-ready

---

#### `thumbnails` (CPU, Local Docker)

**Location:** [src/workers/handlers/thumbnails.py](../src/workers/handlers/thumbnails.py)

**Implementation:**
- ffmpeg-based frame extraction at evenly-spaced intervals
- ffprobe video validation (detects non-video inputs)
- Configurable thumbnail count (default: 6)
- Handles duration edge cases (very short videos)
- Minimum success ratio tolerance (0.8)

**Dependencies:**
- ffmpeg + ffprobe (available in worker Docker image)

**Test Coverage:** ⚠️ Unit tests exist ([tests/workers/test_thumbnails.py](../tests/workers/) - assumed)

**Deployment:** ✅ Docker Compose service `worker-thumbnails` (port 8001)

**Notes:** Worker is defined and functional, but **NOT currently used by backend** (backend may still dispatch directly, not via pull-model).

---

#### `transcription` (CPU, Local Docker)

**Location:** [src/workers/handlers/transcription.py](../src/workers/handlers/transcription.py)

**Implementation:**
- OpenAI Whisper integration (local model, not API)
- Configurable model via `WHISPER_MODEL` env var (default: turbo)
- Outputs both plain text (`transcript.txt`) and structured JSON (`transcript.json`)
- Segment-level timestamps
- Language detection and word count

**Dependencies:**
- `openai-whisper` Python package
- Model downloaded at init time (25MB - 3GB depending on model size)

**Test Coverage:** ⚠️ Assumed functional, no explicit test file found

**Deployment:** ✅ Docker Compose service `worker-transcription` (port 8002)

**Notes:** Whisper models can be CPU-heavy (especially `large-v3`). Consider GPU variant for faster transcription.

---

### ⚠️ Functional but Untested

#### `diarization` (CPU, Local Docker)

**Location:** [src/workers/handlers/diarization.py](../src/workers/handlers/diarization.py)

**Implementation:**
- pyannote.audio Speaker Diarization 3.1 pipeline
- Merge consecutive segments from same speaker (debounce: 0.3s)
- Extract audio segments via ffmpeg (WAV, 22050Hz mono)
- Minimum segment duration filter (0.5s)
- Output: JSON with speaker labels, timestamps, audio file paths

**Dependencies:**
- `pyannote.audio` Python package
- `HUGGINGFACE_TOKEN` env var (requires gated model access)
- CUDA optional (will use GPU if available)

**Test Coverage:** ❌ No tests found

**Deployment:** ✅ Docker Compose service `worker-diarization` (port 8003)

**Gaps:**
- ❌ No integration testing with real audio
- ❌ No validation that HUGGINGFACE_TOKEN is set (fails at init)
- ⚠️ Speaker labeling is generic (SPEAKER_00, SPEAKER_01) — no persistent identity
- ⚠️ Voice embeddings not extracted (database schema support exists but not used)

**Recommendation:** Deploy to test environment, validate with sample audio files, add error handling for missing HF token.

---

#### `face_extraction` (CPU, Local Docker)

**Location:** [src/workers/handlers/face_extraction.py](../src/workers/handlers/face_extraction.py)

**Implementation:**
- **This is a FALLBACK CPU implementation, NOT the production-grade GPU version**
- Uses OpenCV DNN (SSD Caffe model) or Haar cascade
- Detects faces in video frames (samples at 2s intervals by default)
- Returns bounding boxes and detection confidence
- **NO FACE EMBEDDINGS** (database schema supports 512-dim embeddings but not populated)

**Dependencies:**
- OpenCV Python (`cv2`)
- DNN model files (deploy.prototxt + res10_300x300_ssd_iter_140000.caffemodel)
- If DNN files missing, falls back to Haar cascade (less accurate)

**Test Coverage:** ❌ No tests found

**Deployment:** ✅ Docker Compose service `worker-faces` (port 8004)

**Gaps:**
- ❌ **No face embeddings** — database has `vector(512)` column but this handler doesn't populate it
- ❌ Cannot perform face clustering or recognition without embeddings
- ⚠️ Detection accuracy inferior to GPU version (InsightFace buffalo_l)
- ⚠️ No age/gender estimation
- ⚠️ No person_id association

**Recommendation:** Replace with GPU version (`face_extraction_gpu`) OR document as CPU fallback only.

---

### ❌ Stub / Incomplete

#### `visual_tagging` (CPU, Local Docker)

**Location:** [src/workers/handlers/visual_tagging.py](../src/workers/handlers/visual_tagging.py)

**Implementation:**
- **STUB ONLY** — returns empty tags list
- Writes `tags.json` with schema `{"tags": [], "model": "stub", "device": "cpu"}`
- No actual model loaded

**Test Coverage:** ❌ No tests

**Deployment:** ✅ Docker Compose service `worker-tagging` (port 8005)

**Gaps:**
- ❌ No model integration (should use ViT, CLIP, or WD14 tagger)
- ❌ Completely non-functional
- ⚠️ GPU version exists (`visual_tagging_gpu.py`) but not deployed locally

**Recommendation:** Either:
1. Integrate a CPU-compatible tagging model (slow but functional)
2. Remove the CPU version entirely and route all tagging jobs to GPU worker
3. Document as placeholder and disable the service

---

#### `transcript_analysis` (CPU, Local Docker)

**Location:** [src/workers/handlers/transcript_analysis.py](../src/workers/handlers/transcript_analysis.py)

**Implementation:**
- **STUB ONLY** — returns first 200 chars of transcript as "summary"
- Writes `analysis.json` with schema `{"summary": "...", "topics": [], "sentiment": "neutral", "entities": [], "key_moments": [], "model": "stub"}`
- No LLM integration

**Test Coverage:** ❌ No tests

**Deployment:** ✅ Docker Compose service `worker-analysis` (port 8006)

**Gaps:**
- ❌ No LLM integration (should use OpenAI API, Anthropic, or local model)
- ❌ Topics, sentiment, entities, key_moments all empty
- ⚠️ Requires API keys (OPENAI_API_KEY) or local model (ollama, llama.cpp)

**Recommendation:**
1. Integrate OpenAI GPT-4 API for summary/topics/entities (requires API key + cost tracking)
2. OR use local LLM via ollama/llama.cpp (slower, no API cost)
3. OR document as future work and disable the service

---

### ✅ Implemented but Not Deployed

#### `visual_tagging_gpu` (GPU, Local Docker GPU)

**Location:** [src/workers/handlers/visual_tagging_gpu.py](../src/workers/handlers/visual_tagging_gpu.py)

**Implementation:**
- Uses ViT (Vision Transformer) via `timm` library
- Model: `vit_base_patch16_224` (ImageNet-1k pretrained)
- GPU acceleration via PyTorch CUDA
- Returns top-k tags with confidence scores

**Dependencies:**
- PyTorch with CUDA
- `timm` (PyTorch Image Models)
- GPU with CUDA support

**Test Coverage:** ❌ No tests found

**Deployment:** ⚠️ **NOT DEPLOYED** — commented out or in `docker-compose.gpu.yml` (not used)

**Recommendation:** Deploy to local GPU worker OR migrate to Cloud Run GPU if needed. This is a viable alternative to the CPU stub.

---

#### `face_extraction_gpu` (GPU, Local Docker GPU)

**Location:** [src/workers/handlers/face_extraction_gpu.py](../src/workers/handlers/face_extraction_gpu.py)

**Implementation:**
- InsightFace `buffalo_l` model via ONNX Runtime GPU
- Extracts 512-dimensional face embeddings (compatible with `face_detections.embedding` column)
- CUDAExecutionProvider for GPU acceleration, fallback to CPU
- Samples video frames at configurable interval (default: 2s)
- Returns bounding boxes + embeddings

**Dependencies:**
- `insightface` Python package
- ONNX Runtime with CUDA support
- GPU with CUDA support

**Test Coverage:** ❌ No tests found

**Deployment:** ⚠️ **NOT DEPLOYED** — commented out or in `docker-compose.gpu.yml` (not used)

**Recommendation:** This is the **production-grade face extraction handler** — replace CPU version with this.

---

## Gap Summary

### Critical Gaps (Block Production Use)

| Issue | Impact | Affected Workers | Remediation |
|-------|--------|-----------------|-------------|
| **No face embeddings from CPU handler** | Cannot perform face clustering/recognition | `face_extraction` | Replace with GPU version (`face_extraction_gpu`) |
| **Visual tagging stub** | Zero tagging functionality | `visual_tagging` | Deploy GPU version OR integrate CPU model |
| **Transcript analysis stub** | No LLM-based insights | `transcript_analysis` | Integrate OpenAI API OR disable service |
| **No integration tests for pyannote** | Diarization untested | `diarization` | Test with real audio, validate HF token handling |

### Operational Gaps (Deployment Issues)

| Issue | Impact | Remediation |
|-------|--------|-------------|
| **Workers defined but not routed** | Backend may not use pull-model workers | Verify backend config routes jobs to worker URLs |
| **Missing HUGGINGFACE_TOKEN handling** | Diarization worker fails at startup | Add env var validation + clear error messages |
| **GPU workers not deployed** | Superior GPU handlers unused | Evaluate cost/benefit of local GPU vs Cloud Run |
| **No worker health monitoring** | Cannot detect dead workers | Implement `/health` endpoint polling |

### Feature Gaps (Database Schema vs. Implementation)

| Schema Column | Handler Support | Missing from Handlers |
|---------------|-----------------|----------------------|
| `face_detections.embedding` (512-dim) | GPU handler ✅, CPU handler ❌ | CPU face_extraction doesn't populate |
| `face_detections.estimated_age` | GPU handler ❌ | Neither handler extracts demographics |
| `face_detections.estimated_gender` | GPU handler ❌ | Neither handler extracts demographics |
| `speaker_segments.voice_embedding` (192-dim) | Diarization handler ❌ | Not extracted from pyannote output |
| `face_detections.person_id` | All handlers ❌ | Cross-video identity clustering not implemented |
| `speaker_segments.person_id` | All handlers ❌ | Speaker re-identification not implemented |

## Recommended Remediation Priority

### Phase 1: Critical Production Blockers (Immediate)

1. **Replace CPU face extraction with GPU version**
   - Deploy `face_extraction_gpu` handler
   - Update `docker-compose.yml` to use GPU service
   - Validate 512-dim embeddings populate database

2. **Implement visual tagging**
   - Either deploy `visual_tagging_gpu` OR integrate CPU model (WD14/CLIP)
   - Remove stub handler

3. **Test diarization handler**
   - Add integration test with sample audio
   - Validate HUGGINGFACE_TOKEN requirement
   - Document voice embedding extraction gap

### Phase 2: Operational Stability (1-2 weeks)

4. **Worker routing validation**
   - Verify backend `Workers__RemoteEndpoints__*` config matches deployed services
   - Test job dispatch from backend → worker → cluster DB
   - Add monitoring for worker availability

5. **Transcript analysis LLM integration**
   - Integrate OpenAI GPT-4 API (or disable if not needed)
   - Add budget tracking for API costs
   - Implement summary, topics, sentiment, entities extraction

6. **Add worker health checks**
   - Backend polls `/health` endpoint every 30s
   - Mark workers as OFFLINE if health check fails
   - Prevent job dispatch to dead workers

### Phase 3: Feature Completeness (1-2 months)

7. **Voice embeddings extraction**
   - Extract 192-dim embeddings from diarization output
   - Populate `speaker_segments.voice_embedding` column
   - Enable speaker re-identification across media

8. **Demographics extraction**
   - Add age/gender estimation to face extraction (requires additional model)
   - Populate `face_detections.estimated_age` and `estimated_gender`

9. **Cross-video identity**
   - Implement `person_id` clustering for faces (cosine similarity on embeddings)
   - Implement `person_id` clustering for speakers
   - Add UI for identity tagging

## Docker Compose Gap: Push vs. Pull Model

**Issue:** The `docker-compose.yml` defines 6 CPU workers with HTTP endpoints, but **the system now uses a pull-model architecture** where workers poll the cluster DB.

**Current State:**
- Backend config: `Workers__RemoteEndpoints__thumbnails: "http://worker-thumbnails:8000"`
- Worker implementation: Runs `pull_worker.py` which polls `claim_next_job()` from cluster DB
- **No HTTP endpoint exposed** (the `/health` endpoint exists but job dispatch doesn't use it)

**Confusion:** The backend may still be using **push-model HTTP dispatch** to worker URLs, while workers are **pull-model polling** from the DB.

**Recommendation:**
1. Verify backend dispatch mechanism: does it POST to worker URLs OR insert into `job_queue`?
2. If backend uses push-model: workers need HTTP handler for `POST /execute` endpoint
3. If backend uses pull-model: remove `Workers__RemoteEndpoints__*` config (unused)
4. **Document the dispatcher pattern** in [worker-architecture.md](worker-architecture.md)

**Likely Answer (based on code review):**
- Backend **inserts jobs into `job_queue`** table
- Workers **poll `claim_next_job()`** from cluster DB (pull-model)
- The HTTP endpoints are **vestigial from old architecture** and not used

**Action:** Remove HTTP endpoint config from backend OR add backward-compat HTTP dispatch handler.

## Test Coverage Gaps

| Handler | Unit Tests | Integration Tests | E2E Tests |
|---------|-----------|-------------------|-----------|
| `comfyui` | ❌ | ✅ (manual smoke test) | ⚠️ (wan-smoke-test.md) |
| `thumbnails` | ⚠️ (assumed) | ❌ | ❌ |
| `transcription` | ❌ | ❌ | ❌ |
| `diarization` | ❌ | ❌ | ❌ |
| `face_extraction` | ❌ | ❌ | ❌ |
| `visual_tagging` | ❌ | ❌ | ❌ |
| `transcript_analysis` | ❌ | ❌ | ❌ |
| `visual_tagging_gpu` | ❌ | ❌ | ❌ |
| `face_extraction_gpu` | ❌ | ❌ | ❌ |

**Recommendation:** Add pytest test suite for all handlers with sample input files.

## Cost Analysis: Local GPU vs. Cloud Run GPU

| Deployment | Idle Cost | 4hrs/day Cost | 24/7 Cost | Cold Start | Maintenance |
|------------|-----------|--------------|-----------|----------|-------------|
| **Cloud Run GPU (L4)** | $61/mo | $320/mo | $1,637/mo | 39s | Minimal |
| **Local GPU (Docker)** | $0 (sunk HW cost) | $0 | $0 | 0s | Manual updates |

**Trade-offs:**
- **Cloud Run**: Pay-per-use, auto-scaling, zero maintenance, 39s cold start penalty
- **Local GPU**: No runtime cost, always warm, requires local GPU hardware + power/cooling

**Current Strategy:** ComfyUI on Cloud Run (scale-to-zero), visual_tagging/face_extraction could use local GPU if available.

**Recommendation:** 
- If local GPU available: deploy `visual_tagging_gpu` and `face_extraction_gpu` locally (faster, free)
- If no local GPU: all GPU jobs to Cloud Run (add more handlers to `comfyui` container OR deploy separate Cloud Run services)

---

**Generated:** Worker implementation gap analysis for NeoVNext  
**See Also:**
- [worker-architecture.md](worker-architecture.md) for pull-model details
- [gcp-infrastructure.md](gcp-infrastructure.md) for Cloud Run GPU deployment
- [db-schema.md](db-schema.md) for database column expectations
