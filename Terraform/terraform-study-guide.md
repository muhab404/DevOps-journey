# Terraform Study Guide - DevOps & Cloud Engineering

## 1. Introduction & Basics

**What is Terraform?**
Terraform is an open-source Infrastructure as Code (IaC) tool developed by HashiCorp. It allows you to define and provision infrastructure using declarative configuration files.

**Why is Terraform used?**
- **Infrastructure as Code**: Version control your infrastructure
- **Multi-cloud support**: Works with AWS, Azure, GCP, and 100+ providers
- **Declarative approach**: Define desired state, Terraform handles the how
- **Plan before apply**: Preview changes before execution
- **State management**: Tracks infrastructure state for consistency
- **Reusability**: Modules enable code reuse across projects

## 2. Core Concepts

**Key Terminology:**
- **Provider**: Plugin that interacts with APIs (AWS, Azure, etc.)
- **Resource**: Infrastructure component (EC2 instance, S3 bucket)
- **Data Source**: Read-only information from existing infrastructure
- **Module**: Reusable collection of resources
- **State**: Current state of managed infrastructure
- **Plan**: Execution plan showing what will change
- **Backend**: Where state files are stored

**Architecture:**
```
Terraform Configuration (.tf files)
         ↓
    Terraform Core
         ↓
    Provider Plugins
         ↓
    Target Infrastructure
```

**Workflow:**
1. **Write** - Author infrastructure as code
2. **Plan** - Preview changes
3. **Apply** - Provision infrastructure
4. **Destroy** - Clean up resources

## 3. Installation/Setup

**Installation:**
```bash
# Windows (using Chocolatey)
choco install terraform

# macOS (using Homebrew)
brew install terraform

# Linux (manual)
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
unzip terraform_1.6.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/
```

**Verify Installation:**
```bash
terraform version
```

**Basic Setup:**
```bash
# Initialize working directory
terraform init

# Validate configuration
terraform validate

# Format code
terraform fmt

# Plan changes
terraform plan

# Apply changes
terraform apply

# Destroy infrastructure
terraform destroy
```

## 4. Hands-on Examples

**Basic AWS EC2 Instance:**
```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-west-2"
}

resource "aws_instance" "web" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t2.micro"
  
  tags = {
    Name = "HelloWorld"
  }
}
```

**Variables Example:**
```hcl
# variables.tf
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-west-2"
}

# main.tf
resource "aws_instance" "web" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = var.instance_type
  
  tags = {
    Name = "WebServer"
  }
}
```

**Outputs Example:**
```hcl
# outputs.tf
output "instance_ip" {
  description = "Public IP of the instance"
  value       = aws_instance.web.public_ip
}

output "instance_id" {
  description = "ID of the instance"
  value       = aws_instance.web.id
}
```

**Module Example:**
```hcl
# modules/ec2/main.tf
resource "aws_instance" "this" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  tags = var.tags
}

# modules/ec2/variables.tf
variable "ami_id" {
  type = string
}

variable "instance_type" {
  type = string
}

variable "tags" {
  type = map(string)
}

# Root main.tf
module "web_server" {
  source = "./modules/ec2"
  
  ami_id        = "ami-0c02fb55956c7d316"
  instance_type = "t2.micro"
  tags = {
    Name = "WebServer"
    Env  = "Production"
  }
}
```

## 5. Best Practices

**Security:**
- Store sensitive data in variables, not hardcoded
- Use remote state with encryption
- Implement least privilege access
- Use terraform.tfvars.example for documentation
- Never commit .tfvars files with secrets

**Scalability:**
- Use modules for reusability
- Implement proper naming conventions
- Use workspaces for environment separation
- Implement resource tagging strategy

**Maintainability:**
- Use consistent file structure
- Document your code with comments
- Use terraform fmt for formatting
- Implement CI/CD pipelines
- Use semantic versioning for modules

