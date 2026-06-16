## `resource` Block

The `resource` block is the core element in Terraform. It declares infrastructure that Terraform should **create, update, and destroy** to match the desired configuration.

Syntax:

```hcl
resource "<TYPE>" "<NAME>" {
  # arguments and nested blocks
}
```

- **Type** — Identifies what to manage (e.g., `aws_dynamodb_table`). Defined by the provider and documented in the Terraform Registry.
- **Name** — A local label you choose. Used to reference the resource elsewhere in the configuration. Must be unique per resource type within a module.
- **Arguments** — Configure the resource (e.g., `billing_mode`, `runtime`). Required vs. optional arguments depend on the resource type.
- **Nested blocks** — Structured sub-configuration (e.g., `attribute {}`, `environment {}`).

> **Exam tip:** Address a resource as `<TYPE>.<NAME>.<ATTRIBUTE>` — for example, `aws_iam_role.lambda_iam_role.arn`. This differs from `data` sources (`data.aws_ami.example.id`) and `module` outputs (`module.vpc.vpc_id`).

### Resource vs. `data` Source

| Block | Purpose | Lifecycle |
|---|---|---|
| `resource` | Manage infrastructure Terraform creates and owns | Create, read, update, delete |
| `data` | Read existing infrastructure or computed values | Read only — never created or destroyed by Terraform |

Use `resource` when Terraform should manage the object. Use `data` when you need information about something that already exists or is computed outside Terraform (e.g., latest AMI, current AWS account ID).

> **Common mistake:** Using a `resource` block for objects you did not create and do not want Terraform to destroy (e.g., a shared VPC managed by another team). Use a `data` source instead.

### Meta-Arguments

Meta-arguments apply to any resource block, regardless of provider:

