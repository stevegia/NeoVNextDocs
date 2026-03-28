# GCP Provisioning and Teardown

This runbook covers the minimum resources needed for NeoVLab pull-model cloud execution.
It is designed to keep deployments compatible with Cloud SQL + io-cache and to avoid accidental image rebuilds of model-heavy layers.

## Prerequisites

- gcloud authenticated and project selected
- Billing enabled
- Required APIs enabled: run.googleapis.com, sqladmin.googleapis.com, cloudbuild.googleapis.com, storage.googleapis.com

## Resource Names

- Project: `${PROJECT_ID}`
- Region: `${REGION}`
- Cloud SQL instance: `${PROJECT_ID}:${REGION}:neovlab-cluster`
- Cluster database: `cluster`
- Model bucket: `gs://${PROJECT_ID}-neovlab-models`
- IO cache bucket: `gs://${PROJECT_ID}-neovlab-io-cache`
- CPU service: `neovlab-worker`
- GPU service: `neovlab-gpu-worker-${REGION}`

## Provisioning Steps

1. Create storage buckets.

```bash
gsutil mb -l "${REGION}" "gs://${PROJECT_ID}-neovlab-models"
gsutil mb -l "${REGION}" "gs://${PROJECT_ID}-neovlab-io-cache"
```

2. Create Cloud SQL PostgreSQL instance and database.

```bash
gcloud sql instances create neovlab-cluster \
  --database-version=POSTGRES_16 \
  --tier=db-f1-micro \
  --region="${REGION}"

gcloud sql databases create cluster --instance=neovlab-cluster
```

3. Create or update a DB user.

```bash
gcloud sql users set-password neovlab \
  --instance=neovlab-cluster \
  --password="${CLOUD_SQL_PASSWORD}"
```

4. Deploy GPU worker (safe mode by default, no rebuild).

```bash
REBUILD_IMAGE=0 \
CLOUD_SQL_INSTANCE="${PROJECT_ID}:${REGION}:neovlab-cluster" \
CLOUD_SQL_DATABASE="cluster" \
CLOUD_SQL_USERNAME="neovlab" \
CLOUD_SQL_PASSWORD="${CLOUD_SQL_PASSWORD}" \
./docker/deploy-gpu-worker.sh "${PROJECT_ID}" "${REGION}" "${REGION}"
```

## IAM & Security Setup

After creating the Cloud SQL instance and before deploying services, set up
dedicated service accounts and enable SSL.

### 6. Create service accounts (least-privilege)

```bash
# API invoker -- used by backend to call Cloud Run GPU workers
gcloud iam service-accounts create neovlab-api-invoker \
    --display-name="NeoVLab API Invoker" --project="${PROJECT_ID}"

# GPU worker runtime identity
gcloud iam service-accounts create neovlab-gpu-worker \
    --display-name="NeoVLab GPU Worker" --project="${PROJECT_ID}"
```

### 7. Grant IAM roles

```bash
# Invoker can call the GPU worker service
gcloud run services add-iam-policy-binding "neovlab-gpu-worker-${REGION}" \
    --region="${REGION}" \
    --member="serviceAccount:neovlab-api-invoker@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/run.invoker" \
    --project="${PROJECT_ID}"

# GPU worker needs Cloud SQL, Storage, and Run access
for ROLE in roles/cloudsql.client roles/storage.objectUser roles/run.developer; do
    gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
        --member="serviceAccount:neovlab-gpu-worker@${PROJECT_ID}.iam.gserviceaccount.com" \
        --role="${ROLE}"
done
```

### 8. Enable Cloud SQL SSL and password policy

```bash
gcloud sql instances patch neovlab-cluster \
    --require-ssl \
    --enable-password-policy \
    --password-policy-min-length=8 \
    --password-policy-complexity=COMPLEXITY_DEFAULT \
    --project="${PROJECT_ID}"
```

Connection strings must use `SslMode=Require;Trust Server Certificate=true`.

### 9. Generate invoker SA key for local development

