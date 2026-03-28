# NeoVNext RunPod Worker — From-Scratch Setup Runbook

**Last updated:** 2026-03-28
**Platform:** Windows 11 + Docker Desktop + Tailscale + RunPod

This runbook covers everything needed to go from a clean machine to a fully operational
NeoVNext deployment: local Postgres, Tailscale VPN, Windows firewall/port-proxy, RunPod
pod, and the local API/frontend stack.

---

## Prerequisites

| Tool | Where to get it |
|------|----------------|
| Docker Desktop | https://www.docker.com/products/docker-desktop/ |
| Tailscale | https://tailscale.com/download |
| .NET 9 SDK | https://dotnet.microsoft.com/download |
| Node.js 20+ | https://nodejs.org/ |
| Go 1.22+ | https://go.dev/dl/ (only needed for supervisor builds) |
| RunPod account | https://runpod.io |

Environment variables expected on the Windows host (set in user/system env or a `.env` loader):

| Variable | Value |
|----------|-------|
| `runpod_key` | RunPod API key (GraphQL Read+Write) |
| `CLUSTER_DB_PASSWORD` | Postgres password (optional, defaults to `neovlab-local`) |

---

## Part 1 — Local PostgreSQL

### 1.1 Start the container

```powershell
docker compose -f docker/docker-compose.local-infra.yml up -d
```

