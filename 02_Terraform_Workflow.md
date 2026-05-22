## Official workflow

1. Write code
2. Plan
3. Apply

---
## Real workflow

1. Write configuration (in HCL). Involves the creation of one or more configuration files that define your desired infrastructure or resources (VMs, Networks, etc. ).
2. Initialize
3. Plan
4. Apply
5. Destroy (Optional)

---
## The `terraform init` command

It prepares the environment to run other Terraform commands. It downloads the required providers and modules used in the configuration.

`terraform init` configures the backend for storing Terraform state and creates the log file

It is common to run it at the start of the project or any time you add, update or remove the providers or backend settings of your configuration. The providers are usually defined on the `providers.tf` file.

### The terraform Lock File

When you run `terraform init`, terraform creates or updates a file called `.terraform.lock.hcl` in the working directory. This file locks the dependency versions for consistency across the environments. It is a best practices to upload to your version control system.

### The `.terraform` folder

Terraform uses the `.terraform` directory to store critical data for the project. This means providers and modules used in the project.

- The `.terraform/providers` directory, stores cached version of the configurations providers.
- The `.terraform/modules` directory contains downloaded versions of the referenced modules.

Best practice is to ignore it in version control. Sometimes you will need to delete the directory to recreate or clean your working directory.

---
## The `terraform plan` command

Show what terraform will exactly do before doing. Generates an execution plan showing the changes that terraform will make to the resources to match the desired state.  Provides an output of the change that will happen if changes are applied.

It shows the resources to:
1. Create
2. Update
3. Destroy

Is basically a dry-run without impacting "real world" resources. It is crucial for reviewing changes in production envs to prevent incidents and is hepfull to run before any changes but not required for `terraform apply`command.

### Saving a Plan

It helps to guarantee execution for review and approval. It has some benefits like the exact actions in the saved plan will be executed with no surprises. 

You can save a plan as follows:

``` sh
terraform plan -out=myplan.tfplan
terraform apply myplan.tfplan
```

### Resource Graph

Terraform builds a dependency graph from the configuration to determine the order in which to create, update or destroy resources. The resource order in the .tf diles doesn't matter, resource graph is how terraform achieves parallel execution. Is the foundation to understand resource dependencies.

### Dependencies

- **Implicit Dependencies** : Automatic reference to another resource that terraform already knows.
- **Explicit Dependencies** : Used when there is a dependency that terraform cannot automatically detect from resource references alone. You can  use `depends_on` to create an explicit dependency.

### Parallel Resource Operations

Terraform process the resource Graph with parallel execution. It processes nodes as soon as their dependencies are satisfied. Non-dependent resources are created/modified simultaneously.

By default, Terraform manages 10 operations in parallel but can be modified using the `-parallelism` flag:

```sh
terraform apply -parallelism=5
```

Usually change parallelism is not needed, it is used when you are running into API rate limiting issues or you are debugging/troubleshooting. Most users never need to change this setting.

---
## The `terraform apply` command

It tells Terraform to provision the real infrastructure. It takes the defined configuration and creates the resources as defined. This is how Terraform makes the changes to your infrastructure.

`terraform apply` runs a plan first to compare the existing infrastructure to the desired state. By default,  terraform will prompt for a confirmation before proceed  with the changes.

You must use it anytime you want to make changes to the infrastructure. This includes adding, modifying or destroying real-world resources.

### Apply Operations

- **State Locking** - Terraform locks the state file during apply to prevent concurrent modifications.
- **State updates** - State file is updated after each resource is successfully created/modified/destroyed.
- **No automatic rollback** - If a resource fails during apply, Terraform doesn't rollback previous successfully changes. They remain in place.
- Respects Dependencies - Commit to version control and Managed Automatically by Terraform.

### Types of apply

**Standard Appy** 

```sh
$ terraform apply
Confirm (yes/no):
```

**Apply a Saved Plan**

```sh
$ terraform apply myplan.tfplan
```

**Apply & Auto Approve**

```sh
terraform apply -auto-approve
(skips confirmation prompt - use carfully)
```

`-input=false` is also helpful for automation or CI/CD pipelines

### Best Practices

**Before Apply**
- Always review the plan
- Ensure you are in the right path
- Test in Non-Production

**During Apply**
- Read the plan carefully
- Watch for Unexpected Changes
- Be patient with Large Applies

**After apply**
- Verify the changes
- Commit your configuration
- Document Significant Changes

---
## The `terraform destroy` command

The `terraform destroy` command terminates all resources managed by Terraform in the current workspace or working directory.

- Explicitly destroys all infrastructure being managed by Terraform. This operation will show the destroy plan first and then prompt for confirmation before proceeding.
- It is a better practice to remove individual resources from the configuration. Remove a resource block from a .tf file and then run the `terraform apply` will detect the resource and delete it.
- Once a resource is destroyed, Terraform removes it from state file since it is not longer being managed.

### Destroying resources

The default command `terraform destroy` will destroy all resources managed by Terraform. Also, it will respect the dependencies in reverse order.

You can also target a specific resource, if needed, using the destroy command as follows.

```sh
terraform destroy-target=<resource_name>
```

Take in count that if the resource is not deleted from the configuration, next apply command will try to create it again.

---

## The  `terraform validate` command

This command checks your terraform configuration files for syntax errors and ensures they are structurally valid. It does not check whether the configuration will successfully deploy but ensures the code is properly written and references resources and variables correctly.

`terrafor validate` is an essential step to catch errors early before running commands like plan or apply.

It is important to mention that it will not communicate with providers, because of that it will not catch every error.

Before executing the command, you must initialize your working directory first.