| Meta-argument | Purpose |
|---|---|
| `provider` | Select a non-default or aliased provider (see [[04_Terraform_Providers#Multiple Provider Instances (Aliases)]]) |
| `depends_on` | Declare explicit dependency when references alone are not enough |
| `count` | Create multiple similar resources from one block |
| `for_each` | Create multiple resources from a map or set |
| `lifecycle` | Control create/destroy behavior (`create_before_destroy`, `prevent_destroy`, `ignore_changes`) |

> **Exam tip:** `count` and `for_each` are mutually exclusive on the same block. Prefer `for_each` for most production use cases — resource addresses use map keys instead of numeric indexes.

### Example of a Resource

```hcl
resource "aws_dynamodb_table" "basic_dynamodb_table" {
  name         = local.dynamodb_table_name
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "id"

  attribute {
    name = "id"
    type = "S"
  }

  ttl {
    attribute_name = "ttl"
    enabled        = true
  }

  tags = local.common_tags
}
```

- Top-level keys (`name`, `billing_mode`) are **arguments**.
- `attribute {}` and `ttl {}` are **nested blocks** — common on AWS resources.
- After `terraform apply`, Terraform stores resource attributes in **state** and tracks them on future plans.

### Resource Referencing

Resource referencing connects blocks by passing values from one resource to another. When one resource references another's attributes, Terraform creates an **implicit dependency** and orders operations correctly.

Reference syntax: `<TYPE>.<NAME>.<ATTRIBUTE>`

```hcl
resource "aws_lambda_function" "basic_lambda_function" {
  filename      = data.archive_file.basic_lambda.output_path
  function_name = local.lambda_function_name
  description   = "Basic lambda function deployed with Terraform"
  role          = aws_iam_role.lambda_iam_role.arn   # implicit dependency on IAM role
  handler       = "lambda_function.lambda_handler"
  code_sha256   = data.archive_file.basic_lambda.output_base64sha256

  runtime = "python3.12"
  timeout = 10

  environment {
    variables = {
      DYNAMODB_TABLE_NAME = aws_dynamodb_table.basic_dynamodb_table.name
      ENVIRONMENT         = var.environment
      LOG_LEVEL           = "info"
    }
  }

  tags = local.common_tags
}
```

This example shows three reference types in one block:

- **Resource** — `aws_iam_role.lambda_iam_role.arn` (managed by Terraform)
- **Data source** — `data.archive_file.basic_lambda.output_path` (read-only)
- **Variable** — `var.environment` (input from outside the resource)

> **Real-world:** Chaining references (Lambda → IAM role → DynamoDB table name) is how Terraform builds a dependency graph. Avoid hardcoded ARNs or table names — they break portability across environments.

> **Common mistake:** Referencing a resource before it exists in configuration, or using the wrong attribute name from the provider docs. Run `terraform plan` to catch missing or invalid references early.

> **Exam tip:** Removing a `resource` block from configuration and running `terraform apply` schedules that resource for **destroy**. Terraform does not delete resources that are only removed from references but still declared in code.

See also: [[01_Fundamentals#Resource Referencing]] and [[02_Terraform_Workflow#Dependencies]].

## `data` Block

The `data` block reads information Terraform needs at plan/apply time **without managing lifecycle** of that object. Terraform never creates, updates, or destroys data sources.

Two common use cases:

| Use case | Example | What it reads |
|---|---|---|
| **Existing infrastructure** | `data "aws_vpc" "shared"` | A VPC already created outside this configuration |
| **Computed / filtered values** | `data "aws_iam_policy_document" "lambda_assume_role"` | A policy document built from HCL — not an existing AWS object |
| **Provider defaults** | `data "aws_availability_zones" "available"` | Current account/region metadata |

- Retrieve details of existing or computed values to reference in other blocks.
- Resolve dependencies when one block needs properties from another source.
- Avoid hardcoded values by pulling data dynamically at plan time.

> **Exam tip:** `data` sources are **read-only**. Removing a `data` block does **not** destroy anything in the cloud. Contrast with `resource` blocks (see [[#`resource` Block]]).

### Referencing a `data` Block

Reference syntax: `data.<TYPE>.<NAME>.<ATTRIBUTE>`

```hcl
data.aws_iam_policy_document.lambda_assume_role.json
aws_vpc.shared.id
var.environment
```

### Usage Example

```hcl
data "aws_iam_policy_document" "lambda_assume_role" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "lambda_iam_role" {
  name_prefix        = "${local.name_prefix}-lambda-role-"
  assume_role_policy = data.aws_iam_policy_document.lambda_assume_role.json

  lifecycle {
    create_before_destroy = true
  }
}
```

- `aws_iam_policy_document` generates JSON — it does not read an existing IAM policy from AWS.
- The role references `.json` from the data source, creating an implicit dependency.

> **Common mistake:** Treating every `data` block as "existing infrastructure." Policy documents, AMI filters, and caller identity are computed at plan time, not fetched as managed resources.

> **Real-world:** Use `data` sources for shared networking (VPC, subnets) owned by a platform team. Use `resource` blocks only for infrastructure your Terraform workspace owns.

See also: [[01_Fundamentals#Resource Referencing]] — `data` references follow the same dependency rules as resources.

## `variable` Block

The `variable` block defines **inputs** to a module or root configuration. It avoids hardcoding values and makes configurations reusable across environments (dev, staging, prod).

- **Dynamic inputs** — Pass different values per environment or scenario.
- **Centralized values** — Change once, apply everywhere the variable is referenced.
- **Reusability** — Share the same module or pattern across projects and teams.

### Usage Example

```hcl
variable "application_name" {
  description = "Project or application name. Used as a prefix for resource naming."
  type        = string
  default     = "terraform-workflow"

  validation {
    condition     = length(var.application_name) >= 3 && length(var.application_name) <= 20
    error_message = "Application name must be between 3 and 20 characters."
  }
}

variable "environment" {
  description = "Deployment environment (dev, staging, prod). Used as a suffix for resources."
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Invalid environment. Must be one of: dev, staging, prod."
  }
}
```

Reference a variable anywhere in the configuration: `var.environment`

> **Exam tip:** `validation` blocks run at plan time. The `condition` must evaluate to `true` or Terraform fails before apply.

### Variable Types

**Primitive types** (most common on the exam):

| Type | Example value |
|---|---|
| `string` | `"prod"` |
| `number` | `3` |
| `bool` | `true` |

**Complex types**:

| Type | Description | Example |
|---|---|---|
| `list(<TYPE>)` | Ordered sequence; index starts at `0` | `["us-east-1a", "us-east-1b"]` |
| `map(<TYPE>)` | Key-value pairs | `{ dev = "t3.micro", prod = "t3.large" }` |
| `set(<TYPE>)` | Unordered collection of unique values; not indexable | `toset(["a", "b"])` |
| `object({...})` | Structured shape with named attributes | `object({ name = string, port = number })` |
| `tuple([...])` | Fixed-length sequence with typed positions | `tuple([string, number])` |

> **Common mistake:** Indexing a `set` directly (e.g., `my_set[0]`). Convert with `tolist()` first, or use a `list` if order matters.

### Assigning Values to Variables

| Method | Example | Notes |
|---|---|---|
| **Default in `variable` block** | `default = "dev"` | Lowest precedence |
| **Environment variable** | `TF_VAR_environment=prod` | Must prefix with `TF_VAR_` |
| **`.tfvars` file** | `environment = "prod"` in `terraform.tfvars` | Auto-loaded from working directory |
| **Command line** | `terraform plan -var="environment=prod"` | Highest precedence |

Use `-var-file="prod.tfvars"` to load a specific file. Exclude environment-specific `.tfvars` from version control when they contain sensitive values.

### Order of Precedence

When multiple sources define the same variable, **the first item in this list wins** (highest precedence):

1. **Command-line flags** — `-var` and `-var-file`
2. **`*.auto.tfvars`** — loaded automatically, lexical filename order
3. **`terraform.tfvars`** / `terraform.tfvars.json`
4. **Environment variables** — `TF_VAR_<name>`
5. **`default` in the `variable` block** — used only when nothing else supplies a value

```mermaid
flowchart TD
  A["-var / -var-file (highest)"] --> B["*.auto.tfvars"]
  B --> C["terraform.tfvars"]
  C --> D["TF_VAR_* env vars"]
  D --> E["variable default (lowest)"]
```

> **Exam tip:** `-var` on the command line **always wins** over `terraform.tfvars` and `TF_VAR_*`. Defaults never override an explicitly set value.

> **Real-world:** Store non-sensitive defaults in `terraform.tfvars.example` (committed). Keep real `terraform.tfvars` and secrets in `.gitignore`.

Optional arguments worth knowing:

- `sensitive = true` — hides value in CLI output (still stored in state).
- `nullable = false` — variable must receive an explicit value (Terraform 1.1+).

## `output` Block

The `output` block exposes values **after apply** — IP addresses, ARNs, endpoints, or any attribute others need without opening the provider console.

- Display key infrastructure details in the CLI after deployment.
- Pass data from child modules to the root module (or to other modules via root outputs).
- Feed CI/CD pipelines and automation via `terraform output -json`.

Reference from the CLI: `terraform output <NAME>` or `terraform output -json`.

### Usage Example

```hcl
output "lambda_metadata" {
  description = "Lambda function metadata"
  value = {
    name = aws_lambda_function.basic_lambda_function.function_name
    arn  = aws_lambda_function.basic_lambda_function.arn
  }
}

output "dynamodb_table" {
  description = "DynamoDB table metadata"
  value = {
    name = aws_dynamodb_table.basic_dynamodb_table.name
    arn  = aws_dynamodb_table.basic_dynamodb_table.arn
  }
}
```

### Sensitive Outputs

Mark outputs that contain secrets so Terraform redacts them in the CLI:

```hcl
output "database_endpoint" {
  description = "Database connection endpoint"
  value       = aws_db_instance.main.endpoint
  sensitive   = true
}
```

> **Exam tip:** `sensitive = true` on an `output` hides the value in normal CLI output. It does **not** remove the value from state — treat state as sensitive.

### Real-World Scenarios

1. Outputs print in the CLI after a successful `terraform apply`.
2. Values are stored in **state** and retrieved with `terraform output` or `terraform output -json`.
3. Root module **outputs** are public to module consumers. Child module outputs are passed to the root via the `module` block: `module.network.vpc_id`.
4. Use `sensitive = true` for credentials, tokens, or private endpoints.

> **Common mistake:** Expecting outputs during `terraform plan`. Outputs are evaluated at apply time and stored in state after a successful apply.

## `terraform` Block

The top-level `terraform` block sets **project-wide settings**: Terraform Core version, required providers, and backend configuration for remote state.

- **Version control** — Enforce compatible Terraform and provider versions across teams.
- **State storage** — Configure where state is persisted (local by default, remote for teams).
- **Consistency** — Prevent "works on my machine" drift from version mismatches.

> **Exam tip:** The `terraform` block is meta-configuration. It does not create infrastructure — it configures how Terraform runs. Provider _requirements_ go here; provider _settings_ (region, credentials) go in `provider` blocks (see [[04_Terraform_Providers#Required Providers Block]]).

### Usage Example

```hcl
terraform {
  required_version = ">= 1.5.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    archive = {
      source  = "hashicorp/archive"
      version = "~> 2.4"
    }
  }

  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

| Setting | Purpose |
|---|---|
| `required_version` | Terraform CLI version constraint |
| `required_providers` | Provider source + version constraints |
| `backend` | Remote state location and locking |

### Version Constraints

Same operators apply to `required_version` and provider versions:

```hcl
required_version = "1.9.8"         # exact version only
required_version = ">= 1.9.8"      # minimum version
required_version = "~> 1.9.8"      # >= 1.9.8, < 1.10.0
required_version = ">= 1.5, < 2.0" # compound constraint
```

> **Common mistake:** Confusing `required_version` (Terraform **Core** binary) with `required_providers` version (provider **plugins**). They are independent constraints.

> **AWS SAA-C03 tie-in:** S3 backend + DynamoDB table for locking is the standard AWS pattern for team state management. Enable `encrypt = true` on the backend block.

See also: [[03_Terraform_File_Structure#Common Terraform Files]] — where to place the `terraform` block (`versions.tf` or `terraform.tf`).

## Configuration Blocks Summary

| Block | Purpose | Reference syntax |
|---|---|---|
| `terraform` | Version, providers, backend | N/A (meta-configuration) |
| `provider` | Platform connection settings — see [[04_Terraform_Providers]] | `provider = aws.west` (meta-argument) |
| `variable` | Input values | `var.<name>` |
| `output` | Exposed values after apply | `terraform output <name>` |
| `resource` | Managed infrastructure | `<type>.<name>.<attr>` |
| `data` | Read-only lookups | `data.<type>.<name>.<attr>` |

## Related Notes

- [[04_Terraform_Providers]] — provider configuration, aliases, and authentication
- [[01_Fundamentals]] — HCL basics and core components
- [[02_Terraform_Workflow#The `terraform init` command]] — when configuration is validated, planned, and applied
- [[03_Terraform_File_Structure]] — file organization, state files, and where to place blocks
- [[00_Index]] — study progress
