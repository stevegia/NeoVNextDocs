# UNREDACTED Session Overview: Security Hardening & UI Polish (2026-03-26)

> **DO NOT COMMIT** -- this file is gitignored. Contains real project IDs, IPs, and credentials references.

---

## GCP Security Hardening -- Concrete Values

### Project & Region
- **GCP Project:** `neovnext`
- **Primary Region:** `us-east4`
- **Cloud SQL Instance:** `neovnext:us-east4:neovlab-cluster`
- **Cloud SQL Public IP:** `34.21.76.69:5432`
- **Cloud SQL DB:** `cluster` / user: `neovlab` / password: `neovlab`
- **Filestore IP:** `10.243.63.162` (NFS share: `/models`)

### Service Accounts Created

| Account | Email | Roles Granted |
|---------|-------|---------------|
| API Invoker | `neovlab-api-invoker@neovnext.iam.gserviceaccount.com` | `roles/run.invoker` on `neovlab-gpu-worker-east4` |
| GPU Worker | `neovlab-gpu-worker@neovnext.iam.gserviceaccount.com` | `roles/run.developer`, `roles/cloudsql.client`, `roles/storage.objectUser` |

### IAM Commands Executed

```bash
# Create service accounts
gcloud iam service-accounts create neovlab-api-invoker \
    --display-name="NeoVLab API Invoker" --project=neovnext

gcloud iam service-accounts create neovlab-gpu-worker \
    --display-name="NeoVLab GPU Worker" --project=neovnext

# Grant API invoker permission to call Cloud Run GPU service
gcloud run services add-iam-policy-binding neovlab-gpu-worker-east4 \
    --region=us-east4 \
    --member="serviceAccount:neovlab-api-invoker@neovnext.iam.gserviceaccount.com" \
    --role="roles/run.invoker" \
    --project=neovnext

# Grant GPU worker its runtime roles
gcloud projects add-iam-policy-binding neovnext \
    --member="serviceAccount:neovlab-gpu-worker@neovnext.iam.gserviceaccount.com" \
    --role="roles/cloudsql.client"

gcloud projects add-iam-policy-binding neovnext \
    --member="serviceAccount:neovlab-gpu-worker@neovnext.iam.gserviceaccount.com" \
    --role="roles/storage.objectUser"

gcloud projects add-iam-policy-binding neovnext \
    --member="serviceAccount:neovlab-gpu-worker@neovnext.iam.gserviceaccount.com" \
    --role="roles/run.developer"

# Deploy GPU worker with new SA
gcloud run services update neovlab-gpu-worker-east4 \
    --region=us-east4 \
    --service-account="neovlab-gpu-worker@neovnext.iam.gserviceaccount.com" \
    --project=neovnext
```

### Service Account Key for Local Dev

A key was generated for the API invoker SA for local CloudDev profile:

```bash
gcloud iam service-accounts keys create z:\NeoVNext\neovlab-worker-invoker-key.json \
    --iam-account=neovlab-api-invoker@neovnext.iam.gserviceaccount.com \
    --project=neovnext
```

Referenced in `launchSettings.json` CloudDev profile:
```json
"GOOGLE_APPLICATION_CREDENTIALS": "z:\\NeoVNext\\neovlab-worker-invoker-key.json"
```

### Cloud SQL SSL Enforcement

```bash
gcloud sql instances patch neovlab-cluster \
    --require-ssl \
    --enable-password-policy \
    --password-policy-min-length=8 \
    --password-policy-complexity=COMPLEXITY_DEFAULT \
    --project=neovnext
```

Connection string updated in `appsettings.CloudDev.json`:
```
Host=34.21.76.69;Port=5432;Database=cluster;Username=neovlab;Password=neovlab;SslMode=Require;Trust Server Certificate=true
```

### OIDC Token Flow (what was broken and how it was fixed)

**Problem:** Running locally with `gcloud auth login` (user account), the adapter tried:
1. ADC (`GoogleCredential.GetApplicationDefaultAsync`) -- returns null for user accounts
2. `gcloud auth print-identity-token --audiences=<service-url>` -- fails with "Invalid account type for --audiences"
3. Token was null --> 403 from Cloud Run

**Fix in `GoogleCloudGpuPlatformAdapter.cs`:**
- Added fallback: try without `--audiences` flag (works for user accounts)
- Added `GOOGLE_APPLICATION_CREDENTIALS` to CloudDev profile so ADC finds the invoker SA key first

### Cloud Run Services -- Current State

```
neovlab-gpu-worker-east4:
  SA: neovlab-gpu-worker@neovnext.iam.gserviceaccount.com
  GPU: 1x L4, 4 vCPU, 16 GiB
  min-instances: 0, max-instances: 1
  --no-allow-unauthenticated
  VPC: default/default, vpc-egress=private-ranges-only
  NFS: 10.243.63.162:/models -> /models/comfyui
  GCS FUSE: neovnext-neovlab-io-cache -> /data/io-cache
```

### Verification Commands

```bash
# Verify SA on GPU worker
gcloud run services describe neovlab-gpu-worker-east4 \
    --region=us-east4 --project=neovnext \
    --format="value(spec.template.spec.serviceAccountName)"
# Expected: neovlab-gpu-worker@neovnext.iam.gserviceaccount.com

# Verify SSL required on Cloud SQL
gcloud sql instances describe neovlab-cluster --project=neovnext \
    --format="value(settings.ipConfiguration.requireSsl)"
# Expected: True

# Test OIDC token generation (with invoker SA key)
GOOGLE_APPLICATION_CREDENTIALS=z:\NeoVNext\neovlab-worker-invoker-key.json \
    gcloud auth print-identity-token \
    --audiences=https://neovlab-gpu-worker-east4-<hash>.run.app

# Test Cloud SQL TCP connectivity
Test-NetConnection -ComputerName 34.21.76.69 -Port 5432
```

---

## Deferred Security Items

| Item | Status | Notes |
|------|--------|-------|
| Database audit logging | Deferred | Enable when SQL instance scales beyond f1-micro |
| Remove default compute SA Editor role | TODO | Once all services confirmed on dedicated SAs |
| VPC Service Controls | Future | Overkill for current scale |
| Cloud Armor (WAF) | Future | Not needed until public-facing endpoints exist |
| Automated key rotation | Future | Manual rotation sufficient at current scale |

---

## UI Changes Summary

All UI changes are in the redacted changelog -- no secrets involved in those.
See `docs/changelog-2026-03-session-security-and-ui.md` for the full file list.
