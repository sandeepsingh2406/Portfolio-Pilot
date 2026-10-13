# Terraform Infrastructure

This directory contains the Terraform configurations for Portfolio Pilot's AWS infrastructure.

## Structure

Each area of infrastructure has its own independent Terraform directory:

- **`2_sagemaker/`** — SageMaker Serverless endpoint for embeddings
- **`3_ingestion/`** — S3 Vectors, ingest Lambda, and API Gateway for document ingestion
- **`4_researcher/`** — Lambda (container image, public Function URL) running the research agent
- **`5_database/`** — Aurora Serverless v2 PostgreSQL with the Data API
- **`6_agents/`** — Lambda functions for the agent orchestra plus the SQS orchestration queue
- **`7_frontend/`** — API Lambda, API Gateway, S3 static site, and CloudFront
- **`8_enterprise/`** — CloudWatch dashboards for Bedrock/SageMaker usage and agent performance

## Key Design Decisions

### Why separate directories?

1. **Independent deployment** — each area can be deployed or destroyed without affecting the others
2. **Reduced risk** — a mistake in one stack doesn't touch infrastructure that's already running
3. **Clear dependencies** — later stacks consume outputs from earlier ones, making the deployment order explicit

### Why local state?

1. **Simplicity** — no S3 state bucket or DynamoDB lock table to provision up front
2. **Zero dependencies** — you can start deploying immediately
3. **Security** — state files are gitignored automatically and never leave your machine

In a team setting you'd normally move to remote state (S3 + DynamoDB locking) so multiple people can collaborate safely — see **Production Considerations** below.

## Usage

For each stack:

```bash
cd terraform/<stage>        # e.g. 2_sagemaker, 3_ingestion, ...

terraform init               # only needed once per directory
terraform plan                 # review what will be created
terraform apply                 # deploy
terraform destroy                # tear down, when you're done with that stage
```

Deploy in order, since later stages depend on earlier outputs: `2_sagemaker → 3_ingestion → 4_researcher → 5_database → 6_agents → 7_frontend → 8_enterprise`.

## Environment Variables

Some Terraform configurations read values that originate from your root `.env` file (copy them into each stage's `terraform.tfvars`):

- `OPENAI_API_KEY` — agent tracing / the researcher agent
- `ALEX_API_ENDPOINT`, `ALEX_API_KEY` — ingestion API Gateway endpoint and key (stage 3)
- `AURORA_CLUSTER_ARN`, `AURORA_SECRET_ARN` — Aurora cluster and credentials (stage 5)
- `VECTOR_BUCKET` — S3 Vectors bucket name (stage 3)
- `BEDROCK_MODEL_ID`, `BEDROCK_REGION` — Bedrock model configuration (stage 6)
- `CLERK_JWKS_URL` — JWT validation for the frontend API (stage 7)

## State Management

- Each directory keeps its own `terraform.tfstate` file, stored locally and gitignored
- `terraform destroy` in a directory only removes that stack's resources
- If you lose a state file, you'll need to either re-import the existing resources or recreate them

## Production Considerations

This local-state, per-stack layout is optimized for a single developer iterating quickly. For a real production deployment, consider:

- **Remote state** — S3 backend with DynamoDB state locking
- **Modules** — share common configuration across environments
- **Workspaces** — manage dev/staging/prod from the same configuration
- **CI/CD** — automated plan/apply pipelines instead of running Terraform by hand

## Troubleshooting

**State conflicts** — each directory has independent state. To import an existing resource Terraform has lost track of:
```bash
terraform import <resource_type>.<resource_name> <resource_id>
```

**Missing dependencies** — make sure you've deployed earlier stages and copied their outputs into this stage's `terraform.tfvars` / the root `.env`.

**Clean slate** for a single stack:
```bash
terraform destroy
rm -rf .terraform terraform.tfstate*
terraform init
```
