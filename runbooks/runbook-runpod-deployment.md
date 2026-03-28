# RunPod GPU Worker Deployment Runbook

**Date:** 2026-03-28
**Status:** Operational
**Image:** `tagowar/neovnext-gpu-worker:v1.0.14`

---

## Overview

This runbook documents the NeoVNext GPU worker deployment on RunPod. The worker runs the Go supervisor which manages ComfyUI and Python ML handler subprocesses (transcription, diarization, face extraction, visual tagging). It connects to the local PostgreSQL cluster database via Tailscale VPN.

---

## Architecture

```
RunPod Pod (GPU)
├── /entrypoint.sh
│   ├── sshd (port 22, TCP)
│   ├── FileBrowser (port 8080, HTTP)
│   ├── JupyterLab (port 8888, HTTP)
│   ├── tailscaled --tun=userspace-networking
│   ├── socat tunnel: localhost:15432 → tailscale nc → desktop:5432
│   └── /usr/local/bin/supervisor
│       ├── ComfyUI (port 8188, HTTP) — /comfyui → /opt/comfyui-baked
│       ├── transcription handler (port 9001) — /venvs/transcription/bin/python
│       ├── diarization handler (port 9002) — /venvs/diarization/bin/python
│       ├── face_extraction handler (port 9003) — /venvs/face/bin/python
│       └── visual_tagging handler (port 9004) — /venvs/visual-tagging/bin/python
│
└── Tailscale mesh → Desktop (100.84.81.75)
                      └── Docker: postgres:5432 (pgvector/pgvector:pg16)
```

### Key Design Decisions

- **Base image:** `runpod/comfyui:cuda13.0` — provides Python 3.12, CUDA 13.0, torch 2.10+cu130, ComfyUI pre-baked at `/opt/comfyui-baked/`, SSH, FileBrowser, JupyterLab
- **Per-handler venvs:** Each ML handler runs in an isolated venv under `/venvs/<name>/` with `--system-site-packages` to inherit system torch/numpy. Eliminates CUDA version conflicts from the old monolithic pip install.
- **Tailscale userspace networking:** RunPod pods don't expose `/dev/net/tun`, so kernel-mode Tailscale isn't available. We use `--tun=userspace-networking` which doesn't add OS routes, requiring a `socat` TCP bridge for Postgres connectivity.
- **socat Postgres tunnel:** `socat TCP-LISTEN:15432 EXEC:"tailscale nc $DB_TAILSCALE_HOST 5432"` bridges localhost:15432 → Tailscale → desktop Postgres. `DATABASE_URL` uses `localhost:15432`.
- **API file transfer:** The Go supervisor downloads job input files from the C# API (`NEOVLAB_API_BASE_URL=http://<tailscale-ip>:6000`) before passing local paths to Python handlers. Without this env var the supervisor falls back to filesystem mode and passes the Windows `cache_path` stored in the DB, which Linux handlers cannot open. The Windows host must expose port 6000 via `setup-portproxy.ps1` and `setup-tailscale-firewall.ps1`.

---

## Docker Image

**Registry:** Docker Hub (public)
**Repository:** `tagowar/neovnext-gpu-worker`
**Dockerfile:** `docker/worker-gpu-go-runpod.Dockerfile`

### Build & Push

```bash
# From repo root
docker build --platform linux/amd64 \
  -f docker/worker-gpu-go-runpod.Dockerfile \
  -t tagowar/neovnext-gpu-worker:v<VERSION> .

docker push tagowar/neovnext-gpu-worker:v<VERSION>
```

### Version History

