## **What is Terraform?**

- Tool created by HashiCorp for Infrastructure as Code (IaC). Known as the industry standard for IaC.
- Automates the provisioning, management, and versioning of infrastructure.
- It is popular for its _Declarative Approach_. In other words, it focuses on describing what should be done rather than explicitly defining how to do it (like Ansible). This allows Terraform to determine the best execution steps automatically.
- Based on HCL (HashiCorp Configuration Language), a human-readable language used in Terraform to define infrastructure resources.

---

## **Benefits of Terraform (IaC)**

- Codifies infrastructure rather than manually configuring resources through a console.
- Standardizes workflows by using a single tool to deploy and manage resources required for application workloads.
- Platform-agnostic. Can deploy infrastructure to multiple cloud providers and datacenters.
- Version Control and Auditability. Infrastructure changes can be tracked using version control systems like Git, allowing history tracking and rollback capabilities.
- Automation and Efficiency. Reduces manual tasks and accelerates deployments.
- Scalability and Flexibility. Infrastructure can scale up or down easily by modifying code instead of performing manual actions.
- Collaboration. Multiple team members can work on the same infrastructure configuration.
- Consistency and Repeatability. Ensures infrastructure is deployed consistently across environments.

---

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

---
## **Resource Referencing**

Resource referencing allows resources within a configuration to share data and build dependencies between them.

- Creates more dynamic configurations by avoiding hardcoded values.
- Enables automatic dependency mapping. Terraform automatically determines the order of resource creation based on references.
- Resource identifiers. Each block is identified by a type and a name, making it easy to reference throughout the configuration.

---
## **Best Practices for HCL**

- Use the file extensions `.tf` or `.tfvars` to allow tools and Terraform to recognize and apply configurations correctly.
- Always format files correctly. Use the command `terraform fmt`.
- Organize functionality within files. For example: variables in `variables.tf`, outputs in `outputs.tf`, etc.
- Comment and document the code when necessary.
- Avoid hardcoding values whenever possible.
- Use linters, extensions, and additional tools to improve code quality and consistency.

---
## **Core Components**

- **Terraform Core**. CLI tool responsible for provisioning and managing infrastructure resources defined in configuration files.
- **Providers**. Extend Terraform functionality for specific platforms such as cloud providers or SaaS services.
- **Resources**. Infrastructure components or services managed by Terraform.
- **State**. Mechanism Terraform uses to map desired configurations to real-world resources on the target platform.
- **Modules**. Reusable and shareable blocks of code that can be called multiple times.

---
## **Terraform Editions**

- **Terraform Community Edition**. Basic command-line tool.
- **HCP Terraform**. SaaS platform provided by HashiCorp.
- **Terraform Enterprise**. Self-hosted and enterprise-managed version of Terraform.