**File Structure:**
```
project/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars.example
├── versions.tf
└── modules/
    └── vpc/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

## 6. Common Issues & Troubleshooting

**State File Issues:**
- **Problem**: State file corruption or conflicts
- **Solution**: Use remote state, enable state locking, backup state files

**Provider Version Conflicts:**
- **Problem**: Different provider versions causing issues
- **Solution**: Pin provider versions in versions.tf

**Resource Dependencies:**
- **Problem**: Resources created in wrong order
- **Solution**: Use depends_on or implicit dependencies

**Authentication Issues:**
- **Problem**: Provider authentication failures
- **Solution**: Configure credentials properly (AWS CLI, environment variables)

**Plan vs Apply Differences:**
- **Problem**: Plan shows different results than apply
- **Solution**: Refresh state, check for external changes

**Common Commands for Troubleshooting:**
```bash
terraform refresh
terraform state list
terraform state show <resource>
terraform import <resource> <id>
terraform taint <resource>
```

## 7. Interview Q&A

**Q1: What is Terraform state and why is it important?**
A: Terraform state is a file that maps real-world resources to your configuration. It's crucial for tracking resource metadata, performance optimization, and determining what changes need to be made during planning.

**Q2: Explain the difference between terraform plan and terraform apply.**
A: `terraform plan` creates an execution plan showing what actions Terraform will take without making changes. `terraform apply` executes the plan and makes actual changes to infrastructure.

**Q3: What are Terraform modules and their benefits?**
A: Modules are containers for multiple resources used together. Benefits include code reusability, organization, encapsulation, and easier testing and maintenance.

**Q4: How do you handle sensitive data in Terraform?**
A: Use variables with sensitive = true, store secrets in external systems (AWS Secrets Manager, HashiCorp Vault), use terraform.tfvars files (not committed), and enable remote state encryption.

**Q5: What is remote state and why use it?**
A: Remote state stores the state file in a remote location (S3, Azure Storage). Benefits include team collaboration, state locking, encryption, and backup capabilities.

**Q6: Explain Terraform workspaces.**
A: Workspaces allow you to manage multiple environments (dev, staging, prod) with the same configuration by maintaining separate state files for each workspace.

**Q7: What happens if you lose your Terraform state file?**
A: You lose track of managed resources. Solutions include restoring from backup, using terraform import to recreate state, or recreating resources from scratch.

**Q8: How do you handle Terraform version upgrades?**
A: Pin Terraform and provider versions, test in non-production environments, use terraform init -upgrade, and follow upgrade guides for breaking changes.

**Q9: What are data sources in Terraform?**
A: Data sources allow Terraform to fetch information from existing infrastructure or external systems without managing those resources directly.

**Q10: How do you implement CI/CD with Terraform?**
A: Use version control, implement automated testing, use terraform plan in PR reviews, automate terraform apply for approved changes, and implement proper secret management.

## 8. Comparison with Alternatives

**Terraform vs CloudFormation:**
- **Terraform**: Multi-cloud, larger community, HCL syntax
- **CloudFormation**: AWS-native, JSON/YAML, integrated with AWS services

**Terraform vs Ansible:**
- **Terraform**: Infrastructure provisioning, declarative, state management
- **Ansible**: Configuration management, procedural, agentless

**Terraform vs Pulumi:**
- **Terraform**: HCL language, mature ecosystem
- **Pulumi**: Real programming languages (Python, TypeScript), newer

**Terraform vs ARM Templates:**
- **Terraform**: Multi-cloud, better syntax
- **ARM**: Azure-native, integrated with Azure

## 9. Real-World Use Cases

**Multi-Cloud Infrastructure:**
Companies use Terraform to manage resources across AWS, Azure, and GCP with consistent tooling and processes.

**Environment Provisioning:**
Automated creation of development, staging, and production environments with identical configurations.

**Disaster Recovery:**
Quick infrastructure recreation in different regions or cloud providers during outages.

**Compliance and Governance:**
Enforcing infrastructure standards through modules and policies across organizations.

**Microservices Infrastructure:**
Managing complex infrastructure for containerized applications including EKS, load balancers, and networking.

**Database Infrastructure:**
Provisioning and managing RDS instances, MongoDB clusters, and associated networking components.

## 10. Study Checklist

**Must Master Before Claiming Terraform Expertise:**

**Fundamentals:**
- [ ] Understand IaC principles and benefits
- [ ] Know Terraform workflow (write, plan, apply)
- [ ] Understand state management concepts
- [ ] Can write basic resource configurations

**Core Skills:**
- [ ] Use variables, outputs, and locals effectively
- [ ] Create and use modules
- [ ] Implement data sources
- [ ] Handle resource dependencies
- [ ] Use built-in functions

**Advanced Topics:**
- [ ] Configure remote state and backends
- [ ] Implement state locking
- [ ] Use workspaces for environment management
- [ ] Handle sensitive data securely
- [ ] Implement proper error handling

**Operations:**
- [ ] Set up CI/CD pipelines with Terraform
- [ ] Perform state management operations
- [ ] Handle version upgrades
- [ ] Implement monitoring and logging
- [ ] Troubleshoot common issues

**Best Practices:**
- [ ] Follow security best practices
- [ ] Implement proper code organization
- [ ] Use consistent naming conventions
- [ ] Document infrastructure code
- [ ] Implement testing strategies

**Real-World Application:**
- [ ] Deploy multi-tier applications
- [ ] Manage multiple environments
- [ ] Implement disaster recovery
- [ ] Handle compliance requirements
- [ ] Optimize for cost and performance

---

**Final Note:** Practice with real cloud accounts, contribute to open-source Terraform modules, and stay updated with HashiCorp's releases and best practices documentation.