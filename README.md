# Task 15: Terraform Workspaces Research

## Overview
Terraform Workspaces is a feature that enables management of multiple environments (Production, Development, Staging, etc.) within a single Terraform configuration without affecting other workspaces. Each workspace maintains its own isolated state file, allowing safe parallel development and deployment.

---

## What are Terraform Workspaces?

Terraform Workspaces provide environment isolation by:
- Creating separate state files for each workspace
- Allowing switching between environments without configuration changes
- Maintaining isolated infrastructure changes per workspace
- Storing all workspace states in a dedicated directory structure

**Use Cases:**
- Managing development, staging, and production environments
- Testing infrastructure changes in isolation
- Running parallel deployments for different clients/projects
- Creating temporary environments for feature testing

---

## Workspace Commands

### Create a New Workspace
```bash
terraform workspace new <workspace-name>
```
Creates a new workspace and automatically switches to it.

**Example:**
```bash
terraform workspace new development
terraform workspace new staging
terraform workspace new production
```

---

### List All Workspaces
```bash
terraform workspace list
```
Displays all available workspaces with an asterisk (*) indicating the current workspace.

**Output Example:**
```
  default
  development
* staging
  production
```

---

### Switch Between Workspaces
```bash
terraform workspace select <workspace-name>
```
Switches to the specified workspace. All subsequent Terraform operations will use that workspace's state.

**Example:**
```bash
terraform workspace select production
```

---

### Delete a Workspace
```bash
terraform workspace delete <workspace-name>
```
Deletes the specified workspace. Cannot delete a workspace with existing infrastructure or the currently active workspace.

**Example:**
```bash
terraform workspace delete development
```

**Important:** Destroy all resources in the workspace before deleting it.

---

### Select or Create Workspace
```bash
terraform workspace select -or-create <workspace-name>
```
Selects an existing workspace or creates it if it doesn't exist. Useful for automation scripts.

---

### Show Current Workspace
```bash
terraform workspace show
```
Displays the name of the currently active workspace.

---

### Get Help
```bash
terraform workspace help
```
Shows the complete help documentation for workspace commands.

---

## State File Management

### Default State Storage
Without workspaces, Terraform stores state in:
```
terraform.tfstate
```

### With Workspaces
Terraform creates a directory structure:
```
terraform.tfstate.d/
├── development/
│   └── terraform.tfstate
├── staging/
│   └── terraform.tfstate
└── production/
    └── terraform.tfstate
```

**Key Points:**
- Each workspace has its own isolated state file
- The `default` workspace still uses `terraform.tfstate` in the root
- State files are automatically managed by Terraform
- Never manually edit or move these files

---

## Workspace-Specific Variables

Terraform does **not** automatically load different `.tfvars` files for different workspaces. You must implement one of these methods:

### Method 1: Variable Maps

Define a map variable with values for each environment:

```hcl
variable "instance_type" {
  description = "Instance type per environment"
  type        = map(string)
  default = {
    development = "t3.micro"
    staging     = "t3.small"
    production  = "t3.medium"
  }
}

resource "aws_instance" "example" {
  instance_type = var.instance_type[terraform.workspace]
  # ... other configuration
}
```

**Usage:**
```bash
terraform workspace select production
terraform apply  # Uses t3.medium
```

---

### Method 2: Workspace-Based Locals

Use locals with conditional logic based on workspace name:

```hcl
locals {
  environment = terraform.workspace
  
  instance_type = terraform.workspace == "production" ? "t3.medium" : (
    terraform.workspace == "staging" ? "t3.small" : "t3.micro"
  )
  
  instance_count = {
    development = 1
    staging     = 2
    production  = 5
  }[terraform.workspace]
  
  tags = {
    Environment = terraform.workspace
    ManagedBy   = "Terraform"
  }
}

resource "aws_instance" "example" {
  count         = local.instance_count
  instance_type = local.instance_type
  tags          = local.tags
}
```

---

### Method 3: Workspace-Specific tfvars Files (Manual)

Create separate variable files:
```
terraform.development.tfvars
terraform.staging.tfvars
terraform.production.tfvars
```

Apply with explicit file:
```bash
terraform workspace select production
terraform apply -var-file="terraform.production.tfvars"
```

---

## Accessing Current Workspace

Use the `terraform.workspace` expression to reference the current workspace:

```hcl
# In resource names
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name        = "vpc-${terraform.workspace}"
    Environment = terraform.workspace
  }
}

# In conditionals
resource "aws_instance" "monitoring" {
  count = terraform.workspace == "production" ? 1 : 0
  # Only create in production
}

# In outputs
output "environment" {
  value = "Current workspace: ${terraform.workspace}"
}
```

---

## Independent Infrastructure Management

### Key Considerations

1. **Check Active Workspace**
   ```bash
   terraform workspace show
   ```
   Always verify the current workspace before running commands.

2. **Apply Changes**
   ```bash
   terraform workspace select staging
   terraform apply
   ```
   Infrastructure changes apply only to the selected workspace's state.

3. **Isolated Operations**
   - Each workspace maintains independent infrastructure
   - Changes in one workspace don't affect others
   - State files are completely separate

### Workflow Example

```bash
# Create development environment
terraform workspace new development
terraform apply -auto-approve

# Create staging environment
terraform workspace new staging
terraform apply -auto-approve

# Make changes to staging only
terraform workspace select staging
terraform apply

# Production remains unchanged
terraform workspace select production
terraform plan  # Shows no changes
```

---

## Remote Backend with Workspaces

### S3 + DynamoDB Configuration

When using remote backends, Terraform automatically handles multiple workspaces with a single S3 bucket and DynamoDB table.

**Backend Configuration:**
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "project/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

### State File Key Structure

Terraform stores workspace states using different S3 keys:

```
# Default workspace
project/terraform.tfstate

# Named workspaces
project/env:/development/terraform.tfstate
project/env:/staging/terraform.tfstate
project/env:/production/terraform.tfstate
```

**Pattern:**
- Default: `<key>`
- Workspaces: `<key-prefix>/env:/<workspace-name>/<key-suffix>`

---

### DynamoDB Lock Management

Each workspace gets a separate lock entry in the same DynamoDB table:

| LockID | Info |
|--------|------|
| `my-bucket/project/terraform.tfstate-md5` | Default workspace |
| `my-bucket/project/env:/development/terraform.tfstate-md5` | Development |
| `my-bucket/project/env:/staging/terraform.tfstate-md5` | Staging |
| `my-bucket/project/env:/production/terraform.tfstate-md5` | Production |

**Benefits:**
- Single S3 bucket for all workspaces
- Single DynamoDB table for all locks
- Concurrent operations across workspaces
- Reduced resource redundancy
- Simplified management

---

### Remote Backend Best Practices

1. **Enable Versioning**
   ```hcl
   resource "aws_s3_bucket_versioning" "state" {
     bucket = aws_s3_bucket.terraform_state.id
     
     versioning_configuration {
       status = "Enabled"
     }
   }
   ```

2. **Enable Encryption**
   ```hcl
   resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
     bucket = aws_s3_bucket.terraform_state.id
     
     rule {
       apply_server_side_encryption_by_default {
         sse_algorithm = "AES256"
       }
     }
   }
   ```

3. **Block Public Access**
   ```hcl
   resource "aws_s3_bucket_public_access_block" "state" {
     bucket = aws_s3_bucket.terraform_state.id
     
     block_public_acls       = true
     block_public_policy     = true
     ignore_public_acls      = true
     restrict_public_buckets = true
   }
   ```

---

## Complete Workspace Example

### Directory Structure
```
project/
├── main.tf
├── variables.tf
├── outputs.tf
├── backend.tf
└── terraform.tfstate.d/
    ├── development/
    ├── staging/
    └── production/
```

### Configuration Files

**backend.tf**
```hcl
terraform {
  backend "s3" {
    bucket         = "company-terraform-states"
    key            = "app/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

**main.tf**
```hcl
locals {
  environment = terraform.workspace
  
  config = {
    development = {
      instance_type = "t3.micro"
      instance_count = 1
      enable_monitoring = false
    }
    staging = {
      instance_type = "t3.small"
      instance_count = 2
      enable_monitoring = true
    }
    production = {
      instance_type = "t3.medium"
      instance_count = 5
      enable_monitoring = true
    }
  }
  
  env_config = local.config[terraform.workspace]
}

resource "aws_instance" "app" {
  count         = local.env_config.instance_count
  instance_type = local.env_config.instance_type
  
  tags = {
    Name        = "app-${terraform.workspace}-${count.index + 1}"
    Environment = terraform.workspace
  }
}
```

---

## Advantages of Workspaces

✅ **Environment Isolation** - Separate state files prevent accidental changes  
✅ **Simplified Management** - Single codebase for multiple environments  
✅ **Cost Efficiency** - Reuse S3 bucket and DynamoDB table  
✅ **Parallel Operations** - Multiple teams can work simultaneously  
✅ **Easy Switching** - Quick environment transitions with one command  
✅ **Reduced Complexity** - No need for separate backend configurations  

---

## Limitations and Considerations

⚠️ **Variable Management** - No automatic variable file switching  
⚠️ **Naming Conflicts** - Must ensure workspace names don't conflict  
⚠️ **State Access** - All workspaces share backend access permissions  
⚠️ **Not for Multi-Region** - Better suited for environments, not geographic isolation  
⚠️ **Backend Migration** - Moving between backends requires careful planning  

---

## Best Practices

1. **Use Descriptive Names**
   ```bash
   terraform workspace new prod-us-east-1
   terraform workspace new staging-eu-west-1
   ```

2. **Always Verify Active Workspace**
   ```bash
   terraform workspace show
   terraform plan  # Review before apply
   ```

3. **Document Workspace Strategy**
   - Define naming conventions
   - Document variable patterns
   - Maintain workspace inventory

4. **Implement Safeguards**
   ```hcl
   # Prevent accidental production changes
   resource "null_resource" "workspace_check" {
     count = terraform.workspace == "production" ? 0 : 1
     
     provisioner "local-exec" {
       command = "echo 'WARNING: Not in production workspace'"
     }
   }
   ```

5. **Use Remote State for Teams**
   - Always use S3/DynamoDB for collaboration
   - Enable state locking
   - Implement proper IAM permissions

---

## Common Commands Workflow

```bash
# Initialize with remote backend
terraform init

# Create and switch to development
terraform workspace new development
terraform apply

# Create staging from development
terraform workspace new staging
terraform apply

# Promote to production
terraform workspace new production
terraform apply

# List all workspaces
terraform workspace list

# Switch between environments
terraform workspace select staging
terraform plan

# Clean up
terraform workspace select development
terraform destroy
terraform workspace delete development
```

---

## Troubleshooting

**Issue**: Cannot delete workspace  
**Solution**: Destroy all resources first, then delete workspace

**Issue**: Wrong workspace applied  
**Solution**: Always run `terraform workspace show` before operations

**Issue**: State file conflicts  
**Solution**: Ensure only one person/process modifies a workspace at a time

**Issue**: Variables not loading  
**Solution**: Implement workspace-specific variable maps or locals

---

## Summary

Terraform Workspaces provide a lightweight mechanism for managing multiple environments within a single Terraform configuration. By maintaining separate state files and enabling easy switching between workspaces, teams can safely develop, test, and deploy infrastructure across different environments without risking cross-contamination or requiring complex directory structures.