| Version | Change |
|---------|--------|
| v1.0.0 | Initial build (nvidia/cuda:12.1.1 base, monolithic pip) |
| v1.0.1 | Fix CRLF line endings on entrypoint.sh |
| v1.0.2 | Fix tailscaled startup race (poll instead of sleep 2) |
| v1.0.3 | Make Tailscale failures non-fatal (|| true) |
| v1.0.4 | Switch to userspace Tailscale (--tun=userspace-networking) |
| v1.0.5 | Rebase onto runpod/comfyui:cuda13.0, per-handler venvs, socat Postgres tunnel |
| v1.0.6 | pyannote.audio>=3.4,<4 (torch 2.10 weights_only compat), lifespan handler_server |
| v1.0.7 | Patch pyannote io.py for torchaudio 2.10 (AudioMetaData removed) |
| v1.0.8 | Fix pyannote use_auth_token, silence tqdm/uvicorn logs, clean job logging |
| v1.0.9 | Fix pyannote HF auth: set HF_TOKEN env var instead of passing to from_pretrained |
| v1.0.10 | Dynamic io.py patch path (immune to Python version changes), build-time smoke test, logger.exception in init() |
| v1.0.11 | Pin huggingface_hub<0.24 in diarization venv (pyannote use_auth_token crash); entrypoint mkdir all ComfyUI model subdirs so ComfyUI-Manager can't fail on missing diffusion_models/ |
| v1.0.12 | Fix video artifact type: downloadArtifacts reads Content-Type from /view response, saves .mp4/.png/etc instead of .bin; add ipadapter/clip/style_models/photomaker/insightface model subdirs; fix verifyModelMounts to check dir not hardcoded file |
| v1.0.13 | Fix diarization venv: pin transformers>=4.40.0 instead of huggingface_hub<0.24 |
| v1.0.14 | Add --gpu-only to ComfyUI launch (prevents dynamic VRAM loading on H100); diarization loads from local /workspace/pyannote cache if present; fix duplicate failJobSafe method in supervisor |

---

## Pod Configuration

### Container Image
```
tagowar/neovnext-gpu-worker:v1.0.13
```

### Ports

| Port | Protocol | Service |
|------|----------|---------|
| 8080 | HTTP | FileBrowser |
| 8188 | HTTP | ComfyUI |
| 8888 | HTTP | JupyterLab |
| 22 | TCP | SSH |

### Environment Variables

| Variable | Source | Value | Notes |
|----------|--------|-------|-------|
| `TAILSCALE_AUTHKEY` | Secret | `tskey-auth-...` | Reusable ephemeral key |
| `DATABASE_URL` | Secret | `postgresql://neovlab:<pw>@localhost:15432/cluster` | Uses socat tunnel port |
| `DB_TAILSCALE_HOST` | Plain | `100.84.81.75` | Desktop Tailscale IP |
| `NEOVLAB_API_BASE_URL` | Plain | `http://100.84.81.75:6000` | API server for file transfer (inputs/outputs) |
| `HUGGINGFACE_TOKEN` | Secret | `hf_...` | Required for pyannote diarization |
| `PUBLIC_KEY` | Plain | `ssh-ed25519 ...` | SSH access key |
| `MODELS_DIR` | Plain | `/workspace/comfyui` | Network volume path |

### Network Volume
- **Mount path:** `/workspace`
- **Models:** `/workspace/comfyui/` (symlinked to `/comfyui/models` at startup)
- **Required subdirs:** `checkpoints/`, `clip_vision/`, `text_encoders/`, `vae/`, `loras/`, `diffusion_models/wan/`

---

## Local Infrastructure

### PostgreSQL (Docker)

```bash
# Start
docker compose -f docker/docker-compose.local-infra.yml up -d

# Image: pgvector/pgvector:pg16 (required for vector extension)
# Port: 0.0.0.0:5432 (all interfaces for Tailscale access)
# DB: cluster, User: neovlab, Password: neovlab-local (dev)
```

### Windows Firewall + Port Proxy (run as Administrator)

```powershell
# Allow Tailscale traffic to reach Postgres and API
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/setup-tailscale-firewall.ps1

# Bridge Tailscale IP to Docker's localhost ports
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/setup-portproxy.ps1
```

