---
type: agent_requested
description: Write infrastructure as code with Terraform. Use for HCL modules, remote state backends, workspace strategies, plan/apply workflows, and multi-environment IaC patterns.
---

# Terraform Engineer

## When to Use
- Building or refactoring Terraform modules
- Configuring remote state backends with locking
- Setting up multi-environment infrastructure (dev/staging/prod)
- Implementing `terraform plan/apply` CI/CD workflows
- Designing module interfaces (variables, outputs, validation)
- Migrating existing infra to Terraform

## Standard Module Layout

```
terraform-<provider>-<name>/
├── main.tf        # Primary resource definitions
├── variables.tf   # Input declarations with validation
├── outputs.tf     # Output value definitions
├── versions.tf    # Provider + terraform version constraints
├── examples/      # Working usage examples
└── tests/         # Automated tests (terratest, etc.)
```

## HCL Patterns

**variables.tf — always validate inputs:**
```hcl
variable "name" {
  description = "Resource name prefix (1-32 chars)"
  type        = string
  validation {
    condition     = length(var.name) >= 1 && length(var.name) <= 32
    error_message = "Name must be 1-32 characters."
  }
}
variable "tags" {
  description = "Common tags applied to all resources"
  type        = map(string)
  default     = {}
}
```

**main.tf — use for_each over count, dynamic blocks for repetition:**
```hcl
resource "aws_subnet" "private" {
  for_each          = var.private_subnets
  vpc_id            = aws_vpc.this.id
  cidr_block        = each.value.cidr_block
  availability_zone = each.value.az
  tags              = merge(var.tags, { Name = "${var.name}-${each.key}" })
}

resource "aws_security_group" "this" {
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

**versions.tf — always pin provider versions:**
```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}
```

**locals.tf — compute derived values once:**
```hcl
locals {
  environment  = terraform.workspace
  common_tags  = merge(var.tags, { Environment = local.environment, ManagedBy = "terraform" })
  vpc_cidr     = { production = "10.0.0.0/16", staging = "10.1.0.0/16", dev = "10.2.0.0/16" }
}
```

## Remote State Backends

**AWS S3 + DynamoDB locking (recommended):**
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/vpc/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

**Azure Blob (locking automatic):**
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstatestorage"
    container_name       = "tfstate"
    key                  = "production.terraform.tfstate"
    use_azuread_auth     = true
  }
}
```

**State file organization:**
```
state-bucket/
├── production/vpc/terraform.tfstate
├── production/eks/terraform.tfstate
├── staging/vpc/terraform.tfstate
└── dev/vpc/terraform.tfstate
```

## Workspace Strategy

**Workspaces** — best for *similar* environments with minor config differences:
```bash
terraform workspace new staging
terraform workspace select production
terraform workspace show
```

**Separate directories** — better for *significantly different* environments (different modules, providers, account IDs):
```
infra/
├── environments/production/   # has its own backend + tfvars
├── environments/staging/
└── modules/vpc/               # shared module
```

Use partial backend config for environment-specific state paths:
```bash
terraform init -backend-config=config/backend-prod.hcl
```

## plan/apply Workflow

```bash
terraform fmt          # Format code
terraform validate     # Validate syntax and types
terraform plan -out=tfplan   # Preview changes, save plan
terraform show tfplan        # Human-readable review
terraform apply tfplan       # Apply saved plan (no prompt)

# State inspection
terraform state list
terraform state show aws_vpc.main
terraform state mv aws_instance.old aws_instance.new
terraform state rm aws_instance.orphan  # remove without destroy
```

## Environment Management Strategies

| Approach | When to use | Tradeoff |
|----------|-------------|----------|
| Workspaces | Same topology, small config delta | Simple but shares state backend |
| Directories | Different account/region/topology | More explicit, more copy-paste |
| Terragrunt | DRY across many envs | Extra tooling overhead |

## Must Do / Must Not Do

✅ Use remote state with locking for any shared/team environment  
✅ Pin provider versions (`~> 5.0` not `>= 5.0`)  
✅ Validate all input variables with `validation {}` blocks  
✅ Tag every taggable resource for cost attribution  
✅ Run `terraform fmt && terraform validate` in CI  
✅ Use semantic versioning for published modules  
❌ Store secrets in `.tfvars` or state (use Vault/SSM/Key Vault)  
❌ Use local state for production  
❌ Commit `.terraform/` directories or `*.tfstate` files  
❌ Use `count` for resources that may be reordered (use `for_each`)  
❌ Create circular module dependencies  

## References
- Source: https://github.com/Jeffallan/claude-skills/tree/main/skills/terraform-engineer

