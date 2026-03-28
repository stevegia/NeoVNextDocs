# Cloud Security Plan — NeoVNext / neovnext GCP

**Date:** 2026-03-26
**Author:** Aethon (Cloud Goddess)
**Status:** ACTIVE — for Steve's review and implementation

---

## 1. Current Security Posture

### 1.1 What Is In Place (Strengths)

| Control | Status | Detail |
|---------|--------|--------|
| Cloud Run IAM auth | **Active** | `--no-allow-unauthenticated` on all deploy scripts. Google OIDC identity tokens required for every request. |
| Dedicated service accounts | **Active** | `neovlab-api-invoker` (run.invoker only) and `neovlab-gpu-worker` (cloudsql.client + storage.objectUser + run.developer). Default Compute SA replaced. |
| OIDC auth flow | **Active** | Backend obtains OIDC token via ADC or gcloud CLI fallback. Tokens are audience-scoped to the Cloud Run service URL. |
| Application-layer WORKER_TOKEN | **Active** | 64-char hex token generated fresh each deployment via `openssl rand -hex 32`. Worker validates it on every inbound call. Constant-time comparison in Go supervisor. |
| Cloud SQL SSL | **Active** | `requireSsl=true` patched on instance. Connection strings use `SslMode=Require;Trust Server Certificate=true`. |
| Backend bearer token | **Active** | Local API validates a cryptographically-random 32-byte token stored in `data/auth-token.txt`. Frontend fetches it at startup from `/api/auth/token`. Constant-time comparison in `AuthTokenService`. |
| SA key gitignored | **Active** | `*-key.json` pattern in `.gitignore`. |
| Scale-to-zero | **Active** | `min-instances=0`. GPU worker is cold when not in use — there is no always-on attack surface. |
| VPC egress (private-ranges-only) | **Active** | `--vpc-egress private-ranges-only`. Outbound traffic from the worker is restricted to private RFC-1918 ranges. Filestore NFS and Cloud SQL Proxy both live on private IPs. |
| Application budget cap | **Active** | `Gpu:BudgetCents=500`. GPU dispatch blocked if estimated spend exceeds $5. Alerts at 80%. |
| GCP billing alert | **Documented** | `docker/configure-billing-alert.sh` provisions a GCP budget alert at $8. |

### 1.2 Security Gaps Found

#### GAP-1: CRITICAL — Credentials in Plaintext Env Vars (Cloud Run + tmp file)

**`scripts/tmp-env-vars.yaml` contains real secrets in plaintext and is NOT gitignored.**

The file was left on disk after the last deployment attempt. It contains:
- `WORKER_TOKEN: "de6a1c9f4705b238"` — a short (16 hex char = 8 bytes) token, not the expected 64-char token. This suggests it came from the PowerShell script's character-range approach rather than `openssl rand -hex 32`. 8 bytes of entropy is cryptographically insufficient.
- `DATABASE_URL: "postgresql://neovlab:neovlab@/cluster?..."` — the DB password in plaintext
- `CLOUD_RUN_SERVICE_URL: "https://neovlab-gpu-worker-east4-5ttejcnneq-uk.a.run.app"` — real service URL

The deploy script does `Remove-Item $ENV_FILE` at the end, but only on success. A failed deploy leaves this file on disk.

The PowerShell token generation (`-join ((48..57) + (97..102) | Get-Random -Count 64 | ForEach-Object { [char]$_ })`) produces a 64-char hex-alphabet string but uses `Get-Random` which is NOT cryptographically random — it uses the .NET `Random` class seeded from the system clock. The bash script correctly uses `openssl rand -hex 32`.

Additionally, the `tmp-service.yaml` file (also on disk, also not gitignored) contains:
- The real WORKER_TOKEN: `674848c3f68257678...` (plaintext in Cloud Run service YAML dump)
- The real DATABASE_URL with password: `postgresql://neovlab:neovlab@/cluster?...`
- The real Cloud SQL password in `CLOUD_SQL_PASSWORD: neovlab`
- The real service URL and URLs list

