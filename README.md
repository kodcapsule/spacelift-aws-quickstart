# Getting Started with Spacelift: Deploying Your First EC2 Instance

Spacelift is a sophisticated CI/CD platform purpose-built for infrastructure-as-code (IaC), supporting Terraform, Pulumi, CloudFormation, and Kubernetes. Unlike running Terraform manually or via generic CI pipelines, Spacelift gives you policy-as-code, drift detection, run visibility, and clean state management out of the box.

This tutorial walks through everything you need to go from zero to a running EC2 instance managed by Spacelift.

---

## Prerequisites

Before you start, make sure you have:

- A [Spacelift account](https://spacelift.io) (free trial available)
- An AWS account with permissions to create IAM roles, EC2 instances, and VPC resources
- A GitHub, GitLab, Bitbucket, or Azure DevOps account (Spacelift needs a VCS to pull your IaC code from)
- Terraform basics — you don't need to be an expert, but you should know what a `resource` block is
- AWS CLI installed locally (optional, useful for verification)

---

## Step 1: Set Up Your Repository

Spacelift pulls your Terraform code from a Git repository, so start there.

1. Create a new repository (e.g., `spacelift-ec2-demo`) in GitHub or your VCS of choice.
2. Instead of one big `main.tf`, split the configuration into separate files. This is a common convention that keeps stacks readable and maintainable as they grow. Create the following five files in the repo root:

**`versions.tf`** — pins Terraform and provider versions:

```hcl
terraform {
  required_version = ">= 1.7.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

**`providers.tf`** — configures the AWS provider:

```hcl
provider "aws" {
  region = var.aws_region
}
```

**`variables.tf`** — declares all input variables:

```hcl
variable "aws_region" {
  description = "AWS region to deploy into"
  type        = string
  default     = "us-east-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}
```

**`main.tf`** — the actual resources (data source, security group, instance):

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_security_group" "web_sg" {
  name        = "spacelift-demo-sg"
  description = "Allow SSH and HTTP"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Restrict this in production
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "demo" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  tags = {
    Name      = "spacelift-demo-instance"
    ManagedBy = "spacelift"
  }
}
```

**`outputs.tf`** — exposes values you'll want to see after apply:

```hcl
output "instance_public_ip" {
  value = aws_instance.demo.public_ip
}

output "instance_id" {
  value = aws_instance.demo.id
}
```

Your repo structure should now look like this:

```
spacelift-ec2-demo/
├── versions.tf
├── providers.tf
├── variables.tf
├── main.tf
└── outputs.tf
```

3. Commit and push all five files to your repository's main branch. Terraform doesn't care about file names or boundaries — it loads and merges every `.tf` file in the directory as a single configuration — so this split is purely for human readability and doesn't change how Spacelift plans or applies the stack.

> **Tip:** Keep your first stack simple. You can always add complexity (modules, remote state data sources, multiple environments) once the basic flow works end-to-end.

---

## Step 2: Create Your Spacelift Account and Connect Your VCS

1. Sign up at [spacelift.io](https://spacelift.io) and choose your subdomain (e.g., `yourcompany.app.spacelift.io`).
2. During onboarding, Spacelift will prompt you to connect a version control system. Choose GitHub (or your provider) and authorize the Spacelift GitHub App.
3. Grant access to the repository you created in Step 1 (or to all repositories, if you prefer).

---

## Step 3: Connect Spacelift to AWS

Spacelift needs credentials to provision resources in your AWS account. The recommended, most secure approach is **OpenID Connect (OIDC)** federation rather than long-lived access keys.

### Option A: OIDC (Recommended)
> **Tip:** This feature is only available to paid Spacelift accounts. Please check out our pricing page for more information.

Refer to the official documentation for more infor [Use the Spacelift OIDC token to authenticate With AWS](https://docs.spacelift.io/integrations/cloud-providers/oidc/aws-oidc)

1. In AWS IAM, create an **Identity Provider**:
   - Provider type: `OpenID Connect`
   - Provider URL: `https://oidc.spacelift.io`
   - Audience: `https://spacelift.io`

2. Create an IAM role that trusts this identity provider. Attach a trust policy like:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<YOUR_ACCOUNT_ID>:oidc-provider/oidc.spacelift.io"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.spacelift.io:aud": "https://spacelift.io"
        },
        "StringLike": {
          "oidc.spacelift.io:sub": "space:<your-space-id>:(stack|module):<your-stack-slug>*"
        }
      }
    }
  ]
}
```

3. Attach a permissions policy to this role. For this tutorial, `AmazonEC2FullAccess` will work, but for production, scope this down to only the actions your stack needs (least privilege).

4. Note the role's ARN — you'll attach it to your Spacelift stack in Step 4.

### Option B: Static Access Keys (Simpler, Not Recommended for Production)

If you want to move faster for a first test:
1. Create an IAM user with programmatic access and EC2 permissions.
2. Generate an access key and secret.
3. You'll add these as environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`) directly on the Spacelift stack in Step 4.

