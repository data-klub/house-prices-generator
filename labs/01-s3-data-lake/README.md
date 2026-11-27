# Data Klub Lab 01: an S3 data lake for housing listings

**Data Klub** is a peer group of data engineers who learn by building production-style
pipelines together. Lab 01 is the first hands-on AWS exercise. Python generates synthetic Warsaw
flat listings, pandas analyses them, and the files land in Amazon S3 as date-partitioned Parquet,
the layout real data lakes use.

This lab lives in the [house-prices-generator](../../README.md) repo: its small `generate.py` is
where the full generator started. The club's main project,
[DataKlubform](https://github.com/data-klub/DataKlubform), is a medallion lakehouse on Databricks
fed by that generator. DataKlubform runs on Databricks Free Edition and doesn't need AWS, so this
lab is optional practice for anyone who wants to learn AWS.

**Cost:** the AWS free plan covers this lab; a full month of daily runs costs a fraction of a cent.
Set a zero-spend budget before you build anything (see AWS setup below).

Data Klub meets every Wednesday at 19:00 (Warsaw time) on Discord. Questions go there too.

All data is synthetic. No real listings and no credentials are stored in this repo.

## Architecture

**In AWS (runs daily on its own):**

```
EventBridge Scheduler (06:00 Europe/Warsaw)
   └──► Lambda generate-listings (Python 3.12 + AWSSDKPandas layer)
           └──► s3://<bucket>/raw/listings/ingest_date=YYYY-MM-DD/listings.parquet
                   └──► Glue Data Catalog table data_klub.listings ──► Athena SQL
```

**Locally (learning and checks):**

```
generate.py ──► data/listings.parquet ──► analyze.py   (pandas: medians by district, filters)
                         └──► upload.py ──► S3 ──► read back into pandas (round-trip check)
```

| Path | What |
|---|---|
| `lambda/lambda_function.py` | The deployed Lambda. Seeded by date, so a rerun of the same day gives the same data. `{"date": "YYYY-MM-DD"}` backfills |
| `athena/queries.sql` | Table DDL, partition repair, and the queries |
| `generate.py`, `analyze.py`, `upload.py` | Local generator, pandas analysis, boto3 round trip |

- **Deterministic generator:** a fixed seed means the same 1,000 rows every run.
- **Parquet:** columnar and typed. 1,000 rows take about 21 KB.
- **Hive-style partitions** (`ingest_date=…`): one folder per daily load, never overwritten. Spark,
  Databricks and Athena can skip partitions they don't need.
- **No keys in code:** boto3 and the AWS CLI read credentials from the local AWS config.

## AWS setup

- Root user protected with MFA (a passkey) and a zero-spend budget alert. Root is used only for setup.
- A private S3 bucket (`eu-central-1`) with Block Public Access on and SSE-S3 encryption.
- An IAM user for the CLI and SDK with **least privilege**: one inline policy that allows only
  this bucket. It can't list other buckets, create buckets or touch IAM.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Sid": "ListThisBucket", "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::<bucket>" },
    { "Sid": "ReadWriteObjectsInThisBucket", "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::<bucket>/*" }
  ]
}
```

`ListBucket` acts on the bucket ARN; object actions need the `/*` object ARN. Mixing them up is
the most common S3 policy bug.

Each workload has its own role and no stored keys:

| Identity | Can do |
|---|---|
| IAM user `lab-cli` (laptop) | List, read, write and delete objects in this bucket only |
| Lambda execution role | `s3:PutObject` on `raw/listings/*` only (no read, list or delete), plus its own logs |
| Scheduler role | Invoke the one Lambda function |

**Lambda gotcha:** layers with compiled code (numpy, pandas, pyarrow) must match the function's
Python version **and** CPU architecture. An x86_64 layer on an arm64 function fails with
`Unable to import required dependency numpy`.

**Sign-up tip:** use the classic "(advanced)" AWS sign-up. Accounts from the newer project-style
sign-up can be pinned to one region by a service control policy (`explicit deny in a service
control policy`), and they have no root login.

## Run it

Requires [uv](https://docs.astral.sh/uv/) and the [AWS CLI](https://aws.amazon.com/cli/), with `aws configure` done.

```bash
# from the repo root
uv sync --group lab
uv run python labs/01-s3-data-lake/generate.py   # write data/listings.parquet
uv run python labs/01-s3-data-lake/analyze.py    # pandas exploration
uv run python labs/01-s3-data-lake/upload.py     # upload to S3, list partitions, read back
```

Set `BUCKET` in `upload.py` to your own bucket name.

## Skills covered

S3 (buckets, keys, prefixes, partitioned layout), IAM users and access keys, the AWS CLI,
boto3 (clients, paginators, streaming bodies, `ClientError`), least-privilege IAM, root MFA,
explicit deny (SCP) vs implicit deny (no allowing policy), Lambda (layers, execution roles,
cold starts), EventBridge Scheduler (cron), Athena and the Glue Data Catalog (external tables,
partitions, partition pruning),
and pandas (groupby, boolean filters, medians).

## Status (2026-11-18)

- The daily schedule is live. Every run since 2026-11-16 at 06:00 Europe/Warsaw has landed
  without anyone touching it.
- Partitions in S3: 2026-11-14 (manual backfill) through 2026-11-18, with 1,000 rows each.
- Athena median price per m² by district matches pandas to within 0.25%.

## Roadmap

Done:

- [x] Least-privilege IAM policy scoped to one bucket
- [x] Query the Parquet files with SQL in Athena (Glue Data Catalog, partitions), reconciled against pandas
- [x] Run the generator daily with Lambda and EventBridge Scheduler, with date-based backfills

Next, as a one-week sprint:

- [ ] Day 1: backfill from 2026-10-01, Athena partition projection, CloudWatch alarm with SNS email on Lambda errors
- [ ] Day 2: Glue PySpark job, bronze → silver as an Apache Iceberg table
- [ ] Day 3: dbt-athena gold model (median price per m² by district per day) with dbt tests
- [ ] Day 4: Step Functions orchestration (Lambda → Glue → gold refresh), with retries and failure alerts
- [ ] Day 5: infrastructure as code with AWS CDK (Python)
- [ ] Day 6: Tableau dashboard on the Athena gold layer, plus an architecture diagram
- [ ] Day 7: review, plus AWS Cloud Practitioner exam preparation
