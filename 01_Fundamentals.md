## **What is Terraform?**

- Tool created by HashiCorp for Infrastructure as Code (IaC). Known as the industry standard for IaC.
- Automates the provisioning, management, and versioning of infrastructure.
- It is popular for its _Declarative Approach_. In other words, it focuses on describing what should be done rather than explicitly defining how to do it (like Ansible). This allows Terraform to determine the best execution steps automatically.
- Based on HCL (HashiCorp Configuration Language), a human-readable language used in Terraform to define infrastructure resources.

> **Exam tip:** Terraform is _declarative_ and _idempotent_. You describe the desired end state; Terraform figures out create, update, or destroy actions. Contrast with _imperative_ tools (AWS CLI scripts, CloudFormation templates with rigid ordering) where you define steps explicitly.

> **Real-world:** Teams choose Terraform when they need multi-cloud support, a large provider ecosystem, and reusable modules. AWS-only shops sometimes use CloudFormation or CDK; configuration management tools like Ansible complement Terraform rather than replace it.

## **Benefits of Terraform (IaC)**

- Codifies infrastructure rather than manually configuring resources through a console.
- Standardizes workflows by using a single tool to deploy and manage resources required for application workloads.
- Platform-agnostic. Can deploy infrastructure to multiple cloud providers and datacenters.
- Version Control and Auditability. Infrastructure changes can be tracked using version control systems like Git, allowing history tracking and rollback capabilities.
- Automation and Efficiency. Reduces manual tasks and accelerates deployments.
- Scalability and Flexibility. Infrastructure can scale up or down easily by modifying code instead of performing manual actions.
- Collaboration. Multiple team members can work on the same infrastructure configuration.
- Consistency and Repeatability. Ensures infrastructure is deployed consistently across environments.

## **Fundamentals of HCL**

- Declarative language designed to be easy to read and write.
- Used to define infrastructure resources and configurations.

### **Basic Usage**

This is the basic structure of Terraform code:

```hcl
# single-line comment
block_type "block_label" "block_label" {
  first_argument  = expression or value
  second_argument = expression or value
  third_argument  = expression or value
}

attribute_1 = value
attribute_2 = value
```

- Blocks are delimited by `{}` and all contents within the block represent the properties of that block.
- Blocks start with a keyword that defines the block type, such as `resource`, `data`, or `variable`.
- Blocks contain named values, allowing Terraform to access values created or obtained by a block.

### **Real Example**

```hcl
data "aws_availability_zones" "available" {}

resource "aws_vpc" "vpc" {
  cidr_block = var.vpc_cidr

  tags = {
    Name        = var.vpc_name
    Environment = "demo_environment"
    Terraform   = "true"
  }
}
```

### **Style Guidelines**

- Use `#` for single-line comments.
- Use underscores (`_`) to separate multiple words.
- Indent two spaces for each nesting level.
- Align equal signs for values on consecutive lines when possible.
- Use empty lines to separate logical groups of arguments within a block.

## **Resource Referencing**

Resource referencing allows resources within a configuration to share data and build dependencies between them.

- Creates more dynamic configurations by avoiding hardcoded values.
- Enables automatic dependency mapping. Terraform automatically determines the order of resource creation based on references.
- Resource identifiers. Each block is identified by a type and a name, making it easy to reference throughout the configuration.

Reference syntax: `<BLOCK_TYPE>.<NAME>.<ATTRIBUTE>`

```hcl
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.vpc.id          # implicit dependency on aws_vpc.vpc
  cidr_block = "10.0.1.0/24"
  availability_zone = data.aws_availability_zones.available.names[0]
}
```

> **Common mistake:** Hardcoding IDs (e.g., `vpc_id = "vpc-0abc123"`) breaks portability across environments and bypasses Terraform's dependency graph. Always reference other blocks when possible.

> **Exam tip:** References create _implicit dependencies_. Use `depends_on` only when Terraform cannot infer the dependency from attribute references alone (see [[02_Terraform_Workflow#Dependencies]]).

## **Best Practices for HCL**

- Use the file extensions `.tf` or `.tfvars` to allow tools and Terraform to recognize and apply configurations correctly.
- Always format files correctly. Use the command `terraform fmt`.
- Organize functionality within files. For example: variables in `variables.tf`, outputs in `outputs.tf`, etc. (see [[03_Terraform_File_Structure]]).
- Comment and document the code when necessary.
- Avoid hardcoding values whenever possible. Use `variable` blocks and `.tfvars` files per environment.
- Use linters, extensions, and additional tools to improve code quality and consistency.

## **Core Components**

- **Terraform Core**. CLI tool responsible for provisioning and managing infrastructure resources defined in configuration files. Parses HCL, builds the dependency graph, compares desired vs. actual state, and orchestrates changes.
- **Providers**. Extend Terraform functionality for specific platforms such as cloud providers or SaaS services. See [[04_Terraform_Providers]].
- **Resources**. Infrastructure components or services managed by Terraform (e.g., `aws_instance`, `aws_s3_bucket`).
- **State**. Mechanism Terraform uses to map desired configurations to real-world resources on the target platform. See [[03_Terraform_File_Structure#Additional Files]].
- **Modules**. Reusable and shareable blocks of code that can be called multiple times.

> **Exam tip:** Know the difference between **configuration** (`.tf` files — desired state), **state** (mapping to real resources), and **plan** (proposed diff). Terraform Core never talks to APIs directly; providers do.

## **Terraform Editions**

| Edition                         | Description                                                                                                                            | Typical Use                                         |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| **Terraform Community Edition** | Free CLI (`terraform` binary). Local or remote state.                                                                                  | Individual developers, small teams, CI/CD pipelines |
| **HCP Terraform**               | HashiCorp Cloud Platform SaaS (formerly Terraform Cloud). Remote state, run history, collaboration, policy (Sentinel on higher tiers). | Teams needing shared state, RBAC, and run workflows |
| **Terraform Enterprise**        | Self-hosted version of HCP Terraform. Same features, runs in your datacenter.                                                          | Regulated industries, air-gapped environments       |

> **Exam tip:** "Terraform Cloud" was rebranded to **HCP Terraform**. Exam questions may use either name. Community Edition has no built-in remote state or Sentinel — you configure a remote backend (e.g., S3 + DynamoDB) yourself.

> **Real-world:** Most production teams use either HCP Terraform or Community Edition with an S3 backend and DynamoDB locking table. Enterprise is common in finance and government where SaaS is not permitted.

## **Common Mistakes**

- Treating Terraform like a scripting language — running `apply` repeatedly without reviewing the plan.
- Committing `terraform.tfstate` or secrets in `.tfvars` to Git.
- Ignoring `.terraform.lock.hcl` in version control (provider version drift across teammates).
- Confusing **Terraform CLI workspaces** (`terraform workspace`) with **HCP Terraform workspaces** (separate state namespaces in the SaaS product).
