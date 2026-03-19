# Multi-Tenant Application Platform (EKS) - Documentation

## Overview
This platform supports 20+ teams and scales to 50+ teams using AWS EKS with strong multi-tenancy, security, and Terraform-based infrastructure.

---

# 1. Setup and Deployment Instructions

## Prerequisites
- AWS account
- Terraform (~>1.6)
- kubectl
- AWS CLI or EC2 IAM role
- Git

## Steps

### Clone Repo
git clone <repo>
cd <repo>

### Terraform Init
cd terraform/envs/stage
terraform init -backend-config=backend.hcl

### Plan & Apply
terraform plan
terraform apply

### Configure kubectl
aws eks update-kubeconfig --region eu-central-1 --name <cluster>

### Deploy Kubernetes
kubectl apply -f kubernetes/

---

# 2. Design Decisions

- EKS for managed Kubernetes
- Namespace-per-team for isolation
- Terraform for IaC
- S3 backend for state
- IRSA for secure AWS access
- Separate infra & k8s layers

---

# 3. Security Best Practices

- No static credentials
- IAM roles + IRSA
- Private subnets
- Security Groups + NetworkPolicies
- Secrets Manager
- Resource quotas
- Kyverno

---

# 4. Multi-Tenancy Isolation

- Namespace isolation
- RBAC per team
- Default deny NetworkPolicies
- Resource quotas
- IRSA for AWS isolation

---

# 5. Production Security

- Encryption at rest
- Private networking
- Least privilege IAM
- No public DB access

---

# 6. Cost Optimization

- Shared cluster
- Autoscaling
- S3 lifecycle
- Right-sized nodes

---

# 7. Architecture

Internet → ALB → EKS → Namespaces → Pods → RDS/S3

---

# 8. Disaster Recovery

- RDS backups
- S3 versioning


---

# Summary

- Secure multi-tenancy
- Scalable architecture
- Production-ready design
