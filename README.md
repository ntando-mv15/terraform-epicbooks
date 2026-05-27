# EpicBook — AWS Deployment with Terraform

A full-stack Infrastructure as Code (IaC) project that provisions AWS infrastructure using Terraform and deploys the EpicBook web application on an EC2 instance backed by an Amazon RDS MySQL database.

---

## Table of Contents

- [EpicBook — AWS Deployment with Terraform](#epicbook--aws-deployment-with-terraform)
  - [Table of Contents](#table-of-contents)
  - [Project Overview](#project-overview)
  - [Architecture](#architecture)
  - [Prerequisites](#prerequisites)
  - [Project Structure](#project-structure)
  - [Infrastructure Provisioned](#infrastructure-provisioned)
  - [Step-by-Step Deployment Guide](#step-by-step-deployment-guide)
    - [Step 1: Clone This Repo](#step-1-clone-this-repo)
    - [Step 2: Configure AWS Credentials](#step-2-configure-aws-credentials)
    - [Step 3: Set Terraform Variables](#step-3-set-terraform-variables)
    - [Step 4: Initialize Terraform](#step-4-initialize-terraform)
    - [Step 5: Plan and Apply](#step-5-plan-and-apply)
    - [Step 6: SSH into the EC2 Instance](#step-6-ssh-into-the-ec2-instance)
    - [Step 7: Test the Deployment](#step-7-test-the-deployment)
  - [Clean Up Resources](#clean-up-resources)
  - [Key Learnings](#key-learnings)

---

## Project Overview

This project demonstrates how to:

- Use **Terraform** to provision a production-style AWS environment
- Design a **public/private subnet architecture** within a custom VPC
- Deploy an **Ubuntu 22.04 EC2** instance with a `user_data` bootstrap script
- Provision a **managed MySQL database** with Amazon RDS in a private subnet
- Enforce **least-privilege networking**: RDS is only reachable from the EC2 security group, never the public internet
- Dynamically detect the operator's IP at apply time to restrict SSH access

> 🔗 App Source: [pravinmishraaws/theepicbook](https://github.com/pravinmishraaws/theepicbook)
> 📖 For application documentation and installation guide, see the [`docs/`](./docs/) folder.

---

## Architecture

```
Internet
    │
    ▼
┌─────────────────────────────────────────────────────┐
│                    AWS VPC                          │
│                  10.0.0.0/16                        │
│                                                     │
│  ┌──────────────────────────┐                       │
│  │     Public Subnet        │                       │
│  │      10.0.1.0/24         │                       │
│  │                          │                       │
│  │  ┌────────────────────┐  │                       │
│  │  │   EC2 (Ubuntu      │  │◄── SSH (your IP only) │
│  │  │   22.04, t2.micro) │  │◄── HTTP :80           │
│  │  │                    │  │                       │
│  │  │   EpicBook App     │  │◄── :8080              │
│  │  └────────┬───────────┘  │                       │
│  └───────────┼──────────────┘                       │
│              │ MySQL :3306 (internal only)           │
│  ┌───────────┼──────────────────────────┐           │
│  │     Private Subnets                  │           │
│  │   10.0.2.0/24   10.0.3.0/24         │           │
│  │   (us-east-1a)  (us-east-1b)        │           │
│  │                                      │           │
│  │  ┌────────────────────────────────┐  │           │
│  │  │   RDS MySQL (db.t3.micro)      │  │           │
│  │  │   privately accessible only    │  │           │
│  │  └────────────────────────────────┘  │           │
│  └──────────────────────────────────────┘           │
│                                                     │
│  Internet Gateway ──► Public Route Table            │
└─────────────────────────────────────────────────────┘
```

---

## Prerequisites

| Tool | Version | Install Link |
|------|---------|--------------|
| Terraform | >= 1.3.0 | [terraform.io](https://developer.hashicorp.com/terraform/install) |
| AWS CLI | >= 2.0 | [docs.aws.amazon.com](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) |
| An AWS Account | — | [aws.amazon.com](https://aws.amazon.com/free/) |
| An EC2 Key Pair | — | Created in the AWS Console under EC2 → Key Pairs |
| SSH client | Any | Built into Linux/macOS; [PuTTY](https://putty.org/) for Windows |

---

## Project Structure

```
terraform-theepicbook/
├── public/                        # Static frontend assets
├── src/                           # Application source code
├── terraform/
│   ├── main.tf                    # All AWS resource definitions (VPC, EC2, RDS, SGs)
│   ├── outputs.tf                 # Outputs — EC2 public IP and RDS endpoint
│   ├── variables.tf               # Input variable declarations
│   ├── terraform.tfvars.example   # Example variable values (safe to commit)
│   └── user_data.sh               # EC2 bootstrap script — installs Node, Nginx, app
├── .gitignore                     # Excludes node_modules, .tfstate, secrets
├── package.json                   # App dependencies and scripts
└── README.md                      
```

> ⚠️ Never commit `terraform.tfvars`, it contains real credentials. It is excluded via `.gitignore`.

---

## Infrastructure Provisioned

| Resource | Name | Purpose |
|----------|------|---------|
| VPC | `epicbook-vpc` | Isolated network (10.0.0.0/16) |
| Public Subnet | `epicbook-public-subnet` | Hosts the EC2 instance (10.0.1.0/24) |
| Private Subnet | `epicbook-private-subnet` | RDS primary AZ, us-east-1a (10.0.2.0/24) |
| Private Subnet 2 | `epicbook-private-subnet-2` | RDS secondary AZ, us-east-1b (required by RDS subnet group) |
| Internet Gateway | `epicbook-igw` | Allows public subnet to reach the internet |
| Route Table | `epicbook-public-rt` | Routes 0.0.0.0/0 traffic to the IGW |
| EC2 Security Group | `epicbook-ec2-sg` | Allows SSH (your IP only), HTTP, HTTPS, 8080 |
| RDS Security Group | `epicbook-rds-sg` | Allows MySQL :3306 from EC2 SG only |
| EC2 Instance | `epicbook-ec2` | Ubuntu 22.04, t2.micro, bootstrapped via user_data |
| RDS Subnet Group | `epicbook-db-subnet-group` | Spans both private subnets for RDS HA |
| RDS MySQL Instance | `epicbook-rds` | MySQL, db.t3.micro, private subnet, not publicly accessible |

---

## Step-by-Step Deployment Guide

### Step 1: Clone This Repo

```bash
git clone https://github.com/ntando-mv15/terraform-theepicbook.git
cd terraform-theepicbook
```

### Step 2: Configure AWS Credentials

```bash
aws configure
```

Enter your AWS Access Key ID, Secret Access Key, default region (`us-east-1`), and output format (`json`).

Verify it works:

```bash
aws sts get-caller-identity
```

### Step 3: Set Terraform Variables

Copy the example file and fill in your values:

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars`:

```hcl
vpc_cidr             = "10.0.0.0/16"
public_subnet_cidr   = "10.0.1.0/24"
private_subnet_cidr  = "10.0.2.0/24"
private_subnet2_cidr = "10.0.3.0/24"
instance_type        = "t2.micro"
key_name             = "<your-ec2-key-pair-name>"
db_name              = "epicbook"
db_username          = "admin"
db_password          = "<your-secure-password>"
```

> 💡 `key_name` must match the name of an existing key pair in your AWS account (EC2 → Key Pairs in the console).

### Step 4: Initialize Terraform

```bash
terraform init
```

### Step 5: Plan and Apply

```bash
terraform plan
terraform apply
```

Type `yes` when prompted. Provisioning takes **5–10 minutes**, RDS takes the longest.

When complete, Terraform prints:

```
Outputs:

ec2_public_ip = "<your-ec2-ip>"
rds_endpoint  = "<your-rds-host>:3306"
```

### Step 6: SSH into the EC2 Instance

```bash
ssh -i ~/.ssh/<your-key.pem> ubuntu@<ec2_public_ip>
```

> The default Ubuntu 22.04 AMI username is `ubuntu`.

The `user_data` script runs automatically on first boot. It does the following:

1. Installs Node.js, npm, Git, Nginx, and the MySQL client
2. Clones the app and runs `npm install`
3. Writes the database config from Terraform variables
4. Waits for RDS to be ready (retries every 10s)
5. Seeds the database with schema and seed SQL files
6. Starts the app on port 8080 with pm2 and enables auto-restart
7. Configures Nginx as a reverse proxy from port 80 → 8080

Allow 2–3 minutes for it to complete before SSHing in.


### Step 7: Test the Deployment

Open a browser and navigate to:

```
http://<ec2_public_ip>
```

You should see the EpicBook application homepage.

---

## Clean Up Resources

To avoid ongoing AWS charges:

```bash
terraform destroy
```

Type `yes` to confirm. This removes all provisioned infrastructure including the RDS instance, EC2, VPC, and associated networking.

---

## Key Learnings

- **`terraform.tfvars`** separates secrets from code; `.gitignore` keeps it out of version control
- Terraform `outputs` surface the values you need immediately after apply, no hunting through the AWS console
- **Public/private subnet separation** is a core security pattern: the database is never exposed to the internet
- **RDS requires a DB subnet group spanning two AZs**, even for a single-AZ instance, which is why a second private subnet is needed
- **`user_data`** bootstraps EC2 on first boot, eliminating the need to SSH in just to install dependencies
- **Dynamic IP detection** (`data "http" "my_ip"`) locks SSH access to the operator's current IP at apply time, a simple but effective security control


---
