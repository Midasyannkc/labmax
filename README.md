# LabMax

A small multi-tenant SaaS platform with the operations layer built around it: tenant isolation in
Postgres, fair background jobs, bounded load, safe releases, SLO alerting, Terraform for the AWS
layout, and six incidents that were reproduced, root-caused and fixed in the git history.

It was built as a homelab for a Senior DevOps Engineer posting at a multi-tenant GRC SaaS company
(AWS, Terraform, ECS, SQS, CI/CD, SLOs, canary and blue/green, SOC 2). What it proves, and what it
does not, is stated in [docs/coverage.md](docs/coverage.md).

![architecture](docs/charts/architecture.png)

## What runs and what does not

| Runs here, with tests | Written, not run | Documented only |
|---|---|---|
| Postgres 16 row-level security with a real role hierarchy | Terraform apply (no terraform binary, no AWS account) | SOC 2 and ISO 27001 control map |
| Redis Streams per-tenant queue and scheduler | GitLab CI and Azure DevOps pipelines | On-call rotation and runbooks |
| Bounded connection pool, 429 backpressure, client backoff | GitHub Actions run (workflow written; the suite runs locally) | Cross-region replication decision |
| Canary and blue/green over seeded simulated traffic | Prometheus and Grafana dashboards (not included) | |
| Bronze, silver, gold telemetry in DuckDB with quarantine | | |
| Checkov static scan of 28 Terraform resources | | |
| SQS redrive, S3, ECS, CloudWatch, IAM against moto | | |

Not covered: 5 to 8 years of production DevOps, a real on-call rotation, Kubernetes or EKS, Lambda,
a real Kafka cluster, AWS certifications. A project cannot supply those, and this one does not claim to.

## The scenario and the objective

LabMax serves five tenants. The availability SLO is **99.9% of non-health requests below HTTP 500
over 30 days** (docs/slo.md). The platform is judged per tenant, because one tenant failing can hide
inside a healthy global number.

## Data