These must be re-run after system restarts.

### Tailscale Auth Key

Generate at: https://login.tailscale.com/admin/settings/keys
Settings: **Reusable** + **Ephemeral**
Store as a RunPod Secret named `tailscale_authkey`.

---

## Database Schema

9 migrations applied to `cluster` database:

| Migration | Tables Added |
|-----------|-------------|
| 001 | job_queue, worker_registry, cache_manifest |
| 002 | cache_manifest.gcs_evicted_at |
| 003 | thumbnails |
| 004 | staged_inputs |
| 005 | face_detections (pgvector 512-dim) |
| 006 | transcriptions (FTS tsvector) |
| 007 | speaker_segments (pgvector 192-dim) |
| 008 | handler_config |
| 009 | pipeline_jobs, pipeline_child_jobs, persons |

Apply migrations:
```bash
for f in src/data/migrations/cluster/*.sql; do
  docker exec -i neovlab-cluster-db psql -U neovlab -d cluster < "$f"
done
```

---

## Operations

### Pod Management Scripts

```powershell
# Check pod status and SSH port
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/pod-status.ps1

# Restart pod
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/pod-restart.ps1
```

### SSH Into Pod

```bash
# Get current IP/port from pod-status.ps1 first
ssh -i ~/.ssh/runpod_neovnext -p <port> root@<ip>

# Key location: ~/.ssh/runpod_neovnext
```

### Iterative Deployment (no Docker rebuild)

For Go supervisor or Python handler changes without dependency changes:

```bash
# Build supervisor binary locally
cd src/gpu-supervisor
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o supervisor-linux ./cmd/supervisor

# Copy to running pod
scp -P <port> supervisor-linux root@<ip>:/usr/local/bin/supervisor
scp -P <port> src/workers/handlers/foo.py root@<ip>:/app/handlers/

# Restart supervisor on pod
ssh -p <port> root@<ip> "pkill supervisor; /usr/local/bin/supervisor &"
```

---

## Troubleshooting

### Pod crash-loops on DB connection failure
The supervisor exits fatally if it can't connect to Postgres on startup. Check:
1. Tailscale is authenticated (`TAILSCALE_AUTHKEY` set correctly)
2. `DB_TAILSCALE_HOST` is set to the desktop's Tailscale IP
3. `DATABASE_URL` uses `localhost:15432` (not the Tailscale IP directly)
4. Windows port proxy is running (`setup-portproxy.ps1`)
5. Windows firewall allows port 5432 on Tailscale interface (`setup-tailscale-firewall.ps1`)
6. Docker Postgres is running (`docker ps | grep cluster-db`)

### Tailscale auth key invalid/revoked
Generate a new **Reusable** key at https://login.tailscale.com/admin/settings/keys and update the RunPod secret. Keys get auto-revoked if exposed as plaintext — always use RunPod Secrets.

### Diarization handler fails to start
`HUGGINGFACE_TOKEN` env var is required. Token is stored in `z:/neocomfy/neocomfy.db` under key `hf_token`.

### Handler returns 500: "Protocol not found" / Windows path sent to handler
Symptom: `ffmpeg` error `Error opening input: Protocol not found` with a path like `Z:\NeoVNext...`.

Root cause: `NEOVLAB_API_BASE_URL` is not set on the pod. Without it the supervisor falls back to filesystem mode and sends the Windows `cache_path` from the database directly to Linux handlers.

Fix:
1. Add `NEOVLAB_API_BASE_URL=http://<desktop-tailscale-ip>:6000` to the pod's environment variables
2. Re-run `setup-portproxy.ps1` and `setup-tailscale-firewall.ps1` on the Windows host (as Administrator) so port 6000 is reachable via Tailscale
3. Restart the pod

### Models directory not found
Attach a RunPod Network Volume. Create `/workspace/comfyui/` with the required model subdirectories and populate with model weights.
