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
## The `terraform apply` command




