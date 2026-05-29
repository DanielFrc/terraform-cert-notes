## Official workflow

HashiCorp defines three core steps:

1. Write code
2. Plan
3. Apply

## Real workflow

In practice, engineers follow a longer sequence:

1. Write configuration (in HCL). Involves the creation of one or more configuration files that define your desired infrastructure or resources (VMs, networks, etc.).
2. **Initialize** — `terraform init`
3. **Validate** — `terraform validate` (optional but recommended)
4. **Format** — `terraform fmt` (optional but recommended)
5. **Plan** — `terraform plan`
6. **Apply** — `terraform apply`
7. **Destroy** (optional) — `terraform destroy`

> **Exam tip:** `terraform init` is required before `validate`, `plan`, or `apply`. `validate` does not contact providers. `plan` and `apply` do.

## The `terraform init` command

It prepares the environment to run other Terraform commands. It downloads the required providers and modules used in the configuration.

`terraform init` configures the backend for storing Terraform state and initializes the working directory (including the `.terraform/` cache).

It is common to run it at the start of the project or any time you add, update, or remove providers, modules, or backend settings. Providers are usually defined in `providers.tf` or a `terraform` block (see [[04_Terraform_Configuration]]).

### Reconfigure and upgrade flags

```sh
terraform init -reconfigure   # reinitialize backend without migrating state
terraform init -upgrade       # upgrade providers to latest allowed by constraints
```

> **Common mistake:** Running `plan` or `apply` before `init` produces "Module not installed" or "Provider not initialized" errors.

### The Terraform Lock File

When you run `terraform init`, Terraform creates or updates a file called `.terraform.lock.hcl` in the working directory. This file locks provider dependency versions for consistency across environments. It is a best practice to commit this file to version control.

> **Exam tip:** `.terraform.lock.hcl` locks **provider** versions. It is not the state lock (that is handled by the backend during apply).

### The `.terraform` folder

Terraform uses the `.terraform` directory to store critical data for the project — downloaded providers and modules.

- The `.terraform/providers` directory stores cached versions of the configuration's providers.
- The `.terraform/modules` directory contains downloaded versions of the referenced modules.

Best practice is to add `.terraform/` to `.gitignore`. Sometimes you will need to delete the directory and re-run `init` to clean or recreate your working directory.

## The `terraform validate` command

This command checks your Terraform configuration files for syntax errors and ensures they are structurally valid. It does not check whether the configuration will successfully deploy but ensures the code is properly written and references resources and variables correctly.

`terraform validate` is an essential step to catch errors early before running commands like plan or apply.

It will not communicate with providers, so it will not catch every runtime error (e.g., invalid AMI IDs or permission issues).

Before executing the command, you must initialize your working directory first.

## The `terraform fmt` command

Automatically rewrites configuration files to follow canonical HCL formatting. Run before committing code.

```sh
terraform fmt          # format files in current directory
terraform fmt -check   # exit non-zero if formatting is needed (useful in CI)
terraform fmt -recursive
```

> **Real-world:** Most teams enforce `terraform fmt -check` in CI pipelines alongside `terraform validate`.

## The `terraform plan` command

Shows what Terraform will do before making changes. Generates an execution plan showing the changes Terraform will make to resources to match the desired state. Provides an output of the changes that will happen if changes are applied.

It shows resources to:

1. Create (`+`)
2. Update (`~`)
3. Destroy (`-`)
4. Replace (`-/+` — forces destroy and recreate)

It is basically a dry-run without impacting real-world resources. It is crucial for reviewing changes in production environments to prevent incidents and is helpful to run before any changes, but it is not required for `terraform apply` (apply runs its own plan by default).

### Saving a Plan

Saving a plan helps guarantee execution matches review and approval. The exact actions in the saved plan will be executed with no surprises — the plan file is tied to a specific state snapshot.

```sh
terraform plan -out=myplan.tfplan
terraform apply myplan.tfplan
```

> **Exam tip:** A saved plan cannot be applied after the state has changed in a way that invalidates it. This is intentional — it prevents approving one plan and applying another.

> **Real-world:** CI/CD pipelines often run `plan` in a pull request and `apply` the saved plan artifact after merge approval.

### Resource Graph

Terraform builds a dependency graph from the configuration to determine the order in which to create, update, or destroy resources. The resource order in the `.tf` files does not matter; the resource graph is how Terraform achieves parallel execution. It is the foundation for understanding resource dependencies.

