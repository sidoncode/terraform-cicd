# 🚀 Terraform + Nginx EC2 with GitHub Actions CI/CD

Deploy an **Nginx web server on AWS EC2** using Terraform, with remote state stored in **S3 + DynamoDB**, and fully automated via **GitHub Actions**.

---

## 📐 Architecture

```
GitHub Actions (CI/CD)
        │
        ├── PR  →  plan  →  💬 comment on PR
        └── Merge to main  →  apply  →  🌐 Nginx live
                │
                ▼
        ┌───────────────┐        ┌────────────────────┐
        │   S3 Bucket   │        │  DynamoDB Table    │
        │  (tfstate)    │        │  (state lock)      │
        │  versioned ✓  │        │  LockID (HASH)     │
        │  encrypted ✓  │        │  PAY_PER_REQUEST   │
        │  private   ✓  │        └────────────────────┘
        └───────────────┘
                │
                ▼
        ┌───────────────────────────────┐
        │         AWS EC2               │
        │  AMI: Ubuntu (us-east-1)      │
        │  Type: t3.micro               │
        │  Nginx → port 80              │
        │  SSH   → port 22              │
        └───────────────────────────────┘
```

---

## 📁 Project Structure

```
.
├── main.tf                          # EC2 + Security Group
├── outputs.tf                       # Public IP + Nginx URL
├── .github/
│   └── workflows/
│       └── terraform.yml            # GitHub Actions CI/CD
└── README.md
```

---

## ✅ Prerequisites

| Requirement | Details |
|---|---|
| AWS CLI | v2, configured via `aws configure` |
| IAM Permissions | `AmazonS3FullAccess`, `AmazonDynamoDBFullAccess`, `AmazonEC2FullAccess` |
| Terraform | v1.7+ |
| GitHub Repo | Secrets configured (see below) |

---

## 🪣 Part 1 — Remote State Setup (AWS CLI)

Run these **once** before anything else. This creates the S3 bucket and DynamoDB table that store your Terraform state.

### Create the S3 Bucket

```bash
# 1. Create the bucket
aws s3api create-bucket \
  --bucket my-terraform-state-2024 \
  --region us-east-1

# 2. Enable versioning (recover old state files)
aws s3api put-bucket-versioning \
  --bucket my-terraform-state-2024 \
  --versioning-configuration Status=Enabled

# 3. Block all public access (state files must be private)
aws s3api put-public-access-block \
  --bucket my-terraform-state-2024 \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,\
     BlockPublicPolicy=true,RestrictPublicBuckets=true"

# 4. Enable encryption at rest (AES-256)
aws s3api put-bucket-encryption \
  --bucket my-terraform-state-2024 \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

### Create the DynamoDB Lock Table

```bash
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

### Verify in AWS Console

| Resource | Where to Check |
|---|---|
| S3 Bucket | Console → S3 → Buckets |
| Versioning | Bucket → Properties → Bucket Versioning → **Enabled** |
| Encryption | Bucket → Properties → Default Encryption → **SSE-S3** |
| Public Access | Bucket → Permissions → Block Public Access → all **ON** |
| DynamoDB | Console → DynamoDB → Tables → `terraform-state-lock` |

---

## 📄 Part 2 — Terraform Files

### `main.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "my-terraform-state-2024"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}

provider "aws" {
  region = "us-east-1"
}

# ── Security Group ────────────────────────────────────────────────
resource "aws_security_group" "nginx_sg" {
  name        = "nginx-sg"
  description = "Security group for nginx server"

  ingress {                          # HTTP
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {                          # SSH
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {                           # All outbound
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "nginx-sg" }
}

# ── EC2 Instance ──────────────────────────────────────────────────
resource "aws_instance" "nginx_server" {
  ami             = "ami-091138d0f0d41ff90"   # Ubuntu, us-east-1
  instance_type   = "t3.micro"
  security_groups = [aws_security_group.nginx_sg.name]

  user_data = <<-SCRIPT
    #!/bin/bash
    sudo apt update -y
    sudo apt install -y nginx
    systemctl start nginx
    systemctl enable nginx
    echo "<h1>Hello from Terraform + Nginx!</h1>" \
      > /var/www/html/index.html
  SCRIPT

  tags = { Name = "nginx-ec2-server" }
}
```

### `outputs.tf`

```hcl
output "instance_public_ip" {
  description = "Public IP of the nginx EC2 instance"
  value       = aws_instance.nginx_server.public_ip
}

output "nginx_url" {
  description = "Nginx URL"
  value       = "http://${aws_instance.nginx_server.public_ip}"
}
```

---

## 🔑 Part 3 — GitHub Secrets

Go to your repo → **Settings → Secrets and variables → Actions → New repository secret**

| Secret Name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Your IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | Your IAM user secret key |
| `AWS_REGION` | `us-east-1` |

> **IAM Policies needed on your user:**  `AmazonS3FullAccess` · `AmazonDynamoDBFullAccess` · `AmazonEC2FullAccess`

---

## ⚙️ Part 4 — GitHub Actions CI/CD

`.github/workflows/terraform.yml`

```yaml
name: Nginx EC2 — Terraform CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      action:
        description: "Apply or Destroy?"
        required: true
        default: "apply"
        type: choice
        options: [apply, destroy]

env:
  AWS_REGION: us-east-1
  TF_VERSION: 1.7.0

jobs:
  terraform:
    name: Terraform ${{ github.event.inputs.action || 'apply' }}
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id:     ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region:            ${{ env.AWS_REGION }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        run: terraform init

      - name: Terraform Format
        run: terraform fmt -check

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        id: plan
        run: |
          if [[ "${{ github.event.inputs.action }}" == "destroy" ]]; then
            terraform plan -destroy -no-color -out=tfplan
          else
            terraform plan -no-color -out=tfplan
          fi

      - name: Post Plan to PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        env:
          PLAN: ${{ steps.plan.outputs.stdout }}
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `#### 🗺️ Terraform Plan\n\`\`\`\n${process.env.PLAN}\n\`\`\``
            });

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve tfplan

      - name: Terraform Destroy
        if: github.event.inputs.action == 'destroy'
        run: terraform apply -auto-approve tfplan

      - name: Show Nginx URL
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: |
          echo "✅ Nginx is live at:"
          terraform output nginx_url
```

---

## 🚀 Part 5 — Deployment Workflow

### First time / local run

```bash
terraform init      # connect to S3 backend
terraform fmt       # format code
terraform validate  # check syntax
terraform plan      # preview changes
terraform apply     # deploy
```

### Via GitHub Actions

| Action | How |
|---|---|
| Preview changes | Open a Pull Request → plan auto-posted as PR comment |
| Deploy | Merge PR to `main` → auto apply |
| Destroy infra | Actions tab → **Run workflow** → select **destroy** |

---

## 🛡️ Security Checklist

- [x] S3 state bucket versioning enabled
- [x] S3 public access fully blocked
- [x] S3 server-side encryption (AES-256)
- [x] DynamoDB state locking (prevents concurrent applies)
- [x] State encrypted in transit (`encrypt = true`)
- [x] AWS credentials stored as GitHub Secrets (never hardcoded)
- [ ] Restrict SSH `cidr_blocks` to your IP in production
- [ ] Use KMS customer-managed key instead of AES-256
- [ ] Scope IAM user to least-privilege custom policy

---

## 📚 References

- [Terraform S3 Backend](https://developer.hashicorp.com/terraform/language/backend/s3)
- [GitHub Actions — configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)
- [hashicorp/setup-terraform](https://github.com/hashicorp/setup-terraform)