Both files have zero protection if the repo is ever made non-private or shared.

#### GAP-2: HIGH — Cloud Run Ingress Is Set to "all" (Internet-Exposed)

From `tmp-service.yaml`:
```
run.googleapis.com/ingress: all
run.googleapis.com/ingress-status: all
```

The Cloud Run service accepts traffic from **any IP address on the internet**. GCP IAM is the only gate. Any attacker who obtains or brute-forces a valid Google identity token can call the worker directly. While OIDC tokens are difficult to forge, this is unnecessary attack surface for a personal project.

Neither deploy script sets `--ingress internal` or `--ingress internal-and-cloud-load-balancing`. The service is deployed with the GCP default (all).

#### GAP-3: HIGH — Database Password Is Hardcoded and Weak

`appsettings.json`, `appsettings.CloudDev.json`, both deploy scripts, and `tmp-env-vars.yaml` all contain `Password=neovlab`. This is a dictionary word. The Cloud SQL instance has a public IP (`34.21.76.69:5432`). If `requireSsl` is ever accidentally removed or a client connects without enforcing it, the password is trivially brute-forced.

The password should be a random secret stored in Secret Manager, not committed to source code.

#### GAP-4: MEDIUM — `/api/auth/token` Is Unauthenticated and Publicly Callable

`AuthTokenMiddleware` exempts `/api/auth/token` from authentication. The intent is that the local frontend fetches its session token on startup. However, if the backend API is ever exposed beyond localhost (e.g., on a LAN or via port forwarding), any client that can reach port 6000 can fetch the bearer token without any credentials.

The current localhost-only design mitigates this, but it is a design assumption not enforced by the code.

#### GAP-5: MEDIUM — SignalR Hub Is Unauthenticated

The `/hubs/neovlab` path is exempt from `AuthTokenMiddleware`. While SignalR negotiation does need to be accessible, the hub itself should validate the bearer token passed as a query parameter during negotiation. Currently any client can connect to the hub without a valid token.

#### GAP-6: LOW — Service URL Exposed in `appsettings.json`

`appsettings.json` contains the real Cloud Run service URL (`https://neovlab-gpu-worker-east4-221741263387.us-east4.run.app`) in plaintext, committed to the repo. This combined with GAP-2 means anyone who reads the repo knows the exact endpoint. IAM auth still protects it, but defense-in-depth suggests the URL should live in user-secrets alongside the worker token.

#### GAP-7: LOW — `roles/run.developer` Is Overly Broad for the GPU Worker SA

The `neovlab-gpu-worker` service account has `roles/run.developer` which allows it to deploy new Cloud Run revisions and read service configurations. A compromised worker container could redeploy itself with altered parameters (e.g., modified entrypoint). The worker only needs to call its own health endpoint for keepalive — it does not need `run.developer`.

---

## 2. Auth Flow: Local Desktop App to Cloud Run

The complete authentication chain today:

```
[Local Desktop App]
    └── React frontend at localhost:6100
           │
           ├── GET /api/auth/token (no auth required)
           │        └── Returns auto-generated bearer token from data/auth-token.txt
           │
           └── All subsequent API calls: Authorization: Bearer <app-token>
                      │
                      ▼
[Local .NET Backend API at localhost:6000]
    │   Validates app-layer bearer token (AuthTokenMiddleware)
    │
    ├── Reads Gpu:GoogleCloud config (ServiceName, Region, ProjectId)
    │
    └── To call Cloud Run GPU worker:
           │
           ├── Try ADC (GOOGLE_APPLICATION_CREDENTIALS → neovlab-api-invoker-key.json)
           │       └── GoogleCredential.GetOidcTokenAsync(audience=serviceUrl)
           │
           ├── Fallback: gcloud auth print-identity-token --audiences=<serviceUrl>
           │
           └── Fallback: gcloud auth print-identity-token (user account)
                      │
                      ▼
[Cloud Run GPU Worker at *.run.app]
    │   GCP verifies OIDC token (audience, expiry, issuer)
    │   Checks caller has roles/run.invoker on this service
    │
    └── Worker validates X-Worker-Token header matches WORKER_TOKEN env var
```

