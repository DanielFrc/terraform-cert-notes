## Organizing Code

- To avoid one large file that could be hard to maintain, it is better to separate the code by responsibilities.
- It is useful to group blocks and resources by functionality, i.e. all variable definitions in `variables.tf` or network resources in `network.tf`.
- In this way, the code is easy to read, understand, reuse, or maintain.

> **Real-world:** File names are a convention only — Terraform has no opinion on naming. Teams often split by domain (`network.tf`, `compute.tf`, `iam.tf`) or by environment layer. What matters is consistency within the team.

## Executing Code

Separate files do not affect functionality because of the way Terraform reads code. In simple terms, Terraform reads all `.tf` files in a directory as a single configuration, allowing references between files.

```hcl
# variables.tf
variable "vpc_cidr" {
  type = string
}

# main.tf
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr   # valid — same logical configuration
}
```

> **Exam tip:** All `.tf` and `.tf.json` files in the working directory are loaded together. Subdirectories are _not_ loaded unless they contain a module call.

## Common Terraform Files

| File | Purpose |
|---|---|
| `main.tf` | Primary infrastructure resources, or composition of resources defined in other files |
| `variables.tf` | Variable declarations (`variable` blocks) |
| `outputs.tf` | Output values exposed after apply (`output` blocks) |
| `providers.tf` | Provider configuration blocks |
| `versions.tf` | `terraform` block — required providers, version constraints, backend config |
| `terraform.tfvars` | Variable values for the current environment (often gitignored) |
| `*.auto.tfvars` | Automatically loaded variable values (also commonly gitignored for secrets) |

> **Best practice:** Put the `terraform` block (required providers, backend) in `versions.tf` or `terraform.tf`. Keep provider _configuration_ (region, credentials) in `providers.tf`.

Example `versions.tf`:

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}
```

## Additional Files

| File / Directory | Purpose | Version Control |
|---|---|---|
| `terraform.tfstate` | Current state mapping config → real resources | **Never commit** (contains sensitive data) |
| `terraform.tfstate.backup` | Backup of previous state before overwrite | **Never commit** |
| `.terraform.lock.hcl` | Locks provider versions selected by `init` | **Commit** |
| `.terraform/` | Downloaded providers and modules cache | **Gitignore** |
| `.gitignore` | Excludes state, `.terraform/`, secrets | **Commit** |

> **Exam tip:** State files may contain sensitive values in plain text (e.g., generated passwords). Remote backends with encryption at rest (S3 SSE, HCP Terraform) are strongly recommended for production.

> **AWS SAA-C03 tie-in:** A common production pattern is S3 for state storage + DynamoDB for state locking. This prevents two engineers or CI jobs from corrupting state with concurrent applies.

### Recommended `.gitignore` entries

```
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
!*.tfvars.example
.terraform.lock.hcl   # do NOT ignore — commit this file
```

Remove the last line if present in your `.gitignore` — `.terraform.lock.hcl` should be tracked.

## Working Directory

The **working directory** is the folder where you run Terraform commands. Terraform loads all configuration files in that directory (not subdirectories) and stores state relative to that directory unless a remote backend is configured.

Key rules:

- Run `terraform init`, `plan`, and `apply` from the same working directory (or the directory containing the root module).
- Each root module has its own state. Calling a child module does not create separate state — child module resources are tracked in the root module's state.
- Changing directories without re-initing can cause "backend not initialized" errors.

### Root Module vs. Child Module

- **Root module** — the directory where you run CLI commands. Owns the state file (or remote state key).
- **Child module** — called via `module` blocks from the root. Its resources are namespaced in state under `module.<name>`.

## Common Mistakes

- Putting module source code in the same directory as root `.tf` files without a `module` block — Terraform will try to load it as part of the root config.
- Committing `terraform.tfvars` with secrets (AWS keys, database passwords).
- Storing state locally on a laptop for team projects — leads to conflicts and data loss.
- Assuming `terraform.tfvars` is loaded in all contexts — it is auto-loaded only in the working directory; CI may need `-var-file` explicitly.

## Related Notes

- [[02_Terraform_Workflow]] — commands that operate on the working directory
- [[04_Terraform_Configuration]] — provider and backend configuration files
- [[01_Fundamentals#Core Components]] — state and modules overview

## To Be Defined

- Multi-environment directory layouts (`envs/dev`, `envs/prod` vs. workspace-based)
- Monorepo vs. polyrepo strategies for Terraform at scale
- `backend.hcl` partial configuration pattern