- **Request telemetry:** [data/raw/requests.jsonl](data/raw/requests.jsonl), 40,111 lines, seeded and
  **synthetic**, with injected truncated lines, missing fields, clock skew and duplicates, and a
  ground-truth manifest ([requests.jsonl.manifest.json](data/raw/requests.jsonl.manifest.json)). The
  intended real source is the [NASA HTTP web-server logs](https://ita.ee.lbl.gov/html/contrib/NASA-HTTP.html)
  from the Internet Traffic Archive; that host was not reachable from the build sandbox, so it was not used.
- **Pipeline output:** [data/gold/](data/gold) holds silver, quarantine, gold tenant-hourly and gold SLO
  tables as parquet and csv.
- **Cost inputs:** utilisation is synthetic. Fargate prices are the published us-east-1 list rates
  ($0.04048 per vCPU-hour, $0.004445 per GB-hour) from the [AWS Fargate pricing page](https://aws.amazon.com/fargate/pricing/),
  entered by hand and not fetched in this build. Verify before relying on dollar figures.

## Incidents

Each incident has a **reproduce** commit (a failing test) followed by a **fix** commit. `python
scripts/build_artifacts.py` re-measures every number below.

### INC-1: one tenant's burst stalls another tenant (real Redis, virtual clock)
A noisy tenant enqueued 400 jobs of 50 ms; a quiet tenant sent 5 jobs a second later. One global stream
made the quiet tenant wait **19,000 ms**. **Root cause:** first-in first-out across tenants. **Fix:** one
stream per tenant, served round-robin; quiet tenant worst wait **0 ms** here (bounded by one in-flight
job in general). A work-conserving test proves total busy time is unchanged. ADR 001.

![INC-1](docs/charts/inc1_queue_wait.png)

### INC-2: a pooled connection keeps the previous tenant's identity (real Postgres)
After a tenant request on a size-1 pool, a health/export path with no tenant set read **1 row** that
belonged to that tenant. **Root cause:** `set_config(..., false)` is session-scoped and survives
`putconn`; the RLS policy reads it. **Fix:** transaction-local `set_config(..., true)`; untenanted
paths now fail closed with 0 rows. ADR 002, runbook for a suspected leak.

![INC-2](docs/charts/inc2_leak.png)

### INC-3: a spike exhausts Postgres connections (real Postgres, max_connections 30)
60 concurrent requests, one new connection each: **33 failed** with "remaining connection slots are
reserved". **Root cause:** no ceiling on connections and slow downstream work held on each. **Fix:** a
10-connection pool behind a semaphore that sheds excess load as `429` with `Retry-After`, and client
backoff with jitter. Result: **0 raw database errors**; 40 requests were shed and **all 60 were served**
once they retried. This is the rate-limit incident of the project.

![INC-3](docs/charts/inc3_pool.png)

### INC-4: a bad release reaches every user (seeded simulation, real decision code)
A release failing 20% of requests, shipped all at once and noticed by a human 3,000 requests later, gave
users **627 errors (3.14%)**. **Root cause:** no automated analysis between cutover and detection.
**Fix:** canary (5, 25, 50, 100% with at least 100 samples per stage and a 1% ceiling) held it to
**30 errors (0.15%)**; blue/green (smoke test, then watch the first 100 live requests) to **11
(0.055%)**. A healthy release is still promoted. The traffic is simulated; the rollout logic is what is tested.

![INC-4](docs/charts/inc4_release.png)

### INC-5: infrastructure code fails static security checks (real Checkov)
Checkov reported **17 failed, 45 passed** on the first modules: `sqs:*` and `s3:*` on `Resource "*"`, an
unencrypted unversioned S3 bucket with no public access block, SQS and log groups without KMS, no
retention, no VPC flow logs. **Fix:** scoped IAM, a KMS module with rotation, versioning, encryption,
lifecycle, flow logs, 365-day retention. Now **0 failed, 111 passed, 5 documented skips** (ADR 005), and
a test fails if the skip list grows. Terraform itself was never applied.

### INC-6: one malformed log line stops the telemetry pipeline (real DuckDB, data-quality incident)
A truncated write raised `JSONDecodeError` in the silver build and the run produced nothing. **Fix:**
quarantine with a reason. Of 40,111 bronze lines, 39,651 reached silver and **460 were quarantined**
(malformed 162, missing field 109, out-of-range timestamp 78, duplicate 111), reconciled **exactly**
against the generator's injected ground truth. Gold then showed tenant `globex` burning its budget at
**8.2x** (an injected export degradation) while the others stayed under 2x. Small tenants wobble near
1.0x on a handful of errors, which is why the alert rule uses two windows.

![SLO burn](docs/charts/slo_burn.png)

## Cost trade-offs

![cost](docs/charts/cost.png)

Rightsizing four Fargate services to keep p95 usage at or under 60% of allocation cuts the modelled
bill from **$680 to $369 per month** on synthetic utilisation; `search` was already right-sized and
left alone. The 60% target trades money for headroom: at 80% the saving grows, and so does the risk
that a lunch-rush spike (INC-3) lands on a saturated task. Per-tenant streams (ADR 001) cost one stream
per tenant instead of one queue; Redis Streams stands in for SQS and Kafka (ADR 004), so partition
rebalancing and exactly-once semantics are not exercised.

## Run it

```
pip install -r requirements.txt
make test                       # starts local Postgres 16 and Redis, runs 27 tests
python scripts/build_artifacts.py   # rebuilds data/, docs/results.json, docs/charts/
make scan                       # Checkov
```

The Postgres cluster runs as the `postgres` OS user with `max_connections = 30` (scripts/infra.sh).

## Layout

`labmax/` service, jobs, deploy, slo, telemetry, cost, api. `terraform/` seven modules and a prod env.
`tests/` one file per incident plus API, AWS (moto), Terraform policy and coverage. `docs/` coverage,
runbooks, ADRs, SLOs, control map, on-call design. `AI_DEV_LOG.md` how AI assistance was used and checked. `LabMax_executive_summary.pptx` the standalone 7-slide deck (source: `deck/build_deck.js`).

## Git history (through the commit before this README)

```
bf533e7 Christian Kouadio Kouassi Scaffold LabMax: infra script for local Postgres 16 and Redis, config, test fixtures
4a95898 Christian Kouadio Kouassi Add tenant-isolated Postgres schema with row-level security and a connection pool
645b91a Christian Kouadio Kouassi INC-2 reproduce: untenanted session reads rows from the previous tenant on a reused connection
d3b31ac Christian Kouadio Kouassi INC-2 fix: scope the tenant setting to the transaction with set_config(..., true)
6ec935e Christian Kouadio Kouassi INC-3 reproduce: 60 concurrent requests exhaust Postgres max_connections
31e88ca Christian Kouadio Kouassi INC-3 fix: bounded pool with admission control and client backoff
5944021 Christian Kouadio Kouassi INC-1 reproduce: a 400-job burst from one tenant delays another tenant's jobs by 19 s
de6f95b Christian Kouadio Kouassi INC-1 fix: per-tenant streams served round-robin
d19986e Christian Kouadio Kouassi INC-4 reproduce: a 20% error release serves 627 errors (3.14%) before a human rolls it back
8a58852 Christian Kouadio Kouassi INC-4 fix: canary and blue/green strategies with automatic rollback
ce52735 Christian Kouadio Kouassi INC-5 reproduce: Checkov fails 17 of 62 checks on the Terraform modules
0372b6d Christian Kouadio Kouassi INC-5 fix: least-privilege IAM, encrypted and versioned storage, flow logs, 365-day retention
d9bcee2 Christian Kouadio Kouassi INC-6 reproduce: one truncated log line raises JSONDecodeError and stops the pipeline
019e25f Christian Kouadio Kouassi INC-6 fix: quarantine bad lines with a reason; bronze, silver and gold layers
491c050 Christian Kouadio Kouassi Add HTTP API with 429 backpressure and metrics, Fargate rightsizing report, SLO tests
ce4b3a0 Christian Kouadio Kouassi Provision the AWS layout with boto3 against moto: SQS+DLQ redrive, versioned S3, alarm, ECS service, scoped IAM
6581680 Christian Kouadio Kouassi Add CI for GitHub Actions, GitLab and Azure DevOps (GitLab and Azure unrun); reproducible build script, data products and charts
c2f4a06 Christian Kouadio Kouassi Add runbooks, SLO doc, control map, on-call design, ADRs 001-004 and a coverage matrix with a drift test
```
# labmax
