# Worker Architecture

This document visualizes the NeoVNext GPU worker ecosystem running on RunPod infrastructure.

## Worker Architecture Overview

All ML handlers run as Python subprocesses managed by the Go GPU Supervisor on a single
RunPod GPU pod. There are no separate CPU/GPU worker containers — the supervisor handles
everything on the pod, using per-handler Python venvs for isolation.

```mermaid
graph TB
    subgraph "RunPod GPU Pod"
        SUP[Go Supervisor<br/>/usr/local/bin/supervisor]

        subgraph "GPU Handlers (per-handler venvs)"
            H1[":9001 transcription<br/>Whisper turbo ~1.5 GB<br/>/venvs/transcription/"]
            H2[":9002 diarization<br/>pyannote ~0.5 GB<br/>/venvs/diarization/"]
            H3[":9003 face_extraction<br/>InsightFace ~1.5 GB<br/>/venvs/face/"]
            H4[":9004 visual_tagging<br/>ViT-B/16 ~0.5 GB<br/>/venvs/visual-tagging/"]
        end

        COMFY[ComfyUI :8188<br/>/opt/comfyui-baked/<br/>SD/SDXL ~7 GB or Wan 14B ~18 GB]

        subgraph "Pod Services"
            SSH[sshd :22]
            FB[FileBrowser :8080]
            JUP[JupyterLab :8888]
        end

        subgraph "Networking"
            TS[Tailscale<br/>userspace networking]
            SOCAT[socat tunnel<br/>localhost:15432 → desktop:5432]
        end
    end

    API[Desktop C# API :6000]
    DB[(Desktop PostgreSQL<br/>Docker :5432)]

    SUP --> H1
    SUP --> H2
    SUP --> H3
    SUP --> H4
    SUP --> COMFY

    SUP -->|"HTTP file transfer\nGET/PUT /api/cache/..."| API
    SOCAT -->|"Tailscale nc"| DB

    style SUP fill:#00897b,stroke:#004d40,color:#fff
    style COMFY fill:#4285f4,stroke:#1a73e8,color:#fff
    style H1 fill:#e3f2fd,stroke:#2196f3
    style H2 fill:#e3f2fd,stroke:#2196f3
    style H3 fill:#e3f2fd,stroke:#2196f3
    style H4 fill:#e3f2fd,stroke:#2196f3
    style API fill:#1a3a4a,stroke:#00bcd4,color:#e0f7fa
    style DB fill:#ea4335,stroke:#c5221f,color:#fff
```

## Docker Image

- **Base:** `runpod/comfyui:cuda13.0` (Python 3.12, CUDA 13.0, torch 2.10, ComfyUI pre-baked)
- **Dockerfile:** `docker/worker-gpu-go-runpod.Dockerfile`
- **Registry:** Docker Hub `tagowar/neovnext-gpu-worker:v1.0.14`

### Per-Handler Virtual Environments

Each ML handler runs in an isolated venv with `--system-site-packages` to inherit system
torch/numpy without reinstalling. Only handler-specific packages are installed inside each venv.

| Handler | Venv Path | Extra Packages | GPU VRAM |
|---------|-----------|----------------|----------|
| transcription | `/venvs/transcription/` | openai-whisper | ~1.5 GB |
| diarization | `/venvs/diarization/` | pyannote.audio, transformers | ~0.5 GB |
| face_extraction | `/venvs/face/` | insightface, onnxruntime-gpu | ~1.5 GB |
| visual_tagging | `/venvs/visual-tagging/` | timm | ~0.5 GB |
| comfyui | (system python) | Pre-baked in base image | ~7-18 GB |

## Pull-Model Architecture

The Go supervisor uses PostgreSQL row-level locking for job claiming, connecting to the
desktop's PostgreSQL instance via a socat tunnel over Tailscale VPN.

