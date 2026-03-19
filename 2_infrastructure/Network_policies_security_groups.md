# Multi-Tenant Application Platform (EKS) - Architecture & Infrastructure

## Overview
This platform supports 20+ teams and scales to 50+ teams using AWS EKS with strong multi-tenancy, security, and Terraform-based infrastructure.

---

## Multi-tenancy Isolation Strategy

Each tenant gets:
- Dedicated namespace
- RBAC isolation
- ResourceQuota & LimitRange
- NetworkPolicies (default deny)
- IRSA for AWS access
- Secrets via AWS Secrets Manager

---

## Security Model

### AWS Layer
- Private subnets
- Security Groups
- Encryption (RDS, S3)
- IAM roles (no static keys)

### Kubernetes Layer
- Namespace isolation
- RBAC (least privilege)
- NetworkPolicies
- Resource controls

### CI/CD Layer
- No static secrets
- Immutable image tags
- Role-based access

---

## Scalability Approach

### Phase 1 (20+ Teams)
- Shared EKS cluster
- Namespace-per-team
- Shared services

### Phase 2 (50+ Teams)
- Multiple node groups
- Autoscaling (Karpenter)
- GitOps onboarding
- Dedicated infra for critical teams

---

## Terraform Remote State

- S3 backend
- Locking via .tflock
- IAM role-based access

Example:
bucket = "hrs-platform-tf-state"
key    = "stage/eks/terraform.tfstate"

---

## Storage

- RDS → relational data
- S3 → artifacts/logs
- ECR → container images

---

## RBAC & IAM

- Kubernetes RBAC for namespace isolation
- IAM roles for cluster, nodes, CI/CD
- IRSA for pod-level AWS access

---

## Networking

### Security Groups
- ALB → public
- EKS → internal
- RDS → restricted

### NetworkPolicies
- Default deny
- Allow only required traffic

---

## Summary

- Secure multi-tenancy
- Scalable architecture
- Terraform-based infra
- Production-ready design

