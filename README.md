# AWS-CodePipeline-Combo-with-Terraform-Security-Scanning
![image](https://github.com/user-attachments/assets/92acd0e2-8682-4700-8a70-2a33729c6a2f)


Here’s a **comprehensive project walkthrough** for setting up a **secure, validated CI/CD pipeline** using **AWS CodePipeline** with **Terraform**, integrating security scans like **TFLint**, **Checkov**, and **TFSec**, and deploying validated infrastructure via automated pipelines.

---

```markdown
#  AWS CodePipeline Combo with Terraform + Security Scanning

This project demonstrates how to automate infrastructure deployments using:
- AWS CodePipeline
- Terraform (IAC)
- CodeCommit, CodeBuild, S3, KMS, SNS
- ECR for Docker images
- TFLint, Checkov, and TFSec for scans and validations

---

##  Prerequisites

- AWS CLI & IAM permissions
- Terraform installed
- ECR repository created
- Clone the example repo:

```bash
git clone https://github.com/ooghenekaro/aws-codepipeline-teraform.git
cd aws-codepipeline-teraform
```

---

## Step 1: Create ECR Repository

Create an Elastic Container Registry (ECR) to store your custom image.

```bash
aws ecr create-repository --repository-name my-terraform-ci --region us-east-1
```

Take note of the repository URI returned.

---

## Step 2: Build and Push Docker Image to ECR

### 1. Authenticate Docker to AWS ECR

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com
```

### 2. Build and Push the Image

```bash
docker build -t my-terraform-ci .
docker tag my-terraform-ci:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/my-terraform-ci:latest
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/my-terraform-ci:latest
```

---

## Step 3: Set Up CodeCommit and CodeBuild (via Terraform)

Update `terraform.tfvars`:

```hcl
app_name = "my-terraform-app"
sns_email = "alerts@example.com"
```

Apply Terraform code:

```bash
terraform init
terraform plan -out=tf.plan
terraform apply tf.plan
```

This creates:
- CodeCommit repository
- CodeBuild project for Terraform
- SNS topic and subscription for notifications

---

## Step 4: Set Up CodePipeline with Terraform

Pipeline Stages:
1. **Source** – CodeCommit
2. **Validate** – `terraform fmt` and `validate`
3. **Lint** – `tflint`
4. **Security Scan** – `checkov` and `tfsec`
5. **Plan**
6. **Manual Approval (Optional)**
7. **Apply**

Update `buildspec.yml` for each stage.

### Sample `buildspec.yml`

```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      python: 3.9
    commands:
      - pip install checkov tfsec tflint

  pre_build:
    commands:
      - terraform fmt -check
      - terraform validate
      - tflint --init
      - tflint
      - checkov -d .
      - tfsec .

  build:
    commands:
      - terraform init
      - terraform plan -out=tfplan

  post_build:
    commands:
      - terraform apply -auto-approve tfplan
```

---

## Step 5: Remote Backend Setup

Create your S3 bucket and DynamoDB for Terraform state:

```bash
aws s3api create-bucket --bucket terraform-state-bucket-123 --region us-east-1
aws dynamodb create-table \
  --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

### Add to `backend.tf`

```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-bucket-123"
    key            = "codepipeline/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

Reinitialize Terraform:

```bash
terraform init -migrate-state
```

---

## Step 6: Deploy Infrastructure via CodePipeline

After pushing to CodeCommit, CodePipeline will trigger:

- Terraform validation & fmt
- TFLint scan
- Checkov scan
- TFSec scan
- Plan & Apply

---

## Step 7: Test Compliance Enforcement

### Scenario A: Non-compliant Bucket

Add a bucket without encryption or versioning to test:

```hcl
resource "aws_s3_bucket" "bad_bucket" {
  bucket = "non-compliant-bucket-demo"
}
```

Expected: **Checkov or TFSec will fail**, pipeline halts before `apply`.

---

### Scenario B: Compliant Bucket with KMS

```hcl
resource "aws_s3_bucket" "secure_bucket" {
  bucket = "compliant-bucket-demo"
  versioning {
    enabled = true
  }
  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm     = "aws:kms"
        kms_master_key_id = "alias/my-kms-key"
      }
    }
  }
}
```

Expected: **Pipeline passes**, and Terraform deploys the bucket.

---

##  You Can Now Reuse This Pipeline to Deploy

- ECS or EKS Clusters
- S3 Buckets, IAM roles
- Lambda, API Gateway
- Multi-account environments

Just push new `.tf` files to the CodeCommit repo — and the pipeline takes care of the rest.

Once your AWS **CodePipeline** is configured with **CodeCommit** as the source and **CodeBuild** stages (validate, scan, plan, apply), the entire workflow becomes **event-driven**.

###  How it works:
1. You push any new `.tf` file or update to the **CodeCommit** repository:
   ```bash
   git add .
   git commit -m "Add new S3 bucket module"
   git push origin main
   ```

2. AWS CodePipeline detects the change through the **Source** stage.

3. It triggers the pipeline which performs:
   - **`terraform fmt`** → Format check
   - **`terraform validate`** → Syntax validation
   - **`tflint`** → Linting and style
   - **`checkov` + `tfsec`** → Security and compliance scans
   - **`terraform plan`**
   - Optional **manual approval**
   - **`terraform apply`**

4. If everything passes, your infrastructure is **automatically deployed to AWS**.

---

###  What this means:
- **No manual `terraform apply` required**
- **No direct AWS Console clicks**
- **Security built-in**
- You can even manage **multi-environment (dev, stage, prod)** with branches and pipeline filters.

---

![image](https://github.com/user-attachments/assets/25a0327e-1cea-436e-8e85-84626ffc4277)


---

## 🛡 Scanning Tools Recap

| Tool     | Purpose                                   |
|----------|-------------------------------------------|
| TFLint   | Linting and style conventions for Terraform |
| Checkov  | Compliance and security scanning           |
| TFSec    | Security vulnerabilities and misconfigurations |

---

## 📂 Useful Repos

- Terraform CodePipeline Starter: [github.com/lily4499/aws-codepipeline-teraform](https://github.com/lily4499/aws-codepipeline-teraform)
- Checkov Docs: https://www.checkov.io/
- TFSec Docs: https://aquasecurity.github.io/tfsec
- TFLint Docs: https://github.com/terraform-linters/tflint

---


```


 Below is the full reorganization of the **AWS CodePipeline with Terraform** project using **GitHub instead of CodeCommit** as the source.

This setup:
- Uses **GitHub as the source**
- Triggers CodePipeline on each commit to GitHub
- Runs **validation, security scans**, and **Terraform apply**
- Uses **Terraform** to deploy all infrastructure (S3, IAM, ECR, CodeBuild, CodePipeline)

---

```markdown
#  GitHub-Based AWS CodePipeline CI/CD for Terraform with Security Scanning

Deploy secure and compliant Terraform code using:
- GitHub as the source
- AWS CodePipeline + CodeBuild
- Terraform CLI
- Security tools: TFLint, Checkov, TFSec

---

##  Project Overview

**Stages:**

1. Push `.tf` files to GitHub
2. CodePipeline triggers
3. CodeBuild stages:
   - `terraform fmt` + `validate`
   - `tflint`
   - `checkov`
   - `tfsec`
   - `terraform plan`
   - `manual approval` (optional)
   - `terraform apply`

---

## 1️ Prerequisites

- AWS CLI & Terraform installed
- GitHub repository created
- Docker installed for local builds (optional)

---

## 2️ Provision Infrastructure using Terraform

### Structure:
```
.
├── main.tf
├── backend.tf
├── variables.tf
├── buildspec.yml
├── terraform.tfvars
```

---

### Example: `main.tf`

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "artifact_store" {
  bucket = "my-terraform-artifact-store"
  acl    = "private"
}

resource "aws_kms_key" "pipeline_key" {
  description             = "KMS key for CodePipeline"
  deletion_window_in_days = 10
}

resource "aws_codebuild_project" "terraform_build" {
  name          = "TerraformBuild"
  description   = "Build project for Terraform"
  service_role  = aws_iam_role.codebuild_role.arn
  artifacts {
    type = "CODEPIPELINE"
  }
  source {
    type      = "CODEPIPELINE"
    buildspec = "buildspec.yml"
  }
  environment {
    compute_type                = "BUILD_GENERAL1_SMALL"
    image                       = "aws/codebuild/standard:6.0"
    type                        = "LINUX_CONTAINER"
    environment_variables = [
      {
        name  = "TF_VAR_env"
        value = "prod"
      }
    ]
  }
}

resource "aws_codepipeline" "terraform_pipeline" {
  name     = "terraform-ci-pipeline"
  role_arn = aws_iam_role.pipeline_role.arn

  artifact_store {
    location = aws_s3_bucket.artifact_store.bucket
    type     = "S3"
    encryption_key {
      id   = aws_kms_key.pipeline_key.arn
      type = "KMS"
    }
  }

  stage {
    name = "Source"
    action {
      name             = "SourceAction"
      category         = "Source"
      owner            = "ThirdParty"
      provider         = "GitHub"
      version          = "1"
      output_artifacts = ["SourceOutput"]
      configuration = {
        Owner      = "<github-user>"
        Repo       = "<github-repo>"
        Branch     = "main"
        OAuthToken = var.github_token
      }
    }
  }

  stage {
    name = "Build"
    action {
      name             = "TerraformBuild"
      category         = "Build"
      owner            = "AWS"
      provider         = "CodeBuild"
      input_artifacts  = ["SourceOutput"]
      output_artifacts = ["BuildOutput"]
      version          = "1"
      configuration = {
        ProjectName = aws_codebuild_project.terraform_build.name
      }
    }
  }
}
```

---

### Example: `backend.tf`

```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-bucket-123"
    key            = "github-terraform/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

---

### Example: `variables.tf`

```hcl
variable "github_token" {
  type        = string
  description = "GitHub OAuth token"
}
```

---

### Example: `buildspec.yml`

```yaml
version: 0.2

phases:
  install:
    commands:
      - pip install checkov tfsec tflint
  pre_build:
    commands:
      - terraform fmt -check
      - terraform validate
      - tflint --init && tflint
      - checkov -d .
      - tfsec .
  build:
    commands:
      - terraform init
      - terraform plan -out=tfplan
  post_build:
    commands:
      - terraform apply -auto-approve tfplan
```

---

## 3️ Deploy Infrastructure

```bash
terraform init
terraform plan
terraform apply
```

---

## 4️ Push to GitHub

Once your pipeline is deployed:

```bash
git add .
git commit -m "Add S3 module"
git push origin main
```

Your **CodePipeline** will automatically:
- Pull source from GitHub
- Run validation and security scans
- Deploy your infrastructure

---

##  Security Built-In

| Tool     | Description                                   |
|----------|-----------------------------------------------|
| TFLint   | Linter for Terraform                          |
| Checkov  | Security + Compliance scanning                |
| TFSec    | Detects Terraform misconfigurations           |

---

##  End-to-End CI/CD Workflow with GitHub

1. Developer pushes `.tf` files to GitHub
2. AWS CodePipeline pulls changes
3. CodeBuild runs:
   - Format, validate, security scans
   - `terraform plan`
   - Optional `manual approval`
   - `terraform apply`

---

##  Summary

You’ve now replaced CodeCommit with GitHub and created:
- A secure, automated Terraform pipeline
- GitHub → CodePipeline → CodeBuild → AWS infra

---


---