---

## Step 4: Create Your Spacelift Stack

A **Stack** in Spacelift is the core unit of work — it ties together your source code, backend, credentials, and run policies.

1. From the Spacelift dashboard, click **Create Stack**.
2. **Select your repository** and branch (e.g., `main`).
3. **Set the project root** if your Terraform files aren't in the repo root.
4. **Choose the vendor**: Terraform (select the version, e.g., 1.7.x).
5. **Configure the backend**: Spacelift manages Terraform state for you automatically — no need to configure an S3 backend yourself unless you want to.
6. **Attach AWS credentials**:
   - If using OIDC: go to the stack's **Integrations** tab → **AWS** → paste the IAM role ARN from Step 3.
   - If using static keys: go to **Environment** tab → add `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` as environment variables (mark the secret as **sensitive**).
7. Click **Create Stack**.

---

## Step 5: Trigger Your First Run

1. On your new stack's page, click **Trigger** → **Start Plan**.
2. Spacelift will spin up an ephemeral worker, clone your repo, run `terraform init` and `terraform plan`, and show you the proposed changes in real time in the run logs.
3. Review the plan output. You should see Spacelift proposing to create:
   - A security group
   - An EC2 instance
4. If everything looks correct, click **Confirm** to approve the apply phase. Spacelift will run `terraform apply` and provision your resources.

> By default, Spacelift stacks require manual confirmation before applying changes — this is a safety feature. You can enable **auto-apply** later once you trust your pipeline.

---

## Step 6: Verify the Deployment

Once the run completes:

1. Check the **Outputs** section of the run — you should see `instance_public_ip` and `instance_id`.
2. Verify in the AWS Console (EC2 → Instances) that `spacelift-demo-instance` is running.
3. Optionally, SSH in or curl the public IP to confirm connectivity (note: this AMI doesn't have a web server installed by default, so port 80 won't respond unless you add a `user_data` script).

---

## Step 7: Make a Change and Watch Spacelift Track Drift

This is where Spacelift starts to shine over plain Terraform CLI usage:

1. Change `instance_type` from `t3.micro` to `t3.small` in `main.tf`, commit, and push.
2. Spacelift automatically detects the push and triggers a new run (if you've enabled the default VCS trigger).
3. Review and confirm the plan — Spacelift shows you exactly what will change before anything happens.
4. Try Spacelift's **drift detection**: schedule periodic drift checks in the stack's **Settings → Scheduling** tab so Spacelift alerts you if someone manually changes the instance in the AWS console.

---

## Optional Next Steps

Once your first stack is working, consider exploring:

- **Policies (OPA/Rego)** — enforce guardrails, like blocking overly permissive security groups or requiring tags on all resources.
- **Contexts** — reusable groups of environment variables/files you can attach to multiple stacks (e.g., shared AWS credentials across stacks).
- **Modules** — publish and version your own Terraform modules through Spacelift's private registry.
- **Stack dependencies** — chain stacks together (e.g., VPC stack → EC2 stack) using output references.
- **Notifications** — connect Slack or Microsoft Teams for run notifications.

---

## Cleaning Up

To avoid ongoing AWS charges, destroy the resources when you're done testing:

1. Go to your stack in Spacelift.
2. Click **Actions → Destroy**.
3. Confirm — Spacelift will run `terraform destroy` and tear down the EC2 instance and security group.

---

## Summary

| Step | What You Did |
|------|-------------|
| 1 | Wrote a basic Terraform config for an EC2 instance |
| 2 | Created a Spacelift account and connected your VCS |
| 3 | Set up AWS credentials via OIDC (or access keys) |
| 4 | Created a Spacelift stack pointing to your repo |
| 5 | Triggered a plan/apply run |
| 6 | Verified the instance in AWS |
| 7 | Made a change and observed Spacelift's plan/apply workflow |

You now have a working pattern for managing AWS infrastructure through Spacelift — the same approach scales to VPCs, EKS clusters, RDS databases, and full multi-environment architectures as your needs grow.
