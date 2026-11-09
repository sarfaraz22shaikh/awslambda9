# Terraform + AWS: EC2 Web Server and IAM User Management

Tested flow on Ubuntu (WSL/VM) with an AWS free-plan account in **ap-southeast-2 (Sydney)**.

## Contents

1. [Prerequisites](#1-prerequisites)
2. [Create IAM user and access key](#2-create-iam-user-and-access-key)
3. [Install Terraform](#3-install-terraform)
4. [Install AWS CLI v2](#4-install-aws-cli-v2)
5. [Configure and verify AWS CLI](#5-configure-and-verify-aws-cli)
6. [Task 1: Ubuntu EC2 web server with custom page](#6-task-1-ubuntu-ec2-web-server-with-custom-page)
7. [Task 2: Create IAM user `cdac`, delete IAM user `ditiss`](#7-task-2-create-iam-user-cdac-delete-iam-user-ditiss)
8. [Troubleshooting](#8-troubleshooting)
9. [Cleanup and security](#9-cleanup-and-security)

---

## 1. Prerequisites

- Ubuntu machine with internet access and `sudo`
- An AWS account (free plan is fine)
- Use region **ap-southeast-2**. On free-plan accounts other regions can be blocked by an organization SCP, which causes `UnauthorizedOperation ... explicit deny in a service control policy`.

---

## 2. Create IAM user and access key

Do this in the AWS Console (sign in as root or an admin).

**Create the user**

1. Console → **IAM** → **Users** → **Create user**
2. User name: `terraform_admin` (any name) → **Next**
3. Permissions options: **Attach policies directly**
4. Search `AdministratorAccess` and tick the one with type **AWS managed - job function**
   (not `AdministratorAccess-Amplify` or `AdministratorAccess-AWSElasticBeanstalk`)
5. **Next** → **Create user**

> `AdministratorAccess` is for practice only. It avoids permission errors during learning.

**Create the access key**

1. IAM → Users → click the new user → **Security credentials** tab
2. **Create access key** → use case **Command Line Interface (CLI)**
3. Tick the confirmation box → **Next** → **Create access key**
4. Copy the **Access key ID** and **Secret access key**, or click **Download .csv file**.
   The secret is shown only once.

---

## 3. Install Terraform

```bash
sudo apt update
sudo apt install -y gnupg curl unzip lsb-release

curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor --yes -o /usr/share/keyrings/hashicorp.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com noble main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update
sudo apt install -y terraform
terraform -version
```

`noble` is hardcoded because newer Ubuntu codenames are often missing from HashiCorp's repo.

---

## 4. Install AWS CLI v2

```bash
cd /tmp
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip -o awscliv2.zip
sudo ./aws/install
cd ~
aws --version
```

If it reports already installed: `sudo ./aws/install --update`

---

## 5. Configure and verify AWS CLI

```bash
aws configure
```

Enter:

```
AWS Access Key ID:      <your access key id>
AWS Secret Access Key:  <your secret access key>
Default region name:    ap-southeast-2
Default output format:  json
```

Verify:

```bash
aws sts get-caller-identity
aws ec2 describe-vpcs --region ap-southeast-2
```

- `get-caller-identity` must show your IAM user ARN.
- `describe-vpcs` must return JSON. If it returns `UnauthorizedOperation`, see [Troubleshooting](#8-troubleshooting) before continuing.

---

## 6. Task 1: Ubuntu EC2 web server with custom page

### What the code does

| Block | Purpose |
|---|---|
| `provider` | Selects AWS and the region |
| `data "aws_ami"` | Finds the latest Ubuntu 24.04 image from Canonical |
| `aws_security_group` | Allows inbound HTTP (80), all outbound |
| `aws_instance` | Creates the server; `user_data` installs Apache and writes your page |
| `output` | Prints the public IP and website URL |

### Steps

```bash
mkdir ~/tf-ec2-web && cd ~/tf-ec2-web
vim main.tf
```

In vim: press `i`, paste the code, press `Esc`, type `:wq`, press Enter.

### `main.tf`

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
  region = "ap-southeast-2"
}

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"]
  }
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_security_group" "web_sg" {
  name        = "tf-web-sg"
  description = "Allow HTTP"

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

resource "aws_instance" "web" {
  ami                         = data.aws_ami.ubuntu.id
  instance_type               = "t3.micro"
  vpc_security_group_ids      = [aws_security_group.web_sg.id]
  user_data_replace_on_change = true

  user_data = <<-EOT
    #!/bin/bash
    apt-get update -y
    apt-get install -y apache2
    systemctl enable --now apache2

    cat > /var/www/html/index.html <<'HTML'
    <h1>Welcome to DITISS</h1>
    <h2>Name: YOUR NAME</h2>
    <h2>PRN: YOUR PRN</h2>
    HTML
  EOT

  tags = {
    Name = "Terraform-Ubuntu-EC2"
  }
}

output "public_ip" {
  value = aws_instance.web.public_ip
}

output "website_url" {
  value = "http://${aws_instance.web.public_ip}"
}
```

Replace `YOUR NAME` and `YOUR PRN` (or any content you want to show).

### Notes on key lines

- `owners = ["099720109477"]`: Canonical's account, so the AMI is official Ubuntu.
- `egress` rule: Terraform removes AWS's default allow-all outbound rule. Without this, `apt-get` cannot download Apache.
- `user_data_replace_on_change = true`: `user_data` runs only on first boot. This recreates the instance when you edit the script so new content appears.

### Run

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply          # type: yes
```

Wait about 60 seconds for Apache to install, then:

```bash
curl $(terraform output -raw public_ip)
```

Or open `website_url` in a browser (use `http://`, not `https://`).

### Change the content later

Edit the HTML in `main.tf`, then:

```bash
terraform apply
```

The instance is recreated and gets a new public IP.

### Minimal version

```hcl
provider "aws" {
  region = "ap-southeast-2"
}

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*"]
  }
}

resource "aws_security_group" "sg" {
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

resource "aws_instance" "web" {
  ami                         = data.aws_ami.ubuntu.id
  instance_type               = "t3.micro"
  vpc_security_group_ids      = [aws_security_group.sg.id]
  user_data_replace_on_change = true
  user_data                   = <<-EOT
    #!/bin/bash
    apt-get update -y && apt-get install -y apache2
    echo "<h1>Hello, YOUR NAME</h1>" > /var/www/html/index.html
  EOT
}

output "url" {
  value = "http://${aws_instance.web.public_ip}"
}
```

---

## 7. Task 2: Create IAM user `cdac`, delete IAM user `ditiss`

Terraform can only delete resources it manages, so the flow is: create `cdac`, **import** `ditiss`, then remove it from the code and apply.

### Step 1: project

```bash
mkdir ~/tf-iam && cd ~/tf-iam
```

### Step 2: create `ditiss` (only if it does not exist)

```bash
aws iam create-user --user-name ditiss
```

### Step 3: `main.tf`

```hcl
provider "aws" {
  region = "ap-southeast-2"
}

resource "aws_iam_user" "cdac" {
  name = "cdac"
}

resource "aws_iam_user" "ditiss" {
  name          = "ditiss"
  force_destroy = false
}
```

> Keep `force_destroy = false`. With `true`, Terraform calls `iam:DeleteLoginProfile` during deletion, which an organization SCP can deny (`AccessDenied ... explicit deny in a service control policy`).

### Step 4: init, import, apply

```bash
terraform init
terraform import aws_iam_user.ditiss ditiss
terraform plan
terraform apply          # type: yes
```

This creates `cdac` and brings `ditiss` under Terraform management.

### Step 5: delete `ditiss`

Remove the `ditiss` block from `main.tf`, leaving:

```hcl
provider "aws" {
  region = "ap-southeast-2"
}

resource "aws_iam_user" "cdac" {
  name = "cdac"
}
```

Then:

```bash
terraform plan           # shows: 1 to destroy
terraform apply          # type: yes
```

### Step 6: verify

```bash
aws iam get-user --user-name cdac
aws iam list-users --query "Users[].UserName"
```

`cdac` is listed and `ditiss` is gone.

### If deletion fails with `DeleteConflict`

The user still has access keys or a login profile:

```bash
aws iam list-access-keys --user-name ditiss
aws iam delete-access-key --user-name ditiss --access-key-id <ACCESS_KEY_ID>
terraform apply
```

---

## 8. Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `UnauthorizedOperation ... explicit deny in a service control policy` on EC2 actions | Region not allowed for the account (free plan) or account restricted by an organization SCP | Use `ap-southeast-2`. If still denied, upgrade the account plan or contact the organization owner |
| Same error in `ap-south-1` / `us-east-1` | Those regions are blocked on the account | Switch region in `aws configure` and in `provider "aws"` |
| `sts:DecodeAuthorizationMessage` denied | Same SCP | Ignore; the original error already shows the denied action |
| `AccessDenied ... iam:DeleteLoginProfile ... explicit deny` | `force_destroy = true` on `aws_iam_user` | Set `force_destroy = false`, apply, then delete |
| `apt update` fails for the HashiCorp repo | Ubuntu codename not supported | Use `noble` in the repo line (as above) |
| Page does not load after apply | Apache still installing, or `https://` used | Wait 60 seconds, use `http://` |
| `apt-get` fails inside the instance | Missing egress rule | Keep the `egress` block in the security group |
| `<encoded-string>: No such file or directory` | Placeholder typed literally in a command | Replace placeholders with real values in single quotes |
| `aws: command not found` after install | Shell not refreshed | Run `hash -r` or open a new terminal |

Useful checks:

```bash
aws sts get-caller-identity
aws configure get region
aws ec2 describe-vpcs --region ap-southeast-2
terraform validate
```

---

## 9. Cleanup and security

Destroy resources when done so nothing keeps billing:

```bash
cd ~/tf-ec2-web && terraform destroy       # type: yes
```

Security rules:

- Never paste your **secret access key** into chats, screenshots, or Git repositories.
- Do not commit `terraform.tfstate`, `terraform.tfstate.backup` or `.terraform/` to Git. Add them to `.gitignore`.
- Delete unused access keys: IAM → Users → user → Security credentials → access key → **Actions** → **Delete**.
- Prefer a narrower policy than `AdministratorAccess` outside practice environments.
- Set a budget alert: AWS Billing → Budgets.