**Bottom line:** Cloud Run is currently protected by GCP IAM (OIDC) only. There is no IP restriction. Any request reaching the service URL with a valid OIDC token from an identity holding `run.invoker` will be served.

---

## 3. IP Restriction: Options Analysis

### Option A: Keep IAM Auth Only (Current State)

**How it works:** Rely solely on GCP's OIDC verification and IAM `run.invoker` binding.

**Pros:**
- Already deployed. Zero additional work.
- OIDC tokens are cryptographically signed, short-lived (1 hour), audience-scoped. Very hard to steal and replay.
- No dynamic IP problem — IP doesn't matter.

**Cons:**
- Service URL is internet-reachable. If the `neovlab-api-invoker` SA key is ever leaked (e.g., committed to git), an attacker can call the GPU worker from anywhere.
- Relies entirely on one security boundary (GCP IAM). No defense in depth.

**Verdict:** Acceptable for a personal project with a private repo. Becomes insufficient if the repo is ever shared or the SA key is ever exposed.

---

### Option B: Cloud Run Internal Ingress (`--ingress internal`)

**How it works:** Set ingress to `internal`. Only traffic from within the GCP project's VPC (and Cloud Load Balancer) can reach the service. The *.run.app URL becomes unreachable from the internet entirely.

```bash
gcloud run services update neovlab-gpu-worker-east4 \
    --region us-east4 \
    --ingress internal
```

**Pros:**
- Removes the service from the internet completely. The URL stops resolving externally.
- No Cloud Armor cost, no load balancer cost.
- Keeps pull-model working: the worker polls Cloud SQL over the Cloud SQL Proxy (VPC internal), so no inbound traffic from the internet is needed for normal job processing.

**Cons:**
- The local backend at `localhost:6000` is NOT inside GCP's VPC. It connects to Cloud Run via the public internet. `--ingress internal` would break the API's ability to call the worker for health checks.
- The worker's keepalive mechanism pings its own external URL every 60 seconds. With `--ingress internal`, the worker's self-ping would fail (worker is inside the VPC, but the *.run.app URL would be blocked for external-origin traffic).
- Would require a VPN or Cloud Run Proxy setup for the local API to reach the worker.

**Verdict:** Not compatible with the current pull-model architecture unless the local API is moved behind a VPN or Serverless VPC Access. Too much complexity for a personal project.

---

### Option C: Cloud Load Balancer + Cloud Armor IP Allowlist

**How it works:** Put a Global HTTP(S) Load Balancer in front of Cloud Run. Attach a Cloud Armor security policy that allows only Steve's home IP. All other traffic is rejected at the edge (HTTP 403) before reaching Cloud Run.

```
Internet → Cloud Armor (IP allowlist) → Cloud Load Balancer → Cloud Run
```

**Pros:**
- True IP restriction. Attacker with a stolen OIDC token from a different IP is blocked at the edge.
- Cloud Armor provides DDoS protection, WAF rules, rate limiting.
- Works with the current pull-model (worker is still internet-accessible to the load balancer, but the LB only forwards allowed IPs).

**Cons:**
- Cost: Global Load Balancer = ~$18/month minimum (forwarding rule + backend service). Cloud Armor = $5/month policy fee + $1/million requests. For a personal project costing $7-10/month idle, this is a 2-3x cost increase.
- Complexity: Requires creating a serverless NEG, backend service, URL map, SSL cert, forwarding rules. ~15 gcloud commands.
- Dynamic IP problem: Home ISP IPs change. The Cloud Armor policy must be updated whenever the IP changes.
- The Cloud Run service must also change ingress to `internal-and-cloud-load-balancing` so direct *.run.app access is blocked.

