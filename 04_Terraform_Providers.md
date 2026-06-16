## Provider Block

This block is used to extend Terraform's functionality. It allows Terraform to connect to any API-driven platform — mainly cloud platforms, SaaS applications, or on-premises hardware solutions. A provider defines where and what Terraform can manage, such as AWS for cloud resources or GitHub for repositories.

A provider is a plugin that extends the functionality of Terraform Core and enables it to interact with external APIs.

- Resources are implemented by a provider. Without them, Terraform would not be able to provision or manage any infrastructure.
- Providers are maintained by HashiCorp, partners, and the Terraform community.
- Providers are developed separately from Terraform Core, so functionality can be added or changed per version.

Example of a provider configuration block:

```hcl
provider "aws" {
  region  = "us-east-1"
  profile = "example-profile"
}
```

Some providers require authentication arguments, but it is better to define credentials via environment variables or shared credentials files to avoid security leaks. A single provider block enables Terraform to manage multiple resources on a platform.

While providers are stored on GitHub, full documentation can be found on the [Terraform Registry](https://registry.terraform.io/).

When declared, the provider is downloaded during `terraform init` and cached in `.terraform/providers/`.

## Required Providers Block

Declare which providers the configuration needs inside the top-level `terraform` block. This is separate from the provider _configuration_ block.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

| Concept | Block | Purpose |
|---|---|---|
| Provider requirement | `terraform { required_providers {} }` | Declares provider source and version constraints |
| Provider configuration | `provider "aws" {}` | Sets region, credentials, default tags, etc. |

> **Exam tip:** `required_providers` goes in the `terraform` block. `provider "aws"` is a separate block. Confusing these two is a common exam trap.

### Version Constraint Operators

```hcl
version = "5.31.0"       # exact version
version = ">= 5.0"       # minimum version
version = "~> 5.0"       # >= 5.0, < 6.0 ( pessimistic constraint — most common )
version = ">= 4.0, < 6.0" # compound constraint
```

> **Real-world:** Pin major versions with `~>` in `required_providers`. Let `.terraform.lock.hcl` pin exact patch versions for reproducibility across teammates and CI.

## Multiple Provider Instances (Aliases)

Use `alias` when you need two configurations of the same provider — for example, multi-region AWS or multi-account setups.

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

resource "aws_s3_bucket" "east" {
  bucket = "logs-east"
}

resource "aws_s3_bucket" "west" {
  provider = aws.west
  bucket   = "logs-west"
}
```

> **Exam tip:** The default provider has no alias. Use the `provider` meta-argument with an alias reference (e.g., `provider = aws.west`). Omit it to use the default provider configuration.

## Authentication Best Practices

Avoid hardcoding credentials in provider blocks. Preferred methods for AWS:

| Method | Example | When to Use |
|---|---|---|
| Environment variables | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN` | CI/CD, temporary credentials |
| Shared credentials file | `~/.aws/credentials` + `profile` argument | Local development |
| IAM role | EC2 instance profile, ECS task role | Workloads running on AWS |
| OIDC / IAM Identity Center | GitHub Actions → AWS via OIDC | Modern CI/CD without long-lived keys |

```hcl
provider "aws" {
  region  = var.aws_region
  profile = var.aws_profile   # optional — uses default profile if omitted
}
```

> **Common mistake:** Committing `access_key` and `secret_key` in a provider block. This fails security reviews and Terraform Associate exam scenarios.

> **AWS SAA-C03 tie-in:** Prefer IAM roles over access keys. Use least-privilege IAM policies scoped to the Terraform automation role. Separate roles per environment (dev/staging/prod).

## Provider Meta-Argument

Resources inherit the default provider unless you specify otherwise:

```hcl
resource "aws_instance" "example" {
  provider = aws.west
  # ...
}
```

## Implicit vs. Explicit Provider Dependencies

- Provider configuration is evaluated before resources.
- If a resource depends on another resource in a different region/account, use the correct `provider` meta-argument — not `depends_on` on the provider.

## The Terraform Registry

- **Providers** — `registry.terraform.io/providers/hashicorp/aws`
- **Modules** — reusable configurations published by HashiCorp and the community
- **Policy Libraries** — Sentinel policies (Terraform Enterprise / HCP Terraform)

During `init`, Terraform downloads providers from the registry (or a configured mirror) based on `required_providers` source addresses.

## Common Mistakes

- Declaring `provider "aws"` but forgetting `required_providers` — works in older configs but fails provider source validation in Terraform 0.13+.
- Using the wrong provider alias and wondering why resources land in the wrong region.
- Running `init -upgrade` in CI without reviewing provider changelogs — can introduce breaking changes.
- Assuming `validate` downloads or verifies providers — only `init` does.

## Provider Configuration in Modules

Child modules do **not** automatically inherit `provider` blocks from the root module. The root module configures providers; child modules **receive** them through implicit inheritance (default provider) or explicit `providers` mapping on the `module` block.

### Default Inheritance

If a child module uses `provider "aws"` without an alias, it inherits the root module's default AWS provider configuration automatically. No extra mapping is required.

```hcl
# Root module — providers.tf
provider "aws" {
  region = "us-east-1"
}

# Root module — main.tf
module "network" {
  source = "./modules/network"
  # child module uses root's default aws provider
}
```

### Explicit Provider Mapping

When a module uses **aliased** providers, the root module must pass them in using the `providers` meta-argument:

```hcl
# Child module — expects aliased provider (see configuration_aliases below)
resource "aws_s3_bucket" "replica" {
  provider = aws.replica
  bucket   = var.bucket_name
}

# Root module — pass aliased provider into module
module "replication" {
  source = "./modules/replication"

  providers = {
    aws.replica = aws.west   # map module's aws.replica to root's aws.west
  }

  bucket_name = "logs-west"
}
```

> **Exam tip:** The `providers` map keys use the **module's** provider configuration names (including alias). Values reference **root module** provider configurations.

> **Common mistake:** Defining `provider "aws"` blocks inside a reusable module without declaring `configuration_aliases`. Terraform will reject passing aliased providers from the root.

## `configuration_aliases` (Module Authors)

Reusable modules that need non-default providers must declare which provider aliases they accept inside a `terraform` block:

```hcl
# modules/replication/versions.tf
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = "~> 5.0"
      configuration_aliases = [aws.replica]
    }
  }
}
```

- Tells Terraform the module can accept an `aws` provider with alias `replica`.
- The module itself does **not** configure credentials or region — the root module does that and passes the configured provider in.
- Keeps modules portable: the same module can target different accounts or regions depending on how the caller maps providers.

> **Real-world:** Module authors should never hardcode `region` or credentials. Accept providers from the caller and document required `providers` mappings in the module README.

## AWS Default Tags (`default_tags`)

Apply tags to all AWS resources managed by a provider without repeating `tags` on every resource block:

```hcl
provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "Terraform"
      Project     = var.project_name
    }
  }
}
```

- Tags set here are merged with resource-level `tags`. Resource-level tags override `default_tags` on key collision.
- Applies only to resources created through that provider configuration — useful for multi-account setups with separate provider blocks.
- Reduces tag drift and satisfies organizational tagging policies (cost allocation, ownership, compliance).

> **AWS SAA-C03 tie-in:** Consistent tagging supports Cost Explorer, Resource Groups, and SCP enforcement. Centralizing tags in the provider block is cleaner than tagging every resource individually.

> **Exam tip:** `default_tags` is a provider-level setting, not a resource meta-argument. It does not replace resource-specific tags you need for unique identification (e.g., `Name` per instance).

## Private Provider Registries

Organizations using **Terraform Enterprise** or **HCP Terraform** can host private providers instead of relying on the public Terraform Registry.

| Feature | Public Registry | Private Registry |
|---|---|---|
| Access | Open to all | Restricted to organization |
| Use case | Community and HashiCorp providers | Internal providers, air-gapped environments |
| Source address | `hashicorp/aws` | `<hostname>/<namespace>/<name>` |

Example with a private registry:

```hcl
terraform {
  required_providers {
    mycloud = {
      source  = "app.terraform.io/my-org/mycloud"
      version = "~> 1.0"
    }
  }
}
```

- `terraform login` authenticates to HCP Terraform / Terraform Enterprise and enables downloading private providers during `init`.
- Private registries also host **private modules** and **policy libraries** (Sentinel) on enterprise tiers.
- Provider mirrors can cache public providers internally for security and compliance without publishing custom providers.

> **Real-world:** Regulated industries use private registries to vet provider binaries before they reach production CI pipelines. Pair with network egress controls and signed provider verification.

> **Exam tip:** Know that `required_providers` `source` uses the format `hostname/namespace/type` — not just `hashicorp/aws`. Terraform 0.13+ requires an explicit source for all providers.

## Related Notes

- [[05_Terraform_Configuration_Blocks]] — `resource`, `data`, `variable`, `output`, and `terraform` blocks
- [[02_Terraform_Workflow#The `terraform init` command]] — when providers are downloaded
- [[03_Terraform_File_Structure#Common Terraform Files]] — where to place provider and version blocks
- [[01_Fundamentals#Core Components]] — providers as a core component
- [[00_Index]] — study progress
