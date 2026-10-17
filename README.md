# AWS Sales Data Pipeline (Terraform + GitHub Actions CI/CD)

A small, complete, serverless data pipeline: raw CSV lands in S3, a Lambda validates and cleans it,
curated data is written as partitioned files, and you query it with SQL in Athena.
Everything is Infrastructure-as-Code (Terraform) and deployed by a CI/CD pipeline (GitHub Actions + OIDC, no stored AWS keys).

```
                  upload                S3 event              writes
 sales_orders.csv ------> S3 raw  ------------------> Lambda ---------> S3 curated
                        (incoming/*.csv)          validate / dedupe     orders/year=YYYY/month=MM/*.csv.gz   (good rows)
                                                  derive total_amount   quarantine/*_rejects.csv             (bad rows + reason)
                                                         |
                                                         +--> CloudWatch Logs --> Alarm (errors) --> SNS email

 Glue Catalog table (partition projection)  <--  Athena workgroup (1 GB scan cap)  <--  you / BI tools
```

## The dataset
`data/sales_orders.csv` - 5,000 synthetic e-commerce orders (2023-2024). Columns:
`order_id, order_date, customer_id, product, category, quantity, unit_price, country, payment_method`.

It contains **160 deliberately bad rows** (negative quantity, missing customer, impossible date, non-numeric price, duplicates)
so you can watch the data-quality path work. Expected result: **4,840 good, 160 rejected**.
Regenerate any time: `python scripts/generate_dataset.py`.

## Part 1 - Run it on your computer (no AWS account needed)
Requires Python 3.10+.

```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt

python scripts/run_local.py     # runs the real Lambda logic on the dataset, prints a revenue report
pytest -q                       # 12 unit tests
ruff check .                    # lint
```
Output files appear in `local_output/` (same layout as S3: `orders/year=.../month=.../*.csv.gz` and `quarantine/`).

## Part 2 - Deploy to AWS
Prerequisites: AWS account + CLI configured with admin credentials (one time only), Terraform >= 1.10, a GitHub repo.

### Step 1: bootstrap (once, from your laptop)
Creates the Terraform state bucket, the GitHub OIDC provider and the deploy role.
```bash
cd terraform/bootstrap
terraform init
terraform apply -var="github_repo=YOUR_GITHUB_USER/YOUR_REPO" -var="aws_region=eu-west-1"
# note the outputs: state_bucket and github_role_arn
```

### Step 2: configure GitHub
Repo -> Settings:
- **Secrets and variables -> Actions -> Secrets:** `AWS_ROLE_ARN` = the `github_role_arn` output
- **Variables:** `AWS_REGION` = `eu-west-1` (same region as above), `TF_STATE_BUCKET` = the `state_bucket` output
- **Environments -> New environment `production`** -> add yourself as a required reviewer (manual approval gate)

### Step 3: ship it
```bash
git init && git add . && git commit -m "initial" && git branch -M main
git remote add origin https://github.com/YOUR_GITHUB_USER/YOUR_REPO.git && git push -u origin main
```
Workflows:
| Workflow | Trigger | What it does |
|---|---|---|
| `ci.yml` | PRs and non-main pushes | ruff lint, pytest, `terraform fmt -check` + `validate`, Trivy IaC scan |
| `deploy.yml` | push to `main` | tests -> (approval) -> `terraform plan` + `apply` -> smoke test: uploads the dataset, waits for output, runs an Athena query |
| `destroy.yml` | manual | tears everything down after typing `destroy` |

### Step 4: query your data
In the Athena console choose workgroup `sales-pipeline-dev-wg` and database `sales_pipeline_dev_db`, then run the saved queries
(`revenue-by-category`, `monthly-revenue`) or your own:
```sql
SELECT country, round(sum(total_amount),2) AS revenue
FROM orders WHERE year = 2024 GROUP BY country ORDER BY revenue DESC;
```
To feed it new data, upload any CSV with the same columns to `s3://<raw-bucket>/incoming/`.
Quarantined rows are in `s3://<curated-bucket>/quarantine/`.

### Optional: deploy manually without CI
```bash
cd terraform
terraform init -backend-config="bucket=<state_bucket>" -backend-config="key=sales-pipeline/dev.tfstate" -backend-config="region=eu-west-1"
terraform plan -out tfplan && terraform apply tfplan
aws s3 cp ../data/sales_orders.csv s3://$(terraform output -raw raw_bucket)/incoming/sales_orders.csv
```

## Cost
All serverless and pay-per-use. This dataset (about 0.4 MB) costs a few cents or less to run; Athena is capped at 1 GB scanned per query.
Run `destroy.yml` (or `terraform destroy`) when done, then `terraform destroy` in `terraform/bootstrap` to remove the state bucket.

## Design decisions (good interview talking points)
- **Event-driven and serverless:** no servers to patch, scales to zero.
- **Data quality with quarantine:** bad rows are never silently dropped; each carries a `reject_reason`.
- **Idempotent:** output keys derive from the input file name, so re-uploading overwrites instead of duplicating.
- **Partition projection:** Athena needs no crawler or `MSCK REPAIR`, so new partitions are queryable immediately.
- **Least privilege:** Lambda role can only read `raw/incoming/*` and write `curated/orders/*` and `quarantine/*`.
- **Secure by default:** encrypted buckets, public access blocked, TLS-only, versioned raw bucket, log retention, error alarm.
- **Keyless CI/CD:** GitHub OIDC assumes a role; state is in S3 with native locking (`use_lockfile`).
- **Zero-dependency Lambda:** pure Python standard library, so there are no layers or packaging steps.

## Ideas to extend
Convert the curated output to Parquet (add the AWS SDK for pandas Lambda layer), add a Glue Data Quality ruleset,
orchestrate with Step Functions, add a dead-letter queue, publish a QuickSight dashboard, or add a `staging` environment using the `environment` variable.

## Project layout
```
data/sales_orders.csv          sample dataset
scripts/                       generate_dataset.py, run_local.py, smoke_test.py
src/transform/handler.py       Lambda code (pure Python)
tests/                         unit tests
terraform/                     main stack (S3, Lambda, IAM, Glue, Athena, alarms)
terraform/bootstrap/           one-time state bucket + GitHub OIDC role
.github/workflows/             ci.yml, deploy.yml, destroy.yml
```
