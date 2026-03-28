# Infrastructure Topology

> **Note:** GPU workloads have migrated from GCP Cloud Run to RunPod. This document
> reflects the current RunPod + local infrastructure architecture. For legacy GCP
> documentation, see `docs/archive/`.

## RunPod + Desktop Architecture

```mermaid
graph TB
    subgraph "RunPod Cloud"
        subgraph "GPU Pod"
            POD["RunPod Pod\nGPU: configurable (L4, A40, H100, etc.)\nImage: tagowar/neovnext-gpu-worker:v1.0.14\nBase: runpod/comfyui:cuda13.0"]
            SUP["Go Supervisor\n/usr/local/bin/supervisor"]
            COMFY["ComfyUI :8188\n/opt/comfyui-baked/"]
            HANDLERS["Python Handlers\n:9001-9004\n(per-handler venvs)"]
            TS_POD["Tailscale\nuserspace networking"]
            SOCAT["socat tunnel\nlocalhost:15432"]
        end

        NET_VOL[("Network Volume\n/workspace\nComfyUI models\ncheckpoints, loras, VAE, etc.")]
    end

    subgraph "Desktop (Windows)"
        subgraph "Docker Containers"
            PG[("PostgreSQL\npgvector/pgvector:pg16\n0.0.0.0:5432\nDB: cluster")]
        end

        subgraph "Native Processes"
            API["C# API :6000\n(.NET 9)"]
            ELECTRON["Electron :6100\n(React frontend)"]
        end

        subgraph "Local Storage"
            SQLITE[("SQLite\nneovlab.db")]
            IO_CACHE[("io-cache\n/data/io-cache")]
            ARTIFACTS[("artifacts\n/data/neovlab_artifacts")]
        end

        TS_DESK["Tailscale\n100.84.81.75"]
    end

    subgraph "Tailscale Mesh VPN"
        VPN["WireGuard tunnel\nEncrypted peer-to-peer"]
    end

    POD --> SUP
    SUP --> COMFY
    SUP --> HANDLERS
    POD --> NET_VOL

    TS_POD ---|"WireGuard"| VPN
    VPN ---|"WireGuard"| TS_DESK

    SOCAT -->|"tailscale nc\n→ desktop:5432"| VPN
    SUP -->|"HTTP GET/PUT\n/api/cache/..."| VPN

    VPN -->|":5432"| PG
    VPN -->|":6000"| API

    API --> PG
    API --> SQLITE
    API --> IO_CACHE
    API --> ARTIFACTS
    ELECTRON -->|"localhost:6000"| API

    style POD fill:#2a1a2a,stroke:#e91e63,color:#fce4ec
    style SUP fill:#00897b,stroke:#004d40,color:#fff
    style COMFY fill:#4285f4,stroke:#1a73e8,color:#fff
    style HANDLERS fill:#e3f2fd,stroke:#2196f3
    style PG fill:#ea4335,stroke:#c5221f,color:#fff
    style API fill:#1a3a4a,stroke:#00bcd4,color:#e0f7fa
    style ELECTRON fill:#1a3a4a,stroke:#00bcd4,color:#e0f7fa
    style NET_VOL fill:#3a2a1a,stroke:#ff6f00,color:#ffe0b2
    style VPN fill:#1a3a1a,stroke:#4caf50,color:#c8e6c9
```

## Network Data Flow

```mermaid
sequenceDiagram
    participant Entry as entrypoint.sh
    participant TS as Tailscale
    participant Socat as socat tunnel
    participant SUP as Go Supervisor
    participant Comfy as ComfyUI
    participant Handlers as Python Handlers
    participant PG as PostgreSQL<br/>(Desktop Docker)
    participant API as C# API<br/>(Desktop :6000)

    Note over Entry,TS: Pod Startup (~30-60s)
    Entry->>Entry: Start sshd, FileBrowser, JupyterLab
    Entry->>TS: tailscaled --tun=userspace-networking
    Entry->>TS: tailscale up --authkey=$TAILSCALE_AUTHKEY
    TS-->>Entry: Connected to mesh
    Entry->>Socat: socat TCP-LISTEN:15432<br/>EXEC:"tailscale nc $DB_TAILSCALE_HOST 5432"

    Note over SUP,PG: Supervisor Bootstrap
    Entry->>SUP: exec /usr/local/bin/supervisor
    SUP->>SUP: Symlink /workspace/comfyui → /comfyui/models
    SUP->>Comfy: exec python3 /comfyui/main.py --gpu-only
    SUP->>Comfy: Poll GET /system_stats (up to 300s)
    Comfy-->>SUP: 200 OK (ready)

    par Start handlers
        SUP->>Handlers: exec /venvs/*/bin/python handler_server.py
    end
    Handlers-->>SUP: Health checks pass

    SUP->>PG: pgxpool.Connect() via localhost:15432<br/>→ socat → Tailscale → desktop:5432
    SUP->>PG: INSERT worker_registry (state='ready')

    Note over SUP,API: Job Processing Loop
    loop Every 3s
        SUP->>PG: claim_next_job() FOR UPDATE SKIP LOCKED
        alt Job found
            SUP->>API: GET /api/cache/{jobId}/input/{key}<br/>(via Tailscale)
            API-->>SUP: binary stream (input file)
            SUP->>Handlers: POST /handle {input_path, output_dir}
            Handlers-->>SUP: result JSON
            SUP->>API: PUT /api/cache/{jobId}/output/{key}<br/>(via Tailscale)
            API-->>SUP: 200 OK
            SUP->>PG: INSERT cache_manifest + UPDATE status='completed'
        end
    end
```

