# Cloud-Native LLMOps on AWS

An asynchronous LLM service that explores what changes when a small FastAPI application is operated as a real cloud workload. It includes durable jobs, repeatable evaluation, infrastructure as code, deployment gates, metrics, tracing, and rollback paths.

The project runs with a deterministic local provider by default, so its behaviour can be tested without AWS credentials or model calls.

## Architecture

```text
client -> WAF / ALB -> FastAPI -> DynamoDB
                         |
                         +-> SQS -> worker -> Amazon Bedrock

GitHub Actions -> ECR -> ECS Fargate
Terraform      -> networking, IAM, data, monitoring, and deployment resources
```

## Engineering decisions

- Jobs use conditional DynamoDB updates so duplicate SQS deliveries are idempotent.
- Retryable failures stay on the queue; exhausted messages move to a dead-letter queue.
- Prompts travel in the queue payload but are excluded from logs, traces, and stored job records.
- API and worker images run as non-root users with read-only root filesystems.
- GitHub Actions uses AWS OIDC instead of long-lived access keys.
- Staging and production promote the same scanned image digest instead of rebuilding it.
- CloudWatch metrics, OpenTelemetry traces, request IDs, and structured errors make failures traceable across the API and worker.

## Run locally

```bash
cp .env.example .env
docker compose up --build
```

Check the service at `http://localhost:8000/health`. With `LLM_PROVIDER=local`, generation is deterministic and does not call an external model.

Submit an asynchronous job:

```bash
curl -X POST http://localhost:8000/v1/jobs \
  -H "Content-Type: application/json" \
  -d '{"prompt":"hello"}'
```

Use the returned job ID with `GET /v1/jobs/{job_id}`.

## Tests and evaluation

```bash
python -m pytest services/api/tests services/worker/tests evals loadtests
python -m evals.run_eval
```

The test suite covers API behaviour, authentication, provider selection, durable job transitions, duplicate delivery, retry handling, telemetry, and deployment-policy checks.

## AWS mode

Set `LLM_PROVIDER=bedrock` and choose a model with `BEDROCK_MODEL_ID`. In ECS, the task role supplies credentials. Setting `JOB_BACKEND=aws` moves job state to DynamoDB and work distribution to SQS.

Terraform is split into reusable modules and `dev`, `staging`, and `production` environments:

```bash
terraform -chdir=terraform/environments/dev init
terraform -chdir=terraform/environments/dev validate
terraform -chdir=terraform/environments/dev plan
```

See [`terraform/README.md`](terraform/README.md) for state, cost, and environment details.

## Delivery flow

1. Pull requests run formatting, tests, evaluation, and dependency checks.
2. A successful `master` build creates immutable API and worker images and deploys development.
3. Staging promotion reuses those image digests and runs integration and performance checks.
4. Production uses a protected environment, an error-budget gate, and blue/green deployment.
5. A failed rollout restores the previous task definitions.

Operational procedures live in [`docs/runbooks`](docs/runbooks/README.md). They cover queue backlog, model degradation, deployment regression, infrastructure drift, backup recovery, and supply-chain verification.

## Repository layout

| Path | Purpose |
| --- | --- |
| `services/api` | FastAPI endpoints and job submission |
| `services/worker` | SQS worker and durable job processing |
| `services/common` | Shared LLM and observability code |
| `evals` | Deterministic local and remote evaluation |
| `loadtests` | Bounded staging load generator |
| `terraform` | AWS modules and environment stacks |
| `.github/workflows` | CI, deployment, monitoring, and drift checks |

This repository is a portfolio implementation, not a claim that the included defaults fit every production workload. Thresholds, retention, capacity, and cost controls are exposed as configuration because they depend on actual traffic and operating requirements.