```mermaid
sequenceDiagram
    participant SUP as Go Supervisor<br/>(RunPod pod)
    participant DB as PostgreSQL<br/>(Desktop Docker<br/>via socat + Tailscale)

    Note over SUP: Pod starts → entrypoint.sh
    SUP->>SUP: Start Tailscale + socat tunnel
    SUP->>SUP: Start ComfyUI + handler subprocesses
    SUP->>DB: INSERT worker_registry (state='starting')
    SUP->>DB: UPDATE state='ready'

    par Heartbeat Loop (every 30s)
        loop Every 30 seconds
            SUP->>DB: UPDATE last_heartbeat=NOW()
        end
    and Job Polling Loop (every 3s)
        loop Every 3 seconds
            SUP->>DB: claim_next_job()<br/>SELECT job WHERE status='queued'<br/>FOR UPDATE SKIP LOCKED
            alt Job available
                DB-->>SUP: Job row (assigned)
                SUP->>SUP: Download inputs via HTTP<br/>Process with handler<br/>Upload outputs via HTTP
                SUP->>DB: UPDATE status='completed'
            else No job
                DB-->>SUP: NULL
                Note over SUP: Sleep 3s
            end
        end
    and Health Monitor (every 30s)
        loop Every 30 seconds
            SUP->>SUP: GET /health for each handler<br/>Restart any failed handlers
        end
    end
```

## Handler Contract

All handlers in `src/workers/handlers/` implement the same interface, served via
`handler_server.py` as HTTP endpoints.

```mermaid
classDiagram
    class Handler {
        <<interface>>
        +init() None
        +handle(input_path: str, output_dir: str, **kwargs) dict~str, Any~
    }

    class TranscriptionHandler {
        +init() Load Whisper model to GPU
        +handle() Transcribe audio with Whisper
    }

    class DiarizationHandler {
        +init() Load pyannote model to GPU
        +handle() Speaker diarization
    }

    class FaceExtractionHandler {
        +init() Load InsightFace model
        +handle() Detect faces, extract embeddings
    }

    class VisualTaggingHandler {
        +init() Load ViT model to GPU
        +handle() Tag image frames
    }

    class ComfyUIHandler {
        +handle() Submit workflow, poll outputs
    }

    Handler <|-- TranscriptionHandler
    Handler <|-- DiarizationHandler
    Handler <|-- FaceExtractionHandler
    Handler <|-- VisualTaggingHandler
    Handler <|-- ComfyUIHandler
```

### Handler HTTP Protocol

```mermaid
flowchart TD
    START[Job claimed from job_queue]
    DL["Download inputs\nGET /api/cache/{jobId}/input/{key}\nvia Tailscale to desktop API"]
    INIT[handler.init()<br/>Load models if needed]
    HANDLE[POST /handle to handler subprocess<br/>input_path, output_dir, job metadata]

    subgraph "Handler Implementation"
        TRANS[Transcription<br/>Whisper turbo GPU]
        DIAR[Diarization<br/>pyannote GPU]
        FACE[Face Extraction<br/>InsightFace GPU]
        VTAG[Visual Tagging<br/>ViT-B/16 GPU]
        COMFY[ComfyUI Workflow<br/>Submit /prompt, poll /history]
    end

    UL["Upload outputs\nPUT /api/cache/{jobId}/output/{key}\nvia Tailscale to desktop API"]
    REGISTER[INSERT cache_manifest<br/>direction='output']
    COMPLETE[UPDATE job_queue<br/>status='completed']

    START --> DL
    DL --> INIT
    INIT --> HANDLE
    HANDLE --> TRANS
    HANDLE --> DIAR
    HANDLE --> FACE
    HANDLE --> VTAG
    HANDLE --> COMFY
    TRANS --> UL
    DIAR --> UL
    FACE --> UL
    VTAG --> UL
    COMFY --> UL
    UL --> REGISTER
    REGISTER --> COMPLETE

    style DL fill:#1a3a4a,stroke:#00bcd4,color:#e0f7fa
    style UL fill:#1a3a4a,stroke:#00bcd4,color:#e0f7fa
    style TRANS fill:#e3f2fd,stroke:#2196f3
    style DIAR fill:#e3f2fd,stroke:#2196f3
    style FACE fill:#e3f2fd,stroke:#2196f3
    style VTAG fill:#e3f2fd,stroke:#2196f3
    style COMFY fill:#4285f4,stroke:#1a73e8,color:#fff
```

## Storage Access Patterns