## Storage Hierarchy

```mermaid
graph LR
    subgraph "RunPod Pod"
        POD_CACHE["/data/io-cache\n(ephemeral pod disk)\nJob inputs + outputs during processing"]
        POD_MODELS["/workspace/comfyui\n(Network Volume)\nComfyUI model weights"]
        POD_COMFY["/comfyui → /opt/comfyui-baked\n(baked in Docker image)\nComfyUI application code"]
    end

    subgraph "Desktop (Docker volumes)"
        DESK_CACHE["/data/io-cache\n(io-cache volume)\nStaged inputs + uploaded outputs"]
        DESK_ART["/data/neovlab_artifacts\n(artifact-data volume)\nPermanent artifact storage"]
    end

    POD_CACHE -->|"HTTP PUT\nvia Tailscale"| DESK_CACHE
    DESK_CACHE -->|"File.Copy\n(ArtifactSyncService)"| DESK_ART

    style POD_CACHE fill:#2a1a2a,stroke:#e91e63,color:#fce4ec
    style POD_MODELS fill:#3a2a1a,stroke:#ff6f00,color:#ffe0b2
    style POD_COMFY fill:#263238,stroke:#78909c,color:#b0bec5
    style DESK_CACHE fill:#e8f5e9,stroke:#4caf50
    style DESK_ART fill:#e8f5e9,stroke:#4caf50
```

## Cost Comparison (RunPod vs. former GCP)

| Resource | RunPod (current) | GCP Cloud Run (former) |
|----------|-----------------|----------------------|
| **GPU compute** | ~$0.50-0.74/hr (L4/A40) | $0.65/hr (L4) + overhead |
| **Idle cost** | $0/hr (pod stopped) | $0/hr (scale-to-zero) |
| **Storage** | Network Volume: $0.07/GB/mo | Filestore NFS: $40.96/mo (1 TiB min) |
| **Database** | Local Docker PG: $0 | Cloud SQL: $14.20/mo |
| **Networking** | Tailscale (free tier) | VPC + Private Google Access |
| **Typical monthly** | **~$30-50** (light use) | **~$320** (light use) |

## Key Configuration

### RunPod Pod

| Setting | Value |
|---------|-------|
| Docker Image | `tagowar/neovnext-gpu-worker:v1.0.14` |
| GPU | Configurable per pod (L4, A40, H100) |
| Network Volume | `/workspace` (ComfyUI models) |
| Exposed Ports | 22 (SSH), 8080 (FileBrowser), 8188 (ComfyUI), 8888 (JupyterLab) |

### Desktop Infrastructure

| Component | Details |
|-----------|---------|
| PostgreSQL | `pgvector/pgvector:pg16` in Docker, port 5432 (all interfaces) |
| C# API | .NET 9, port 6000, with CacheTransferController |
| Electron Frontend | React, port 6100 |
| Tailscale | IP 100.84.81.75 |
| Port Proxy | `setup-portproxy.ps1` bridges Tailscale IP to Docker ports |
| Firewall | `setup-tailscale-firewall.ps1` allows Tailscale traffic |

### Database Schema (9 migrations)

| Migration | Tables |
|-----------|--------|
| 001 | job_queue, worker_registry, cache_manifest |
| 002 | cache_manifest.gcs_evicted_at |
| 003 | thumbnails |
| 004 | staged_inputs |
| 005 | face_detections (pgvector 512-dim) |
| 006 | transcriptions (FTS tsvector) |
| 007 | speaker_segments (pgvector 192-dim) |
| 008 | handler_config |
| 009 | pipeline_jobs, pipeline_child_jobs, persons |

---

**See Also:** [worker-architecture.md](worker-architecture.md) for handler details, [gpu-supervisor.md](gpu-supervisor.md) for Go supervisor internals
