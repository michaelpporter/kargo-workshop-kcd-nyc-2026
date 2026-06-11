# Kargo Workshop — KCD New York 2026

A hands-on workshop that builds a complete [Kargo](https://kargo.io) promotion pipeline for a
**guestbook** application whose backend is an **AWS Lambda** (deployed with Terraform). It exercises
the full breadth of Kargo: freight & promotions, verification, custom steps, `send-message`,
integration with external change management systems, email notifications, Terraform/IaC promotion,
and a multi-region production fan-out driven by an Argo CD `ApplicationSet`.

## Architecture

- **Frontend** (`app/`): a static guestbook served by Nginx. At container start it
  reads `LAMBDA_URL` and renders `config.js`. Guestbook entries are kept client-side;
  when a Lambda URL is configured the page also fetches the backend's greeting/version.
- **Backend** (`lambda/`): a small stateless Python Lambda packaged as a Zip by
  Terraform (`env/<stage>/terraform/`) — no ECR required.
- **Chart** (`charts/guestbook/`): renders the frontend Deployment/Service; Argo CD
  applies it per environment using `env/<stage>/k8s/values.yaml`.

The top-level `argocd/` (an AppProject + `ApplicationSet`) creates the five apps:
`guestbook-dev`, `guestbook-staging`, and `guestbook-prod-amer-east` / `-west` /
`-emea`.`

## Prerequisites

This needs to happen before we start the workshop steps.

First, **fork this repo on GitHub and clone your fork** locally. A couple of things still have to be
done by hand because they live in the Akuity dashboard:

- **Install the agents:** connect the Argo CD and Kargo agents into your cluster (from the Akuity
  dashboard).
- **Create an admin password** for Argo CD and Kargo.

Second, the script needs the [Argo CD
CLI](https://argo-cd.readthedocs.io/en/latest/cli_installation/) and [Kargo
CLI](https://docs.kargo.io/user-guide/installing-the-cli) on your `PATH` for the apply, project, and
secret steps. If you don't want to install the CLI, you can create the secrets manually in the UI
instead

Everything else is automated by [`scripts/setup.sh`](scripts/setup.sh). Run it from the root of your
clone:

```sh
./scripts/setup.sh
```

The script:

1. Detects your fork from the `origin` remote and rewrites every `steps/*/warehouse.yaml`,
   `steps/*/stages.yaml`, `argocd/application-set.yaml`, `charts/guestbook/values.yaml`, and
   `env/{dev,staging,prod}/terraform/env.auto.tfvars` to point at your fork / your `ghcr.io` image
   and your `participant` handle, then commits and pushes to `main` so GitHub Actions builds
   `ghcr.io/<you>/<repo>`. **After the build finishes, make the package public** (the script prints
   the settings URL).
2. Optionally logs you in to Argo CD and Kargo (you must pass `--argocd-host` and/or `--kargo-host`).
   Otherwise it assumes you are already logged in.
3. Applies the Argo CD `AppProject` + `ApplicationSet` from `argocd/`.
4. Creates the `guestbook` Kargo project.
5. Creates the Kargo project secrets (`git-creds`, `smtp-credentials`, `kargo-step-snow`,
   `aws-creds`), prompting for each credential (leave a prompt blank to skip it). For `git-creds`
   you'll need a GitHub PAT with `repo` scope; the ServiceNow and AWS credentials are provided during
   the workshop.

Useful flags (`./scripts/setup.sh --help` for the full list):

```sh
# Provide hosts to have the script log you in to both CLIs:
./scripts/setup.sh --argocd-host <your-argocd-host> --kargo-host https://<your-kargo-host>

# Override the auto-detected identity, or skip sections you've already done:
./scripts/setup.sh --owner <handle> --repo <name> --skip-secrets
```

## Workshop steps

Each step is a complete, applyable snapshot of the pipeline. If you need to catch up, start from
that step. If you installed the `kargo` CLI, you can apply the step manifests with:

```sh
kargo apply -f steps/<step>/
```

Each step directory has a brief README containing a summary of what the step adds.

1. [**Your first promotion**](steps/1-your-first-promotion/README.md) — a Warehouse and a single
   `dev` Stage; run your first promotion and explore the UI.
2. [**Building a pipeline**](steps/2-building-a-pipeline/README.md) — add `staging` downstream of
   `dev` and auto-promote `dev` via a promotion policy.
3. [**Multi-region prod**](steps/3-multi-region-prod/README.md) — add a `prod` gate plus three
   regional Stages that fan out via the Argo CD ApplicationSet.
4. [**Verification**](steps/4-verification/README.md) — add Argo Rollouts AnalysisTemplates so Kargo
   verifies freight after each promotion.
5. [**PR approval gate**](steps/5-pr-approval-gate/README.md) — turn `staging` into a PR gate that
   emails a link and blocks until merge.
6. [**ServiceNow**](steps/6-servicenow/README.md) — add a ServiceNow change-request gate to `prod`.
7. [**Notifications**](steps/7-notifications/README.md) — add an EventRouter that emails on
   promotion events.
8. [**Beyond Kubernetes**](steps/8-beyond-kubernetes/README.md) — apply the Lambda via Terraform and
   inject its URL into the app.
9. [**Custom steps**](steps/9-custom-steps/README.md) — run any container as a promotion step (Trivy
   scan + hello-world).

[`steps/complete/`](steps/complete/README.md) holds the finished pipeline (all steps combined) for reference.

## AWS setup for the Lambda

If you wanted to run this entirely on your own with a personal AWS account, the Terraform deploys a
Lambda using a **shared, pre-created execution role** so the shared workshop credentials only need
to manage Lambdas — they never provision IAM roles. An admin creates the role **once**:

```sh
aws iam create-role --role-name lambda-execution-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam attach-role-policy --role-name lambda-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

Its ARN should be used as the default of `var.execution_role_arn` in `env/<stage>/terraform/variables.tf`.

You then need an IAM user with Lambda management on `function:kargo-guestbook-*`, `iam:PassRole` on
the one execution role, and S3 access to the Terraform state bucket. The full policy is in
[`docs/workshop-aws-policy.json`](docs/workshop-aws-policy.json) for reference.