### Dependencies

- **Implicit Dependencies** — Created automatically when one resource references another's attributes (e.g., `aws_subnet.public.vpc_id = aws_vpc.main.id`).
- **Explicit Dependencies** — Used when there is a dependency Terraform cannot automatically detect from resource references alone. Use `depends_on` to declare it.

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  depends_on = [aws_iam_role_policy_attachment.ssm]
}
```

> **Common mistake:** Overusing `depends_on`. Prefer attribute references whenever possible — they are more precise and self-documenting.

> **Exam tip:** `depends_on` accepts a list of resources or modules. It does not accept arbitrary strings or provider names.

### Parallel Resource Operations

Terraform processes the resource graph with parallel execution. It processes nodes as soon as their dependencies are satisfied. Non-dependent resources are created or modified simultaneously.

By default, Terraform manages 10 operations in parallel but can be modified using the `-parallelism` flag:

```sh
terraform apply -parallelism=5
```

Usually changing parallelism is not needed. It is used when you hit API rate limiting or when debugging/troubleshooting. Most users never need to change this setting.

## The `terraform apply` command

It tells Terraform to provision the real infrastructure. It takes the defined configuration and creates the resources as defined. This is how Terraform makes changes to your infrastructure.

`terraform apply` runs a plan first to compare the existing infrastructure to the desired state. By default, Terraform will prompt for confirmation before proceeding with the changes.

You must use it anytime you want to make changes to the infrastructure. This includes adding, modifying, or destroying real-world resources.

### Apply Operations

- **State Locking** — Terraform locks the state file during apply to prevent concurrent modifications (requires a remote backend with locking support, e.g., S3 + DynamoDB).
- **State Updates** — State file is updated after each resource is successfully created, modified, or destroyed.
- **No Automatic Rollback** — If a resource fails during apply, Terraform does not roll back previously successful changes. They remain in place. You must fix the error and re-apply, or manually reconcile.
- **Respects Dependencies** — Apply follows the dependency graph, same as plan.

> **Real-world:** A partial apply failure is one of the most common production incidents. Always review the plan, use saved plans for production, and have a rollback strategy (often a Git revert + re-apply, not an automatic undo).

### Types of Apply

**Standard Apply**

```sh
$ terraform apply
Confirm (yes/no):
```

**Apply a Saved Plan**

```sh
$ terraform apply myplan.tfplan
```

**Apply with Auto Approve**

```sh
terraform apply -auto-approve
# skips confirmation prompt — use carefully
```

`-input=false` is also helpful for automation or CI/CD pipelines (disables interactive variable prompts).

### Best Practices

**Before Apply**

- Always review the plan
- Ensure you are in the right directory and workspace
- Test in non-production first

**During Apply**

- Read the plan carefully
- Watch for unexpected changes (especially destroys)
- Be patient with large applies

**After Apply**

- Verify the changes in the provider console or with `terraform show`
- Commit your configuration (not state, unless using a local-only workflow)
- Document significant changes

## The `terraform destroy` command

The `terraform destroy` command terminates all resources managed by Terraform in the current workspace or working directory.

- Explicitly destroys all infrastructure being managed by Terraform. This operation will show the destroy plan first and then prompt for confirmation before proceeding.
- It is better practice to remove individual resources from the configuration. Remove a resource block from a `.tf` file and then run `terraform apply` — Terraform will detect the resource is no longer desired and delete it.
- Once a resource is destroyed, Terraform removes it from the state file since it is no longer being managed.

> **Exam tip:** Prefer removing resources from configuration + `apply` over `destroy` for production. `destroy` removes _everything_ in state for that workspace.

### Destroying Resources

The default command `terraform destroy` will destroy all resources managed by Terraform. It respects dependencies in reverse order.

You can also target a specific resource using:

```sh
terraform destroy -target=aws_instance.example
```

Take into account that if the resource is not removed from the configuration, the next `apply` will try to create it again.

> **Common mistake:** Using `-target` in production as a regular workflow. It bypasses the full dependency graph and can leave infrastructure in an inconsistent state. Reserve it for emergencies and debugging.

## Related Notes

- [[03_Terraform_File_Structure]] — where configuration and state files live
- [[04_Terraform_Configuration]] — provider setup required before init
- [[00_Index]] — study progress and weak areas