```bash
gcloud iam service-accounts keys create neovlab-worker-invoker-key.json \
    --iam-account="neovlab-api-invoker@${PROJECT_ID}.iam.gserviceaccount.com" \
    --project="${PROJECT_ID}"
```

Set `GOOGLE_APPLICATION_CREDENTIALS` to this file path in your CloudDev launch profile.
The key file pattern `*-key.json` is gitignored.

---

## Safe Deployment Rules

- Default behavior is safe deploy: `REBUILD_IMAGE=0`.
- Set `REBUILD_IMAGE=1` only when you intentionally publish a new image.
- This prevents replacing model-heavy or memory-intensive image layers during routine config rollouts.

## Teardown Steps

1. Scale GPU worker to zero immediately (kill switch).

```bash
gcloud run services update "neovlab-gpu-worker-${REGION}" --region "${REGION}" --max-instances 0
```

2. Delete Cloud Run service.

```bash
gcloud run services delete "neovlab-gpu-worker-${REGION}" --region "${REGION}" --quiet
```

3. Delete Cloud SQL database and instance.

```bash
gcloud sql databases delete cluster --instance=neovlab-cluster --quiet
gcloud sql instances delete neovlab-cluster --quiet
```

4. Delete storage buckets.

```bash
gcloud storage rm --recursive "gs://${PROJECT_ID}-neovlab-io-cache"
gcloud storage rm --recursive "gs://${PROJECT_ID}-neovlab-models"
```

5. Delete service accounts.

```bash
gcloud iam service-accounts delete "neovlab-api-invoker@${PROJECT_ID}.iam.gserviceaccount.com" --quiet
gcloud iam service-accounts delete "neovlab-gpu-worker@${PROJECT_ID}.iam.gserviceaccount.com" --quiet
```

6. Remove budget alert (if configured).

```bash
gcloud billing budgets list --billing-account="${BILLING_ACCOUNT}"
gcloud billing budgets delete "${BUDGET_NAME}" --billing-account="${BILLING_ACCOUNT}"
```

---

## Resource Cost Reference

Approximate costs for the minimum NeoVLab GCP configuration (us-east4, as of 2026).  
All resources are scale-to-zero or pay-per-use unless noted.

| Resource | SKU / Tier | Estimated Cost | Notes |
|----------|-----------|----------------|-------|
| Cloud SQL (cluster DB) | db-f1-micro, PostgreSQL 16 | ~$7 USD/month | Runs continuously; stop instance to pause billing |
| Cloud Run CPU worker | 1 vCPU, 512 MiB, scale-to-zero | ~$0-$1 per test session | Billed per request-second; free tier covers light testing |
| Cloud Run GPU worker | 1x NVIDIA L4 GPU, scale-to-zero | ~$1-$3 per hour active | Scale to 0 max-instances immediately after GPU tests |
| Cloud Storage (models bucket) | Standard, ~7 GB | ~$0.15/month | One-time populate; no egress within GCP region |
| Cloud Storage (io-cache bucket) | Standard, per job | ~$0.02/GB | Negligible for test volumes; clean up after sessions |
| Cloud Build (image build) | First 120 min/day free | $0.003/build-minute | Only charged when `REBUILD_IMAGE=1` |
| **Total (idle monthly)** | | **~$7-$10/month** | Dominated by Cloud SQL instance running 24/7 |
| **Total (active GPU session)** | | **~$10-$15/session** | Includes 1-2h GPU + Cloud SQL + storage |

### Cost Control Tips

- **Pause Cloud SQL** between development sessions: `gcloud sql instances patch neovlab-cluster --activation-policy NEVER`
- **Resume Cloud SQL** before deploying: `gcloud sql instances patch neovlab-cluster --activation-policy ALWAYS`
- **GPU kill switch**: `gcloud run services update neovlab-gpu-worker-${REGION} --region ${REGION} --max-instances 0`
- **Set a billing alert** via `docker/configure-billing-alert.sh` to get notified before budget is exceeded