Or use the helper script:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/start-local-infra.ps1
```

- **Image:** `pgvector/pgvector:pg16` (required — standard postgres doesn't have the vector extension)
- **Port:** `0.0.0.0:5432` (all interfaces, so Tailscale traffic can reach it)
- **DB:** `cluster` | **User:** `neovlab` | **Password:** `neovlab-local` (default)
- **Volume:** `neovlab-cluster-data` (persists across restarts)

### 1.2 Apply migrations

Run all 9 migrations in order:

```bash
for f in src/data/migrations/cluster/*.sql; do
  docker exec -i neovlab-cluster-db psql -U neovlab -d cluster < "$f"
done
```

Or on Windows:

```powershell
Get-ChildItem src/data/migrations/cluster/*.sql | Sort-Object Name | ForEach-Object {
    Write-Host "Applying $($_.Name)..."
    Get-Content $_.FullName | docker exec -i neovlab-cluster-db psql -U neovlab -d cluster
}
```

Verify:

```powershell
docker exec neovlab-cluster-db psql -U neovlab -d cluster -c "\dt"
```

Expected: `job_queue`, `worker_registry`, `cache_manifest`, `thumbnails`, `staged_inputs`,
`face_detections`, `transcriptions`, `speaker_segments`, `handler_config`,
`pipeline_jobs`, `pipeline_child_jobs`, `persons`

---

## Part 2 — Tailscale VPN

Tailscale creates the secure tunnel between the RunPod GPU pod and your desktop Postgres.

### 2.1 Install and authenticate Tailscale

Download and install from https://tailscale.com/download. Sign in via the UI.

Note your desktop's Tailscale IP:

```powershell
tailscale ip -4
```

This is `DB_TAILSCALE_HOST` — the address the RunPod pod will use to reach your desktop.
Example: `100.84.81.75`

### 2.2 Windows Firewall rules (run as Administrator)

Creates inbound rules that allow Postgres (5432) and API (5000) traffic **only** on the
Tailscale interface — not exposed to the public internet.

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/setup-tailscale-firewall.ps1
```

This creates:
- `NeoVLab PostgreSQL via Tailscale` — TCP 5432 inbound on Tailscale adapter
- `NeoVLab API via Tailscale` — TCP 5000 inbound on Tailscale adapter

### 2.3 Windows port proxy (run as Administrator)

Docker Desktop's NAT doesn't forward Tailscale traffic to containers automatically.
A `netsh portproxy` rule bridges the gap:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/setup-portproxy.ps1
```

This creates:
- `<tailscale-ip>:5432` → `127.0.0.1:5432` (Postgres in Docker)
- `<tailscale-ip>:5000` → `127.0.0.1:5000` (API)

**Important:** These rules are lost on system restart. Re-run after every reboot.

### 2.4 Verify connectivity

From PowerShell:

```powershell
$ip = (tailscale ip -4).Trim()
Test-NetConnection $ip -Port 5432   # Should show TcpTestSucceeded: True
```

If it fails:
1. Check Docker is running and container is healthy: `docker ps`
2. Re-run `setup-portproxy.ps1`
3. Re-run `setup-tailscale-firewall.ps1`
4. Check Tailscale is authenticated: `tailscale status`

---

## Part 3 — RunPod Pod

### 3.1 Create a Tailscale auth key

Go to https://login.tailscale.com/admin/settings/keys

- Click **Generate auth key**
- Enable: **Reusable** + **Ephemeral**
- Copy the key (`tskey-auth-...`)
- Store it as a RunPod Secret named `tailscale_authkey` (never paste in plaintext env vars — it will get auto-revoked)

### 3.2 Create the pod

In RunPod UI → **Pods** → **Deploy**:

**Container Image:**
```
tagowar/neovnext-gpu-worker:v1.0.7
```

**GPU:** Any NVIDIA GPU with 16GB+ VRAM (L4, A4000, 3090, etc.)

**Expose Ports:**
| Port | Type | Purpose |
|------|------|---------|
| 22 | TCP | SSH |
| 8080 | HTTP | FileBrowser |
| 8188 | HTTP | ComfyUI |
| 8888 | HTTP | JupyterLab |

> Note: Do NOT expose 5432. Postgres traffic goes through Tailscale only.

**Network Volume:** Attach a volume at `/workspace` with at least 100GB for models.

### 3.3 Environment variables

Set these in the pod configuration. Use **RunPod Secrets** for sensitive values.

| Variable | Type | Value | Notes |
|----------|------|-------|-------|
| `TAILSCALE_AUTHKEY` | Secret | `tskey-auth-...` | From step 3.1 — must be Secret |
| `DATABASE_URL` | Secret | `postgresql://neovlab:neovlab-local@localhost:15432/cluster` | Uses socat tunnel port 15432 |
| `DB_TAILSCALE_HOST` | Plain | `100.84.81.75` | Your desktop's Tailscale IP from step 2.1 |
| `HUGGINGFACE_TOKEN` | Secret | `hf_...` | Required for pyannote diarization model |
| `PUBLIC_KEY` | Plain | `ssh-ed25519 ...` | Your SSH public key for pod access |
| `MODELS_DIR` | Plain | `/workspace/comfyui` | Path on network volume |

**Critical:** `DATABASE_URL` must use `localhost:15432` (the socat tunnel port), NOT the
Tailscale IP directly. Userspace Tailscale doesn't add OS routes, so direct TCP to
100.x.x.x addresses fails. The socat tunnel bridges it.

### 3.4 SSH key setup

If you don't have an SSH key:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/runpod_neovnext
```

Paste the contents of `~/.ssh/runpod_neovnext.pub` into the `PUBLIC_KEY` env var.

### 3.5 Model directory setup (first time only)

SSH into the pod after it starts, then create the required subdirectories:

```bash
# Get IP/port first
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/pod-status.ps1

ssh -i ~/.ssh/runpod_neovnext -p <port> root@<ip>

# On the pod:
mkdir -p /workspace/comfyui/{checkpoints,clip_vision,text_encoders,vae,loras,diffusion_models/wan}
```

---

## Part 4 — Local API + Frontend

### 4.1 Create the data directory

The C# API needs this to exist before first run (it creates the SQLite DB inside it):

```bash
mkdir -p src/backend/NeoVLab.Api/data/io-cache
mkdir -p src/backend/NeoVLab.Api/logs
```

### 4.2 Configure user secrets (one-time)

```bash
cd src/backend/NeoVLab.Api

dotnet user-secrets set "Gpu:Platform" "RunPod"
dotnet user-secrets set "Gpu:Enabled" "true"
dotnet user-secrets set "Gpu:RunPod:PodId" "<pod_id>"   # from pod-status.ps1
```

The API key is read from the `runpod_key` environment variable automatically (no secret needed if it's set in your environment).

### 4.3 Install frontend dependencies (first time)

```bash
cd src/frontend
npm install
```

### 4.4 Start everything

```powershell
.\Start-NeoVLab.ps1
```

Or individually:

```powershell
.\Start-NeoVLab.ps1 start api       # http://localhost:6000
.\Start-NeoVLab.ps1 start frontend  # http://localhost:6100
.\Start-NeoVLab.ps1 start desktop   # WPF window
```

---

## Part 5 — Verify End-to-End

### 5.1 Check pod is up

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/pod-status.ps1
```

Expected: `Status: RUNNING`, ports listed with public IP.

### 5.2 SSH into pod and check logs

```bash
ssh -i ~/.ssh/runpod_neovnext -p <port> root@<ip>

# Check supervisor is running
ps aux | grep supervisor

# Check handler logs
journalctl -f   # or check /workspace/runpod-slim/ logs
```

All four handlers should report `Handler <name> ready`:
```
transcription handler ready
diarization handler ready
face_extraction handler ready
visual_tagging handler ready
```

### 5.3 Check Tailscale on the pod

```bash
tailscale status     # Should show your desktop as a peer
tailscale ping 100.84.81.75   # Should get replies
```

If Tailscale shows "not authenticated":
- Check `TAILSCALE_AUTHKEY` is set as a Secret (not plain env)
- Generate a new Reusable+Ephemeral key and update the Secret

### 5.4 Check DB connectivity on the pod

```bash
psql postgresql://neovlab:neovlab-local@localhost:15432/cluster -c "\dt"
```

If this fails:
1. Check `socat` is running: `ps aux | grep socat`
2. Check `DB_TAILSCALE_HOST` is set to the correct Tailscale IP
3. Verify Tailscale is authenticated (step 5.3)
4. On Windows: re-run `setup-portproxy.ps1` (may have been lost on reboot)

### 5.5 Check API GPU status

Open http://localhost:6000/api/gpu/status — should return pod status JSON.

---

## Part 6 — After System Restart

After rebooting the Windows host, re-run these (both require Administrator):

```powershell
# Restore port proxy rules (lost on reboot)
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/setup-portproxy.ps1

# Start local Postgres
docker compose -f docker/docker-compose.local-infra.yml up -d
```

Tailscale and Docker Desktop start automatically on boot, but the port proxy rules do not persist.

---

## Troubleshooting Reference

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Pod crash-loops immediately | Bad entrypoint / CRLF | Check image version is v1.0.7+ |
| `tailscale up` fails on pod | Auth key invalid/expired | Generate new Reusable key, update RunPod Secret |
| `password authentication failed` | Wrong DB password in URL | `DATABASE_URL` password must match `CLUSTER_DB_PASSWORD` (default: `neovlab-local`) |
| DB connection timeout from pod | Tailscale not routing / port proxy missing | Re-run `setup-portproxy.ps1` on Windows, verify `tailscale ping` from pod |
| Diarization handler fails | `HUGGINGFACE_TOKEN` not set | Add as RunPod Secret |
| `GPU warm-pool` logs in API | Old GCloud user secret | `dotnet user-secrets set "Gpu:Platform" "RunPod"` |
| `RunPod pod ID not configured` | PodId not set | `dotnet user-secrets set "Gpu:RunPod:PodId" "<id>"` |
| Frontend `vite not recognized` | `node_modules` not installed | `cd src/frontend && npm install` |
| Port proxy not working after reboot | Rules don't persist | Re-run `setup-portproxy.ps1` as Admin |
