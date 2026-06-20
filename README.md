# Terraform Multicloud

> Production-ready Terraform modules for multi-account AWS and GCP infrastructure provisioning — covering account vending, EKS clusters, IAM, networking, and cloud governance patterns for enterprise environments.

![Terraform](https://img.shields.io/badge/Terraform-%3E%3D1.5-7B42BC?logo=terraform)
![AWS](https://img.shields.io/badge/AWS-Multi--Account-orange?logo=amazon-aws)
![GCP](https://img.shields.io/badge/GCP-Multi--Project-4285F4?logo=google-cloud)
![License](https://img.shields.io/badge/License-MIT-green)

---

## Overview

This repository provides a modular Terraform library for provisioning and managing enterprise-scale infrastructure across AWS and GCP. Designed for platform engineering and cloud infrastructure teams, each module is independently versioned, composable, and production-tested.

Core focus areas:
- **Account / Project vending** — automated bootstrapping of new AWS accounts and GCP projects with baseline guardrails
- **EKS clusters** — opinionated multi-AZ Kubernetes clusters with managed node groups, IRSA, and add-ons
- **IAM** — least-privilege role patterns, SCPs, and federated identity for both clouds
- **Networking** — hub-and-spoke VPCs, Transit Gateway, VPC peering, and GCP Shared VPC
- **Governance** — policy-as-code controls, config rules, and org-level guardrails

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         AWS Organizations                            │
│                                                                      │
│  ┌────────────────┐   ┌─────────────────┐   ┌────────────────────┐  │
│  │  Management    │   │  Network Hub    │   │  Shared Services   │  │
│  │  Account       │   │  Account        │   │  Account           │  │
│  │  (root / SCPs) │   │  (TGW / VPCs)   │   │  (tooling / CICD)  │  │
│  └────────────────┘   └────────┬────────┘   └────────────────────┘  │
│                                │ Transit Gateway                     │
│          ┌─────────────────────┼──────────────────────┐             │
│          │                     │                      │             │
│  ┌───────▼──────┐   ┌──────────▼──────┐   ┌──────────▼──────┐     │
│  │  Dev Account │   │ Staging Account │   │  Prod Account   │     │
│  │  EKS / RDS   │   │  EKS / RDS      │   │  EKS / RDS      │     │
│  └──────────────┘   └─────────────────┘   └─────────────────┘     │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                         GCP Organization                             │
│                                                                      │
│  ┌────────────────┐   ┌──────────────────────────────────────────┐  │
│  │  Org Policies  │   │  Host Project (Shared VPC)               │  │
│  │  + Folders     │   │  └── Service Projects (dev / prod)       │  │
│  └────────────────┘   └──────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Module Index

| Module | Cloud | Description |
|---|---|---|
| `aws/account-vending` | AWS | Baseline account setup, Config, CloudTrail, budget alerts |
| `aws/networking/vpc` | AWS | Multi-AZ VPC with public/private/intra subnets |
| `aws/networking/transit-gateway` | AWS | TGW with RAM sharing and route table management |
| `aws/eks` | AWS | EKS cluster with managed node groups, IRSA, Karpenter |
| `aws/iam/roles` | AWS | Cross-account roles, IRSA patterns, SCPs |
| `aws/iam/scp` | AWS | Organization-level Service Control Policies |
| `gcp/project-vending` | GCP | Project creation with billing, IAM, and API enablement |
| `gcp/networking/shared-vpc` | GCP | Shared VPC host and service project setup |
| `gcp/iam/workload-identity` | GCP | Workload Identity Federation for OIDC/AWS cross-cloud |
| `gcp/gke` | GCP | GKE Autopilot and Standard cluster modules |
| `common/tagging` | Both | Tagging standards and enforcement helpers |
| `common/dns` | Both | Route53 / Cloud DNS with cross-cloud resolution |

---

## Repository Structure

```
terraform-multicloud/
├── modules/
│   ├── aws/
│   │   ├── account-vending/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── README.md
│   │   ├── eks/
│   │   │   ├── main.tf
│   │   │   ├── node_groups.tf
│   │   │   ├── irsa.tf
│   │   │   ├── addons.tf
│   │   │   └── variables.tf
│   │   ├── networking/
│   │   │   ├── vpc/
│   │   │   └── transit-gateway/
│   │   └── iam/
│   │       ├── roles/
│   │       └── scp/
│   └── gcp/
│       ├── project-vending/
│       ├── gke/
│       ├── networking/
│       │   └── shared-vpc/
│       └── iam/
│           └── workload-identity/
├── examples/
│   ├── aws-landing-zone/
│   ├── aws-eks-cluster/
│   ├── gcp-project-baseline/
│   └── multicloud-connectivity/
├── tests/
│   └── terratest/
└── docs/
    ├── account-vending.md
    ├── eks-patterns.md
    └── multicloud-networking.md
```

---

## Prerequisites

| Tool | Version |
|---|---|
| Terraform | >= 1.5 |
| AWS CLI | >= 2.x |
| `gcloud` CLI | >= 450.x |
| AWS Organizations | Enabled |
| GCP Organization | Configured |

### Provider Configuration

```hcl
# providers.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
  assume_role {
    role_arn = "arn:aws:iam::${var.management_account_id}:role/TerraformOrchestrator"
  }
}

provider "google" {
  project = var.gcp_project_id
  region  = var.gcp_region
}
```

---

## Usage

### AWS Account Vending

```hcl
module "dev_account" {
  source = "./modules/aws/account-vending"

  account_name      = "my-org-dev"
  account_email     = "aws-dev@example.com"
  organizational_unit = "ou-prod-workloads"
  
  baseline_config = {
    enable_cloudtrail    = true
    enable_config        = true
    enable_guardduty     = true
    enable_security_hub  = true
    monthly_budget_usd   = 5000
  }

  tags = {
    environment = "dev"
    team        = "platform-engineering"
    cost-center = "CC-1042"
  }
}
```

### EKS Cluster

```hcl
module "eks_prod" {
  source = "./modules/aws/eks"

  cluster_name    = "prod-eks-us-east-1"
  cluster_version = "1.31"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnet_ids

  node_groups = {
    general = {
      instance_types = ["m6i.xlarge", "m6a.xlarge"]
      min_size       = 3
      max_size       = 20
      desired_size   = 6
      disk_size      = 100
    }
    compute = {
      instance_types = ["c6i.2xlarge"]
      min_size       = 0
      max_size       = 50
      desired_size   = 2
      taints = [{
        key    = "workload"
        value  = "compute"
        effect = "NO_SCHEDULE"
      }]
    }
  }

  enable_karpenter     = true
  enable_cluster_autoscaler = false
  enable_aws_lb_controller  = true
  enable_external_dns       = true

  tags = {
    environment = "prod"
    team        = "platform-engineering"
  }
}
```

### GCP Project Vending

```hcl
module "ml_project" {
  source = "./modules/gcp/project-vending"

  project_name    = "company-ml-platform"
  folder_id       = var.engineering_folder_id
  billing_account = var.billing_account_id

  enabled_apis = [
    "container.googleapis.com",
    "aiplatform.googleapis.com",
    "bigquery.googleapis.com",
    "storage.googleapis.com",
  ]

  iam_bindings = {
    "roles/editor" = ["group:ml-team@example.com"]
    "roles/viewer" = ["group:auditors@example.com"]
  }

  labels = {
    environment = "prod"
    team        = "ml-platform"
    cost-center = "cc-2099"
  }
}
```

### Multicloud Connectivity (AWS ↔ GCP)

```hcl
module "multicloud_vpn" {
  source = "./examples/multicloud-connectivity"

  aws_vpc_id         = module.vpc.vpc_id
  aws_region         = "us-east-1"
  gcp_network        = module.shared_vpc.network_name
  gcp_region         = "us-central1"
  shared_secret      = var.vpn_shared_secret
  bgp_aws_asn        = 64512
  bgp_gcp_asn        = 64513
}
```

---

## Governance Controls

### AWS Service Control Policies

Pre-built SCPs included:

| SCP | Purpose |
|---|---|
| `deny-root-user` | Blocks all root account API actions |
| `deny-regions` | Restricts workloads to approved regions |
| `require-mfa` | Enforces MFA for IAM console access |
| `deny-public-s3` | Blocks public S3 bucket ACLs and policies |
| `restrict-ec2-types` | Limits instance families to approved list |
| `deny-leave-org` | Prevents accounts from leaving the Org |

```hcl
module "scps" {
  source = "./modules/aws/iam/scp"

  organization_id = var.org_id
  policies        = ["deny-root-user", "deny-public-s3", "deny-regions"]
  target_ous      = [var.workload_ou_id]
}
```

---

## Testing

Terratest integration tests are provided for critical modules:

```bash
cd tests/terratest/
go test -v -run TestEKSModule -timeout 45m
go test -v -run TestVPCModule -timeout 15m
go test -v -run TestAccountVending -timeout 30m
```

---

## Remote State

State is managed per environment in S3 (AWS) and GCS (GCP):

```hcl
# AWS
terraform {
  backend "s3" {
    bucket         = "company-tfstate-us-east-1"
    key            = "prod/eks/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# GCP
terraform {
  backend "gcs" {
    bucket = "company-tfstate"
    prefix = "prod/gke"
  }
}
```

---

## Contributing

1. Fork the repository and create a feature branch
2. Ensure modules include `variables.tf`, `outputs.tf`, and a `README.md`
3. Add Terratest coverage for any new module
4. Follow the tagging standards defined in `modules/common/tagging`
5. Submit a PR with a description of the infrastructure change and blast radius

---

## License

MIT — see [LICENSE](LICENSE) for details.