**Verdict:** Correct solution if this were production. Cost-prohibitive for a personal project at current scale.

---

### Option D: VPC Connector + Private Internal Access (with Local VPN)

**How it works:** Route all access to Cloud Run through a VPN (e.g., Cloud VPN or Tailscale) that puts Steve's laptop inside the GCP VPC. Set Cloud Run to `--ingress internal`.

**Pros:** Maximum security. No internet exposure whatsoever.

**Cons:** Tailscale/Cloud VPN setup is non-trivial. Tailscale requires a relay node in GCP. Not worth the overhead for a personal project.

**Verdict:** Overkill.

---

### Option E: Recommended — IAM Hardening + SA Key in Secret Manager + Dynamic IP Script (No LB Cost)

This is the pragmatic choice for a personal project that takes security seriously without paying load balancer costs.

**Layer 1:** Fix the actual gaps (GAP-1, GAP-3) — this has more impact than IP restriction.

**Layer 2:** Store the `neovlab-api-invoker` SA key in Secret Manager rather than a file on disk. The local API reads it from the environment at startup.

**Layer 3:** Add a lightweight IP-based defense via Cloud Run request headers. Cloud Run passes the real client IP in `X-Forwarded-For`. The Go supervisor can validate that the calling IP matches the expected home IP. This is not cryptographically strong (X-Forwarded-For can be spoofed), but adds a layer without infrastructure cost.

