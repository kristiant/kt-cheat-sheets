# Terraform

**What it is:** An infrastructure-as-code tool that provisions cloud resources from declarative config (`.tf` files), tracking what exists in a **state** file.

**Why people use it:** Reproducible, version-controlled, reviewable infrastructure. Plan changes before applying, and tear it all down with one command — no clicking through consoles.

**Typically used for:** Provisioning cloud infra (VPCs, databases, Lambdas, queues, secrets) across AWS/GCP/Azure, and wiring an LLM app's backing services (S3, SQS, Secrets Manager, ECS/Lambda).

---

## Core workflow

```bash
terraform init        # download providers, set up backend (run once / on provider change)
terraform plan        # preview changes — read this before every apply
terraform apply       # create/update to match config (prompts for approval)
terraform destroy     # tear everything down
terraform fmt         # format files
terraform validate    # check syntax
```

---

## Anatomy

```hcl
# provider — what cloud, which region
provider "aws" {
  region = "ap-southeast-2"
}

# variable — typed input
variable "env" {
  type    = string
  default = "dev"
}

# resource — a thing to create
resource "aws_sqs_queue" "ingest" {
  name = "${var.env}-ingest-queue"
  tags = { Environment = var.env }
}

# output — surface a value after apply
output "queue_url" {
  value = aws_sqs_queue.ingest.url
}
```

```bash
terraform apply -var="env=prod"        # override a variable
TF_VAR_env=prod terraform apply        # or via env var
```

---

## Referencing & dependencies

Terraform infers order from references — no need to declare dependencies manually.

```hcl
resource "aws_lambda_function" "parser" {
  function_name = "parser"
  environment {
    variables = {
      QUEUE_URL  = aws_sqs_queue.ingest.url          # implicit dependency on the queue
      SECRET_ARN = aws_secretsmanager_secret.openai.arn
    }
  }
}

# explicit dependency when there's no reference
resource "aws_lambda_event_source_mapping" "trigger" {
  depends_on = [aws_iam_role_policy.lambda_sqs]
}
```

---

## State

Terraform records reality in `terraform.state`. **Store it remotely** for any team — a shared backend with locking prevents two people corrupting it.

```hcl
terraform {
  backend "s3" {
    bucket         = "mycorp-tfstate"
    key            = "app/prod.tfstate"
    region         = "ap-southeast-2"
    dynamodb_table = "tf-locks"     # state locking
    encrypt        = true
  }
}
```

```bash
terraform state list                    # what's tracked
terraform state show aws_sqs_queue.ingest
terraform import aws_sqs_queue.ingest <queue-url>   # adopt an existing resource
```

> ⚠️ State can contain secrets in plaintext. Always encrypt the backend and restrict access; never commit `terraform.state` to git.

---

## Advanced

### Modules (reusable, parameterised infra)

```hcl
module "ingest" {
  source      = "./modules/queue-worker"
  env         = var.env
  memory_size = 512
}

# reference module outputs
output "url" { value = module.ingest.queue_url }
```

### Loops — `for_each` and `count`

```hcl
# one queue per name (for_each — keyed, stable)
resource "aws_sqs_queue" "q" {
  for_each = toset(["parse", "ingest", "embed"])
  name     = "${each.key}-queue"
}

# N identical resources (count — index-based)
resource "aws_instance" "worker" {
  count         = 3
  instance_type = "t3.small"
}
```

> Prefer `for_each` over `count` when items have identities — removing one `count` item re-indexes and can destroy/recreate others.

### Conditionals & locals

```hcl
locals {
  is_prod   = var.env == "prod"
  log_level = local.is_prod ? "warn" : "debug"
}

resource "aws_lambda_function" "fn" {
  memory_size = local.is_prod ? 1024 : 256
}
```

### Provisioning an LLM app's backing services

```hcl
# secret for the model API key (value passed via TF_VAR_openai_api_key)
resource "aws_secretsmanager_secret" "openai" {
  name = "${var.env}/openai"
}
resource "aws_secretsmanager_secret_version" "openai" {
  secret_id     = aws_secretsmanager_secret.openai.id
  secret_string = jsonencode({ apiKey = var.openai_api_key })
}

# bucket for RAG source documents
resource "aws_s3_bucket" "docs" {
  bucket = "${var.env}-rag-docs"
}
```

> See [aws-secrets-manager.md](aws-secrets-manager.md) for reading the secret at runtime.

### Workspaces (multiple envs, one config)

```bash
terraform workspace new prod
terraform workspace select prod
# reference with terraform.workspace in config
```

> Many teams prefer separate state files / directories per env over workspaces, for stronger isolation.

---

## Practical recipes

**Preview only, save the plan, then apply exactly that:**
```bash
terraform plan -out=tfplan
terraform apply tfplan
```

**Target a single resource (surgical change):**
```bash
terraform apply -target=aws_sqs_queue.ingest
```

**Destroy one resource without touching the rest:**
```bash
terraform destroy -target=aws_lambda_function.parser
```

**See the full diff without colour (for CI logs / review):**
```bash
terraform plan -no-color
```

**Recreate a resource from scratch:**
```bash
terraform apply -replace=aws_lambda_function.parser
```

**Auto-approve in CI (⚠️ no human gate):**
```bash
terraform apply -auto-approve
```

---

## Tips

- Always read `terraform plan` before `apply` — it shows create/update/**destroy** counts.
- Remote, encrypted, locked backend (S3 + DynamoDB) for any shared project; never commit state.
- `for_each` over `count` when resources have stable identities.
- Pass secrets via `TF_VAR_*` env vars or a secrets backend — never hardcode in `.tf` files (they're in git).
- Keep `.tf` files small and modular; extract repeated patterns into modules.
- Pin provider versions in `required_providers` so applies are reproducible.
- `terraform fmt` + `validate` in CI keeps configs clean and catches errors before plan.
