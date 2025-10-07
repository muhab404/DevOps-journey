# Terraform Guide

## 1. Infrastructure as Code (IaC) Concepts

**What is Infrastructure as Code**: Managing infrastructure through code rather than manual processes.

**Benefits**: Version control, repeatability, consistency, automation, documentation.

**Terraform Purpose**: Declarative tool for building, changing, and versioning infrastructure safely and efficiently.

**Benefits of Terraform State**: Tracks resource mappings, metadata, performance optimization, and collaboration.

## 2. Terraform Basics

### Terraform Provider Block
```hcl
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
```

### Terraform Resource Block
```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t2.micro"
  
  tags = {
    Name = "WebServer"
  }
}
```

### Terraform Input Variables
```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "availability_zones" {
  description = "List of AZs"
  type        = list(string)
  default     = ["us-west-2a", "us-west-2b"]
}
```

### Terraform Local Variables
```hcl
locals {
  common_tags = {
    Environment = "production"
    Project     = "web-app"
  }
  
  instance_name = "${var.environment}-${var.project}-instance"
}
```

### Terraform Data Block
```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
}
```

### Terraform Configuration Block
```hcl
terraform {
  required_version = ">= 1.0"
  
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-west-2"
  }
}
```

### Terraform Module Block
```hcl
module "vpc" {
  source = "./modules/vpc"
  
  cidr_block = "10.0.0.0/16"
  name       = "main-vpc"
}
```

### Terraform Output Block
```hcl
output "instance_ip" {
  description = "Public IP of the instance"
  value       = aws_instance.web.public_ip
}

output "vpc_id" {
  value = module.vpc.vpc_id
}
```

## 3. Providers & Provisioners

### Provider Installation and Versioning
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.0"
    }
  }
}
```

### Using Multiple Providers
```hcl
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "us_west"
  region = "us-west-2"
}

resource "aws_instance" "east" {
  provider = aws.us_east
  # ... configuration
}
```

### Generating SSH Keys using TLS Provider
```hcl
resource "tls_private_key" "ssh_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "deployer" {
  key_name   = "deployer-key"
  public_key = tls_private_key.ssh_key.public_key_openssh
}
```

### Terraform Provisioners
```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t2.micro"
  
  provisioner "remote-exec" {
    inline = [
      "sudo apt update",
      "sudo apt install -y nginx"
    ]
    
    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = tls_private_key.ssh_key.private_key_pem
      host        = self.public_ip
    }
  }
}
```

## 4. Terraform State Management

### Backend Configuration
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### Remote State Data Source
```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "vpc/terraform.tfstate"
    region = "us-west-2"
  }
}
```

## 5. Terraform Modules

### Module Structure
```hcl
# modules/ec2/main.tf
variable "instance_type" {
  type = string
}

variable "ami_id" {
  type = string
}

resource "aws_instance" "this" {
  ami           = var.ami_id
  instance_type = var.instance_type
}

output "instance_id" {
  value = aws_instance.this.id
}

# Root module usage
module "web_server" {
  source = "./modules/ec2"
  
  instance_type = "t2.micro"
  ami_id        = data.aws_ami.ubuntu.id
}
```

### Module Versioning
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 3.0"
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
}
```

## 6. Terraform Workflow

Commands are executed in sequence:
- `terraform init` - Initialize working directory
- `terraform validate` - Validate configuration
- `terraform plan` - Create execution plan
- `terraform apply` - Apply changes
- `terraform destroy` - Destroy infrastructure

## 7. Variables, Functions, and Advanced Config

### Variable Validation
```hcl
variable "instance_type" {
  type = string
  
  validation {
    condition = contains([
      "t2.micro", "t2.small", "t2.medium"
    ], var.instance_type)
    error_message = "Instance type must be t2.micro, t2.small, or t2.medium."
  }
}
```

### Built-in Functions
```hcl
locals {
  # String functions
  upper_name = upper(var.project_name)
  
  # Collection functions
  subnet_ids = values(aws_subnet.private)[*].id
  
  # Encoding functions
  user_data = base64encode(file("${path.module}/user-data.sh"))
  
  # Date/time functions
  timestamp = formatdate("YYYY-MM-DD", timestamp())
}
```

### Dynamic Blocks
```hcl
resource "aws_security_group" "web" {
  name = "web-sg"
  
  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}

variable "ingress_ports" {
  type    = list(number)
  default = [80, 443, 22]
}
```

### Resource Lifecycles
```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t2.micro"
  
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    ignore_changes       = [ami]
  }
}
```

## 8. Debugging & Utilities

### Debugging Environment Variables
```bash
export TF_LOG=DEBUG
export TF_LOG_PATH=terraform.log
terraform apply
```

### Auto Formatting
```bash
terraform fmt -recursive
```

## 9. HCP Terraform (Terraform Cloud)

### Remote Backend Configuration
```hcl
terraform {
  cloud {
    organization = "my-org"
    
    workspaces {
      name = "my-workspace"
    }
  }
}
```

### Workspace Variables
```hcl
# Set via Terraform Cloud UI or API
# Environment variables: TF_VAR_instance_type
# Terraform variables: instance_type
```