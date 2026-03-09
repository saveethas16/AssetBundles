# Medallion DABs Template

Reusable Databricks Asset Bundles template for medallion architecture pipelines.
Supports Dev / Staging / UAT / Prod environments out of the box.

---

## File Ownership — What to Change vs What to Leave Alone

| File | Own by | Change for new project? |
|---|---|---|
| `.github/workflows/_run-tests.yml` | Platform team | ❌ Never |
| `.github/workflows/_deploy.yml` | Platform team | ❌ Never |
| `.github/workflows/deploy-*.yml` | Project team | ✅ Branch name, secrets only |
| `databricks.yml` | Project team | ✅ bundle.name, hosts, variable values |
| `resources/job.yml` | Project team | ✅ Tasks, cluster, schedule |
| `resources/dlt_pipeline.yml` | Project team | ✅ Notebook paths |
| `src/notebooks/**` | Project team | ✅ Replace entirely |
| `tests/test_pipeline.py` | Project team | ✅ Schema, fixture data, assertions |

---

## New Project Checklist

Work through these in order. Each step has `← CHANGE` markers in the relevant file.

### Step 1 — `databricks.yml`
- [ ] Change `bundle.name` to your project name (no spaces, e.g. `sales_pipeline`)
- [ ] Replace all four `workspace.host` URLs with your actual workspace URLs
- [ ] Update `catalog` values per target (e.g. `dev_sales`, `prod_sales`)
- [ ] Update `volume_name` values per target
- [ ] Update `file_name` to your source file
- [ ] Update `dlt_pipeline_name` values per target

### Step 2 — `resources/job.yml`
- [ ] Change the job resource key and `name` to match your project
- [ ] Update `node_type_id` and `num_workers` for your data volume
- [ ] Replace all `notebook_path` values with your notebook paths
- [ ] Update `base_parameters` keys to match what your notebooks expect
- [ ] Set your `quartz_cron_expression` and `timezone_id`
- [ ] Uncomment and set `run_as` service principal for prod

### Step 3 — `resources/dlt_pipeline.yml`
- [ ] Replace all notebook `path` values with your DLT notebooks
- [ ] Add any new `configuration` keys your notebooks call `spark.conf.get()` for

### Step 4 — `src/notebooks/`
- [ ] Replace all notebooks with your project's actual pipeline notebooks
- [ ] Every notebook must declare `dbutils.widgets.text()` for all parameters it uses
- [ ] Every DLT notebook must use `spark.conf.get()` (not widgets) for config

### Step 5 — `tests/test_pipeline.py`
- [ ] Update `sample_orders` fixture schema to match your source columns
- [ ] Update sample data rows to include your edge cases (nulls, dupes, bad values)
- [ ] Update `TestSilver._transform()` to mirror your silver notebook exactly
- [ ] Update column names in all assertions

### Step 6 — `.github/workflows/deploy-*.yml`
- [ ] No changes needed if your branches are named `dev`, `staging`, `uat`, `main`
- [ ] Update `job_name` in `deploy-prod.yml` if you changed the job resource key

### Step 7 — GitHub Repository Setup
- [ ] Add these Secrets in Settings → Secrets → Actions:

```
DEV_DATABRICKS_HOST
DEV_DATABRICKS_TOKEN
STAGING_DATABRICKS_HOST
STAGING_DATABRICKS_TOKEN
UAT_DATABRICKS_HOST
UAT_DATABRICKS_TOKEN
PROD_DATABRICKS_HOST
PROD_DATABRICKS_TOKEN     ← use a Service Principal token, not personal
```

- [ ] Create GitHub Environments: `dev`, `staging`, `uat`, `prod`
- [ ] Add protection rules to `prod` environment (required reviewers, branch restriction to `main`)

---

## Adding a New Environment

Only two files need to change:

**1. Create `.github/workflows/deploy-<env>.yml`** — copy any existing caller:
```yaml
name: Deploy to <Env>
on:
  push:
    branches:
      - <your-branch>
jobs:
  deploy:
    uses: ./.github/workflows/_deploy.yml
    with:
      target: <env>
      profile: <ENV>
      run_job: false
    secrets:
      DATABRICKS_HOST: ${{ secrets.ENV_DATABRICKS_HOST }}
      DATABRICKS_TOKEN: ${{ secrets.ENV_DATABRICKS_TOKEN }}
```

**2. Add a target block to `databricks.yml`** — copy any existing target, update host and variable values.

**3. Add two GitHub Secrets** — `ENV_DATABRICKS_HOST` and `ENV_DATABRICKS_TOKEN`.

---

## CI/CD Flow

```
push → dev      →  tests → deploy to Dev workspace
push → staging  →  tests → deploy to Staging workspace
push → uat      →  tests → deploy to UAT workspace
push → main     →  tests → deploy to Prod workspace → run medallion_daily_job
```

Tests always run first. A failing test blocks the deploy.

---

## Running Tests Locally

```bash
pip install pytest pyspark==3.5.0 delta-spark==3.0.0
pytest tests/ -v --tb=short
```

## Validating the Bundle Locally

```bash
databricks bundle validate                    # dev (default)
databricks bundle validate --target staging
databricks bundle validate --target prod
```
