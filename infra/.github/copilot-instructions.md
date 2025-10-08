## Purpose

These notes are written for automated coding agents (and humans) to be immediately productive in the infra/ workspace of this repository. They summarize the big-picture architecture, developer workflows, repository conventions, and exact examples of commands and files to change.

## Big picture (what this repo contains and why)
- This folder holds Terraform-based infrastructure for a blockchain DApp. The top-level structure is:
  - `infra/global/` — shared resources (CloudTrail, IAM, S3 buckets)
  - `infra/env/{dev,staging,prod}/` — environment-specific stacks (each has `main.tf` and `backend.tf`)
  - `infra/modules/` — reusable Terraform modules (examples: `eks/`, `rds-postgres/`, `redis/`, `vpc/`, `iam-roles/`)
  - `infra/versions.tf` — required Terraform and provider versions (terraform >= 1.5, aws ~> 5.0, kubernetes ~> 2.0)

Why: infra is separated by environment and reuses modules for consistency. Global resources are provisioned once (account-level), then per-environment stacks consume modules.

## CI / deployment workflows (what agents should mirror or use)
- Workflows are in `infra/ci-cd/workflows/`:
  - `terraform-ci.yml` — PR and push plan jobs for `infra/**` (runs `terraform init` + `terraform plan` per environment)
  - `deploy-infra.yml` — manual workflow to run `terraform apply` per environment (deploy order: global -> dev -> staging -> prod)
  - App/backend pipelines (see `deploy-app.yml` and `deploy-backend.yml`) show how images are built and deployed.

Concrete commands agents may run locally when editing infra (examples):
```
cd infra/global
terraform init
terraform plan -var="account_id=248982879830"

cd infra/env/dev
terraform init
terraform plan
terraform apply -auto-approve
```

For application and backend builds the CI uses these commands (copy in local testing):
```
cd app
npm ci
npm test
npm run build

cd backend
go test ./...
go build -o main cmd/server/main.go
```

Image build & deploy pattern (from workflows):
- Build Docker image in `app` or `backend`.
- Push to ECR using `aws-actions/amazon-ecr-login` credentials and `docker build`/`docker push`.
- Backend deployment uses `aws eks update-kubeconfig` then `kubectl set image deployment/...`.

## Project-specific conventions & patterns
- Environment layout: each environment has its own directory under `infra/env/`. Changes to a single environment should be limited to that folder unless intentionally changing shared/global resources.
- Module usage: prefer adding/updating modules under `infra/modules/` and referencing them from `infra/env/*/main.tf`. Example modules present: `eks/`, `rds-postgres/`, `redis/`, `vpc/`, `iam-roles/`.
- CI-first assumptions: workflows `cd` into infra directories and run terraform; do not assume a remote backend is configured locally. Check `infra/env/*/backend.tf` before running plan/apply (files exist but may be empty in this checkout).
- Provider & tooling versions: respect `infra/versions.tf` (Terraform >=1.5, aws provider ~>5.0, kubernetes ~>2.0).

## Integration points & secrets (what agents must not modify inadvertently)
- GitHub Actions expect the following secrets: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `S3_BUCKET_NAME`, `CLOUDFRONT_DISTRIBUTION_ID`, `EKS_CLUSTER_NAME`.
- Terraform plans in workflows sometimes pass `-var="account_id=248982879830"` — be careful when changing account-scoped resources.
- The app pipeline syncs `app/dist/` to an S3 bucket and invalidates CloudFront — changing this requires updating both CI and CloudFront settings.

## Helpful files to review when making changes
- `infra/versions.tf` — Terraform & provider constraints
- `infra/ci-cd/workflows/terraform-ci.yml` and `deploy-infra.yml` — canonical CI/CD commands and order of deployment
- `infra/modules/` — add or modify reusable components here
- `infra/env/*/main.tf` and `infra/env/*/backend.tf` — environment-specific entrypoints

## Safe edit guidance for automated agents (do this, not that)
- Do: Make small, refactorable changes to module inputs, add variables with default values, and update module usage in `infra/env/dev` for testing.
- Do: Run `terraform init` and `terraform plan` in the target env directory before proposing apply-ready changes. Include the exact plan output in PR description.
- Don't: Run `terraform apply` against `prod` or push CI changes that trigger production deploys without an explicit human approval step. The repo's `deploy-infra.yml` is manually dispatched and staged; respect that.
- Don't: Edit or embed secrets in code. Reference secrets via GitHub Actions or the Terraform variable/secret injection pattern.

## Example PR description template for infra changes
Short summary of change

- Tested: `cd infra/env/dev && terraform init && terraform plan` (attach plan output)
- Files changed: list module and env files
- Migration notes: any state changes, `account_id`, or resource replacements

## Final notes & assumptions made from repo
- Many `*.tf` files (like `backend.tf`, some `global` files) are present but empty in this checkout. Verify remote state/backends before running destructive ops.
- CI demonstrates how the repository expects builds and deployments to be performed — follow those steps when automating tasks.

If any section is unclear or you'd like me to include more examples (for example, sample `terraform plan` output parsing or an EKS deploy snippet), tell me which part to expand.