```mermaid
graph TB
    subgraph "RunPod Pod"
        POD_CACHE["/data/io-cache\n(ephemeral pod disk)"]
        NET_VOL["/workspace\n(RunPod Network Volume)\nComfyUI models"]
    end

    subgraph "Desktop"
        API["C# API :6000\nCacheTransferController"]
        DESK_CACHE["/data/io-cache\n(Docker volume)"]
        DESK_ART["/data/neovlab_artifacts\n(permanent storage)"]
    end

    subgraph "Tailscale VPN"
        VPN["WireGuard mesh\nuserspace networking"]
    end

    POD_CACHE -->|"HTTP PUT via VPN\nupload outputs"| VPN
    VPN -->|"to API :6000"| API
    API -->|"write"| DESK_CACHE
    DESK_CACHE -->|"ArtifactSyncService\nFile.Copy"| DESK_ART

    API -->|"HTTP GET via VPN\ndownload inputs"| VPN
    VPN -->|"to pod"| POD_CACHE

    style POD_CACHE fill:#2a1a2a,stroke:#e91e63,color:#fce4ec
    style NET_VOL fill:#3a2a1a,stroke:#ff6f00,color:#ffe0b2
    style VPN fill:#1a3a1a,stroke:#4caf50,color:#c8e6c9
    style DESK_CACHE fill:#e8f5e9,stroke:#4caf50
    style DESK_ART fill:#e8f5e9,stroke:#4caf50
```

## Configuration Environment Variables

### RunPod Pod

| Variable | Purpose | Example |
|----------|---------|---------|
| `DATABASE_URL` | PostgreSQL via socat tunnel | `postgresql://neovlab:pw@localhost:15432/cluster` |
| `DB_TAILSCALE_HOST` | Desktop Tailscale IP for socat target | `100.84.81.75` |
| `NEOVLAB_API_BASE_URL` | Desktop API for file transfer | `http://100.84.81.75:6000` |
| `TAILSCALE_AUTHKEY` | Reusable ephemeral Tailscale key | `tskey-auth-...` |
| `HUGGINGFACE_TOKEN` | HuggingFace token for pyannote | `hf_...` |
| `WORKER_TOKEN` | Auth token (match API config) | `secret123` |
| `ENABLED_HANDLERS` | Comma-separated handler list | `comfyui,transcription,...` |
| `IO_CACHE_DIR` | Artifact storage path | `/data/io-cache` |
| `MODELS_DIR` | Network volume model path | `/workspace/comfyui` |
| `POLL_INTERVAL_SECONDS` | Job polling frequency | `3` |
| `LEASE_DURATION_MINUTES` | Job lease timeout | `10` |

### Desktop API (appsettings.json)

| Config Key | Purpose | Example |
|------------|---------|---------|
| `Gpu:Platform` | Platform adapter | `RunPod` |
| `Gpu:RunPod:ApiKey` | RunPod API key | (or `runpod_key` env var) |
| `Gpu:RunPod:PodName` | Pod name for ID resolution | `neovnext-worker` |
| `Gpu:RunPod:WorkerToken` | Token for CacheTransferController | `secret123` |

## Job Claiming Strategy

The pull-model uses PostgreSQL row-level locking for job claiming:

```sql
-- claim_next_job() in Go supervisor (internal/db/)
WITH next_job AS (
    SELECT job_id
    FROM job_queue
    WHERE (
        status = 'queued'
        OR (
            status IN ('assigned', 'running')
            AND lease_expires_at < NOW()  -- Reclaim expired leases
        )
    )
    AND worker_function = $1  -- e.g., 'transcription'
    ORDER BY priority DESC, created_at ASC
    LIMIT 1
    FOR UPDATE SKIP LOCKED  -- Skip if another worker already claimed
)
UPDATE job_queue
SET status = 'assigned',
    assigned_worker_id = $2,
    lease_expires_at = NOW() + INTERVAL '10 minutes',
    updated_at = NOW()
FROM next_job
WHERE job_queue.job_id = next_job.job_id
RETURNING *;
```

---

**See Also:** [gpu-supervisor.md](gpu-supervisor.md) for Go supervisor internals, [runpod-infrastructure.md](runpod-infrastructure.md) for infrastructure details
