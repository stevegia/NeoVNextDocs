# Database Cutover & Recovery Runbook

> **Scope**: spec 005-storage-architecture — dual-database schema rewrite  
> **Applies to**: NeoVLab API (SQLite private DB) and cluster DB (PostgreSQL — local Docker or Cloud SQL)

---

## Overview

NeoVLab uses two separate databases:

| Database | Technology | Contains | Managed by |
|----------|-----------|----------|-----------|
| Private API DB | SQLite (`data/neovlab.db`) | User data, model registry, GPU platform config | EF Core migrations (API startup) |
| Cluster DB | PostgreSQL (`cluster` schema) | job_queue, worker_registry, cache_manifest, storage_domains | `ClusterDbMigrationRunner` (API startup) |

Cutover scenarios arise when:
- Deploying spec 005 schema for the first time against an existing Cloud SQL instance
- Recovering from a corrupted or wiped cluster DB
- Migrating config rows from a live GCP Cloud SQL instance to a newly initialised cluster DB

---

## Scenario 1: First Deployment of Spec 005 on GCP

### Step 1: Apply cluster schema to Cloud SQL

The API applies the cluster schema automatically on startup via `ClusterDbMigrationRunner`.
Ensure `CLUSTER_DB_*` environment variables are set on the Cloud Run service before deploying.

```bash
# Verify schema was applied (run from local machine or Cloud Shell)
gcloud sql connect neovlab-cluster --user=neovlab --database=cluster
\dt        -- should list: job_queue, worker_registry, cache_manifest, storage_domains, ...
\q
```

If tables are missing, restart the Cloud Run API service to re-run migrations:

```bash
gcloud run services update neovlab-api --region us-east4 --update-env-vars DEPLOY_TRIGGER=$(date +%s)
```

### Step 2: Populate model registry and GPU platform config

After the schema is applied, copy config rows from the live GCP Cloud SQL source to the
new cluster DB using `scripts/populate-cluster-config.py`:

```bash
cd z:\NeoVNext   # or your repo root

python scripts/populate-cluster-config.py \
  --src "host=<SRC_HOST> dbname=neovlab user=neovlab password=<SRC_PASS> sslmode=require" \
  --dest "host=<DEST_HOST> dbname=cluster user=neovlab password=<DEST_PASS> sslmode=require"
```

Verify the import was successful:

```bash
python scripts/populate-cluster-config.py \
  --src "host=<SRC_HOST> dbname=neovlab user=neovlab password=<SRC_PASS> sslmode=require" \
  --dest "host=<DEST_HOST> dbname=cluster user=neovlab password=<DEST_PASS> sslmode=require" \
  --smoke-test
```

The `--smoke-test` flag exits non-zero if the destination row counts are lower than the source counts.

---

## Scenario 2: Cluster DB Recovery (Local Docker)

### Full wipe and recreate (no user data at risk)

The cluster DB holds **no private user data**. It can be wiped and recreated freely:

```bash
cd docker

# Stop all services
docker compose down

# Remove ONLY the cluster DB volume
docker volume rm neovnext_cluster-db-data

# Restart -- ClusterDbMigrationRunner recreates all tables on first connection
docker compose up -d

# Confirm schema applied
docker compose logs backend | grep -i migration
```

### Repopulate config after local cluster DB wipe

If you previously had model registry rows in the local cluster DB, restore them from
the SQLite private DB (which is unaffected by cluster DB wipes):

```bash
python scripts/populate-cluster-config.py \
  --src "host=localhost port=5432 dbname=cluster user=neovlab password=neovlab" \
  --dest "host=localhost port=5432 dbname=cluster user=neovlab password=neovlab"
```

> **Note**: For local dev, source and dest may be the same instance if you are populating
> a fresh cluster DB from an existing SQLite export. The script is idempotent — re-running
> it is safe due to `ON CONFLICT DO UPDATE` upsert logic.

---

## Scenario 3: Cluster DB Recovery (GCP Cloud SQL)

### Full wipe and recreate

```bash
# Delete instance (stops billing immediately)
gcloud sql instances delete neovlab-cluster --project=YOUR_PROJECT_ID --quiet

# Recreate from scratch (see runbook-gcp-gpu.md, Phase 1.4)
gcloud sql instances create neovlab-cluster \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=us-east4 \
  --project=YOUR_PROJECT_ID

gcloud sql databases create cluster --instance=neovlab-cluster --project=YOUR_PROJECT_ID
```

Then redeploy the API to trigger `ClusterDbMigrationRunner`, and repopulate config:

```bash
python scripts/populate-cluster-config.py \
  --src "host=<SRC_HOST> dbname=neovlab user=neovlab password=<SRC_PASS> sslmode=require" \
  --dest "host=<NEW_CLOUD_SQL_HOST> dbname=cluster user=neovlab password=<PASS> sslmode=require" \
  --smoke-test
```

### Partial recovery (jobs lost, config intact)

If the cluster DB is intact but the `job_queue` table is corrupted:

```sql
-- Connect via gcloud sql connect neovlab-cluster --user=neovlab --database=cluster
TRUNCATE TABLE job_queue;
-- Workers will continue polling; the API will accept new jobs immediately.
```

---

## Scenario 4: Private SQLite DB Recovery

The private API DB contains user data (model_registry, gpu_platform_config, etc.) and is
**not** wiped during cluster DB operations.

### Recreate from scratch (dev only)

```bash
rm data/neovlab.db
dotnet run --project src/backend/NeoVLab.Api
# EF Core migrations recreate all tables on startup
```

### Export before wipe

```bash
sqlite3 data/neovlab.db ".mode json" "SELECT * FROM model_registry;" > data/model_registry_backup.json
sqlite3 data/neovlab.db ".mode json" "SELECT * FROM gpu_platform_config;" > data/gpu_platform_config_backup.json
```

---

## Verification Checklist

After any cutover or recovery:

- [ ] `GET /health` returns 200
- [ ] `GET /api/observability/queue-depth` returns `{"Depth": 0}` (or a valid count)
- [ ] `GET /api/observability/error-rate` returns a valid response
- [ ] `docker compose logs backend | grep -i migration` shows no errors
- [ ] `SELECT COUNT(*) FROM model_registry` on cluster DB matches expected count
- [ ] Workers re-register within 30s of startup (`GET /api/workers` lists active workers)

---

## Related

- `scripts/populate-cluster-config.py` — FR-045 post-deploy config population script
- `docs/runbook-gcp-gpu.md` — Full GCP provisioning and teardown guide
- `specs/005-storage-architecture/quickstart.md` — Build/deploy validation steps
- `specs/005-storage-architecture/data-model.md` — Full schema reference