**Layer 4 (if Steve's IP is static or predictable):** Add a Cloud Armor rule to the existing service ingress using the no-LB approach — Cloud Run supports `--ingress internal-and-cloud-load-balancing` without requiring you to actually create a load balancer if you use Cloud Armor's serverless connector. However, without a load balancer, Cloud Armor is not applicable to Cloud Run directly.

**Actual recommendation for "IP whitelisting" at zero cost:** Use the ingress flag approach for future services, and for the current service, accept that OIDC is the primary gate. Implement the IP check in the Go supervisor as a secondary validation.

---

## 4. Recommended IP Restriction Approach

Given the cost/complexity tradeoffs, the recommendation is **two-tier hardening**:

**Tier 1 (Fix immediately — no IP change needed):** Fix the credential hygiene gaps that are the real risk vectors.

**Tier 2 (Optional IP restriction — low cost):** If IP restriction is desired without load balancer cost, implement it at the application layer in the Go supervisor.

The case for a Cloud Armor + Load Balancer setup only makes sense when:
- The project has a static public-facing URL (custom domain)
- Monthly budget for infrastructure increases are acceptable (~$20/month overhead)
- The threat model includes stolen SA keys (which Tier 1 largely addresses anyway)

---

## 5. Implementation Plan

### Phase 1: Fix Critical Gaps (Immediate)

#### Step 1.1 — Delete and gitignore the plaintext secret files

```bash
# From z:\NeoVNext

# Delete the leftover env files with real credentials
rm scripts/tmp-env-vars.yaml
rm tmp-service.yaml
rm tmp-startup-probe-patch.yaml
```

Add to `.gitignore`:
```
# Temporary deploy artifacts with credentials
tmp-service.yaml
tmp-startup-probe-patch.yaml
scripts/tmp-env-vars.yaml
scripts/tmp-env-vars*.yaml
```

#### Step 1.2 — Fix WORKER_TOKEN generation in PowerShell deploy script

The PowerShell script uses `Get-Random` which is not cryptographically random. Fix it to use `[System.Security.Cryptography.RandomNumberGenerator]`:

In `scripts/deploy-gpu-supervisor.ps1`, replace:
```powershell
$WORKER_TOKEN = -join ((48..57) + (97..102) | Get-Random -Count 64 | ForEach-Object { [char]$_ })
```

With:
```powershell
$rng = [System.Security.Cryptography.RandomNumberGenerator]::Create()
$bytes = New-Object byte[] 32
$rng.GetBytes($bytes)
$WORKER_TOKEN = ($bytes | ForEach-Object { $_.ToString("x2") }) -join ""
```

This produces 32 cryptographically random bytes encoded as 64 hex chars — same as `openssl rand -hex 32`.

#### Step 1.3 — Move DB password to Secret Manager

The password `neovlab` is committed in appsettings files and deploy scripts. Remediation (code-layer change — route to Go-Gasmoth / Dominika):

1. Generate a strong password:
   ```bash
   openssl rand -base64 32
   ```

2. Create Secret Manager secret:
   ```bash
   echo -n "THE_NEW_PASSWORD" | gcloud secrets create neovlab-cluster-db-password \
       --data-file=- \
       --project=neovnext
   ```

3. Grant the GPU worker SA access to the secret:
   ```bash
   gcloud secrets add-iam-policy-binding neovlab-cluster-db-password \
       --member="serviceAccount:neovlab-gpu-worker@neovnext.iam.gserviceaccount.com" \
       --role="roles/secretmanager.secretAccessor" \
       --project=neovnext
   ```

4. Update Cloud Run to mount the secret as an environment variable:
   ```bash
   gcloud run services update neovlab-gpu-worker-east4 \
       --region us-east4 \
       --update-secrets "CLOUD_SQL_PASSWORD=neovlab-cluster-db-password:latest" \
       --project=neovnext
   ```

5. Update `appsettings.CloudDev.json` and all deploy scripts to read the password from the environment or a local `.env` file (gitignored), not from plaintext config.

6. Rotate the SQL user password:
   ```bash
   gcloud sql users set-password neovlab \
       --instance=neovlab-cluster \
       --password="THE_NEW_PASSWORD" \
       --project=neovnext
   ```

Note: The appsettings files contain plaintext passwords — this is a code-layer change. Document for Dominika to update the connection string pattern, and Go-Gasmoth to update the Python worker's DATABASE_URL construction.

#### Step 1.4 — Store WORKER_TOKEN in Secret Manager

Treat WORKER_TOKEN like a credential. After each deployment, store the generated token in Secret Manager rather than printing it to stdout:

```bash
# After generating WORKER_TOKEN
echo -n "$WORKER_TOKEN" | gcloud secrets create neovlab-worker-token-east4 \
    --data-file=- --project=neovnext

# Or update if exists
echo -n "$WORKER_TOKEN" | gcloud secrets versions add neovlab-worker-token-east4 \
    --data-file=-

# Grant API invoker SA access (so the backend can read it)
gcloud secrets add-iam-policy-binding neovlab-worker-token-east4 \
    --member="serviceAccount:neovlab-api-invoker@neovnext.iam.gserviceaccount.com" \
    --role="roles/secretmanager.secretAccessor" \
    --project=neovnext
```

The backend's `Gpu:GoogleCloud:GpuWorkerToken` config value can then be sourced from the secret rather than stored in user-secrets manually. This is a code-layer integration — document for Go-Gasmoth.

#### Step 1.5 — Fix the `neovlab-gpu-worker` SA role (remove run.developer)

The worker SA has `roles/run.developer` which is overly broad. Downgrade to `roles/run.viewer`:

```bash
# Remove run.developer
gcloud projects remove-iam-policy-binding neovnext \
    --member="serviceAccount:neovlab-gpu-worker@neovnext.iam.gserviceaccount.com" \
    --role="roles/run.developer"

# Add run.viewer (allows reading service config for keepalive URL discovery)
gcloud projects add-iam-policy-binding neovnext \
    --member="serviceAccount:neovlab-gpu-worker@neovnext.iam.gserviceaccount.com" \
    --role="roles/run.viewer"
```

Verify the keepalive mechanism still works after this change (it uses the metadata server for URL discovery, not the Cloud Run API — so this should be safe).

#### Step 1.6 — Add ServiceUrl to user-secrets instead of appsettings.json

The Cloud Run service URL in `appsettings.json` is committed to git. Move it to user-secrets:

```bash
dotnet user-secrets set "Gpu:GoogleCloud:ServiceUrl" "https://neovlab-gpu-worker-east4-5ttejcnneq-uk.a.run.app"
```

Remove the `ServiceUrl` key from `appsettings.json`. The code already handles missing ServiceUrl gracefully (falls back to derived URL).

---

### Phase 2: Optional IP Restriction (If Desired)

This phase is optional. Implement only if Steve wants an additional layer beyond GCP IAM.

#### Option 2A: IP Allowlist in Go Supervisor (No Infrastructure Cost)

Note: This is a code-layer change — route to Go-Gasmoth.

Add an `ALLOWED_CALLER_IPS` environment variable to the Go supervisor. On each inbound request, check the `X-Forwarded-For` header (Cloud Run sets this to the real client IP). Reject requests from IPs not in the allowlist with HTTP 403.

```
ALLOWED_CALLER_IPS=203.0.113.42  # Steve's home IP
```

Caveats:
- `X-Forwarded-For` can be spoofed in some configurations. Cloud Run's forwarded headers are set by GCP infrastructure, so they are trustworthy in this context.
- Must handle IP rotation (see Section 6).

This adds a meaningful second factor. An attacker with a stolen OIDC token from a non-whitelisted IP gets HTTP 403 from the application layer.

#### Option 2B: Cloud Armor + Load Balancer (Full IP Whitelist)

Use only if a load balancer is already planned (e.g., for a custom domain).

```bash
# 1. Create a serverless NEG for Cloud Run
gcloud compute network-endpoint-groups create neovlab-gpu-neg \
    --region=us-east4 \
    --network-endpoint-type=serverless \
    --cloud-run-service=neovlab-gpu-worker-east4 \
    --project=neovnext

# 2. Create backend service
gcloud compute backend-services create neovlab-gpu-backend \
    --global \
    --load-balancing-scheme=EXTERNAL \
    --project=neovnext

gcloud compute backend-services add-backend neovlab-gpu-backend \
    --global \
    --network-endpoint-group=neovlab-gpu-neg \
    --network-endpoint-group-region=us-east4 \
    --project=neovnext

# 3. Create Cloud Armor security policy
gcloud compute security-policies create neovlab-home-ip-policy \
    --description="Restrict Cloud Run GPU worker to Steve's home IP" \
    --project=neovnext

# 4. Add IP allowlist rule
STEVE_HOME_IP="YOUR.HOME.IP.HERE"
gcloud compute security-policies rules create 1000 \
    --security-policy=neovlab-home-ip-policy \
    --src-ip-ranges="${STEVE_HOME_IP}/32" \
    --action=allow \
    --project=neovnext

# 5. Deny everything else
gcloud compute security-policies rules create 2147483647 \
    --security-policy=neovlab-home-ip-policy \
    --src-ip-ranges="0.0.0.0/0" \
    --action=deny-403 \
    --project=neovnext

# 6. Attach policy to backend service
gcloud compute backend-services update neovlab-gpu-backend \
    --global \
    --security-policy=neovlab-home-ip-policy \
    --project=neovnext

# 7. Update Cloud Run ingress
gcloud run services update neovlab-gpu-worker-east4 \
    --region=us-east4 \
    --ingress=internal-and-cloud-load-balancing \
    --project=neovnext
```

**Estimated additional monthly cost:** ~$18-23/month (load balancer forwarding rule + Cloud Armor policy).

---

## 6. Dynamic IP Contingency

Steve's home ISP likely assigns a dynamic IP. This is the main practical challenge with IP whitelisting.

### Approach A: Script to Update Cloud Armor Policy

Create `scripts/update-home-ip.ps1`:

```powershell
# Update Cloud Armor allowlist with current home IP
# Usage: Run this when your home IP changes

$CURRENT_IP = (Invoke-WebRequest -Uri "https://api.ipify.org" -UseBasicParsing).Content.Trim()
Write-Output "Current IP: $CURRENT_IP"

gcloud compute security-policies rules update 1000 `
    --security-policy=neovlab-home-ip-policy `
    --src-ip-ranges="${CURRENT_IP}/32" `
    --project=neovnext

Write-Output "Cloud Armor updated: only $CURRENT_IP/32 allowed"
```

Run this script whenever the home IP changes. IP changes are typically detectable because Cloud Run calls start returning 403.

### Approach B: Script to Update Application-Layer Allowlist

If using Option 2A (Go supervisor check), update the Cloud Run env var:

```powershell
$CURRENT_IP = (Invoke-WebRequest -Uri "https://api.ipify.org" -UseBasicParsing).Content.Trim()
Write-Output "Current IP: $CURRENT_IP"

gcloud run services update neovlab-gpu-worker-east4 `
    --region us-east4 `
    --update-env-vars "ALLOWED_CALLER_IPS=$CURRENT_IP" `
    --project=neovnext

Write-Output "Worker IP allowlist updated to $CURRENT_IP"
```

### Approach C: Dynamic DNS + CIDR Range

Use a DDNS service (e.g., No-IP, DuckDNS) that keeps a hostname pointed at the current home IP. Write a scheduled task that:
1. Resolves the DDNS hostname to get the current IP
2. Compares with the stored Cloud Armor rule
3. Updates if changed

Most home ISPs assign IPs from a limited pool for a given customer. A `/28` or `/27` CIDR block often covers the range the ISP uses for a given address. This reduces update frequency at the cost of a slightly broader allowlist.

### Detection: How to Know the IP Changed

If using Cloud Armor or application-layer IP check, a stale IP rule manifests as HTTP 403 from the GPU worker. The backend logs:
```
Cloud Run GPU health check returned 403 for https://neovlab-gpu-worker-east4-...run.app
```

The immediate remediation is to run the update script above.

---

## 7. Security Gaps Summary and Priority

| Gap | Severity | Effort | Action |
|-----|----------|--------|--------|
| GAP-1: Plaintext secret files on disk | **Critical** | 5 min | Delete `tmp-env-vars.yaml` and `tmp-service.yaml`. Add to `.gitignore`. |
| GAP-1b: Weak WORKER_TOKEN generation (PS) | **High** | 10 min | Fix PowerShell `Get-Random` → `RandomNumberGenerator`. |
| GAP-2: Cloud Run ingress = all | **High** | 0 min (by design) | Accepted. OIDC is the gate. Address if load balancer is added later. |
| GAP-3: DB password is weak and committed | **High** | 30 min | Generate strong password, move to Secret Manager. Coordinate with Dominika and Go-Gasmoth. |
| GAP-4: `/api/auth/token` unauthenticated | **Medium** | Inform only | Acceptable for localhost-only use. Document the assumption. |
| GAP-5: SignalR hub unauthenticated | **Medium** | Code change | Route to Go-Gasmoth to add hub-level token validation. |
| GAP-6: Service URL in appsettings.json | **Low** | 5 min | Move to user-secrets. |
| GAP-7: Worker SA has run.developer | **Low** | 5 min | Downgrade to run.viewer. |

---

## 8. Items Requiring Code Changes (Route to Lilithex / Go-Gasmoth)

The following security gaps require changes to application code, not infrastructure:

1. **GAP-3 (backend side):** Update connection string handling in `appsettings.CloudDev.json` and `appsettings.json` to read DB password from environment variable rather than plaintext. Owner: Go-Gasmoth / Dominika.

2. **GAP-3 (worker side):** Update `DATABASE_URL` construction in `deploy-gpu-supervisor.ps1` and `deploy-gpu-worker-go.sh` to read from Secret Manager at deploy time, not hardcode the password. Owner: Go-Gasmoth.

3. **GAP-5:** Add bearer token validation to the SignalR hub connection in `NeoVLabHub.cs`. The hub should validate the `access_token` query parameter on `OnConnectedAsync`. Owner: Go-Gasmoth.

4. **Optional 2A:** Add `ALLOWED_CALLER_IPS` validation to the Go supervisor's HTTP handler. Owner: Go-Gasmoth.

---

## 9. Mermaid: Security Boundary Diagram

```mermaid
graph TB
    subgraph "Internet (Untrusted)"
        ATTACKER[Attacker]
        STEVE[Steve's Desktop]
    end

    subgraph "GCP Project: neovnext"
        subgraph "IAM Boundary"
            OIDC[GCP IAM Auth\nOIDC Token Verification\nroles/run.invoker]
        end

        subgraph "Cloud Run: neovlab-gpu-worker-east4"
            WORKER_AUTH[Go Supervisor\nWORKER_TOKEN Validation]
            GPU[L4 GPU Worker\nHandlers]
        end

        subgraph "Private VPC"
            CSQL[(Cloud SQL\nPostgreSQL\nSSL required)]
            NFS[Filestore NFS\n/models]
        end

        subgraph "Cloud Storage"
            GCS[io-cache bucket\nartifacts bucket]
        end
    end

    subgraph "Steve's LAN"
        API[.NET Backend API\nlocalhost:6000]
        FRONTEND[React Frontend\nlocalhost:6100]
    end

    FRONTEND -->|Bearer token\napp-layer auth| API
    API -->|OIDC token\nroles/run.invoker SA| OIDC
    OIDC -->|Verified| WORKER_AUTH
    WORKER_AUTH -->|WORKER_TOKEN OK| GPU
    GPU --> CSQL
    GPU --> NFS
    GPU --> GCS

    ATTACKER -->|No OIDC token| OIDC
    ATTACKER -->|OIDC token\nbut wrong IP\n(if Option 2A implemented)| WORKER_AUTH
    STEVE -->|Direct desktop use| FRONTEND

    style OIDC fill:#4285f4,color:#fff
    style WORKER_AUTH fill:#34a853,color:#fff
    style ATTACKER fill:#ea4335,color:#fff
    style STEVE fill:#fbbc04
```

---

## 10. Quick Reference: Security Scripts

All scripts should live in `scripts/security/`.

### Check current home IP
```powershell
(Invoke-WebRequest -Uri "https://api.ipify.org" -UseBasicParsing).Content
```

### Update Cloud Armor IP allowlist (if implemented)
```powershell
# scripts/security/update-home-ip.ps1
$CURRENT_IP = (Invoke-WebRequest -Uri "https://api.ipify.org" -UseBasicParsing).Content.Trim()
gcloud compute security-policies rules update 1000 `
    --security-policy=neovlab-home-ip-policy `
    --src-ip-ranges="${CURRENT_IP}/32" `
    --project=neovnext
```

### Check Cloud Run ingress setting
```bash
gcloud run services describe neovlab-gpu-worker-east4 \
    --region us-east4 \
    --format "value(metadata.annotations['run.googleapis.com/ingress'])" \
    --project=neovnext
```

### Verify IAM bindings on Cloud Run service
```bash
gcloud run services get-iam-policy neovlab-gpu-worker-east4 \
    --region us-east4 \
    --project=neovnext
```

### Rotate WORKER_TOKEN (redeploy with new token)
```powershell
# Regenerate token and redeploy
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/deploy-gpu-supervisor.ps1
# Then update backend user-secrets with new token printed at end of script
```

### Downgrade worker SA role (one-time fix for GAP-7)
```bash
gcloud projects remove-iam-policy-binding neovnext \
    --member="serviceAccount:neovlab-gpu-worker@neovnext.iam.gserviceaccount.com" \
    --role="roles/run.developer"

gcloud projects add-iam-policy-binding neovnext \
    --member="serviceAccount:neovlab-gpu-worker@neovnext.iam.gserviceaccount.com" \
    --role="roles/run.viewer"
```
