# Multi-tenancy Isolation Strategy & Security Model

## Overview

The platform is designed to support multiple engineering teams on a shared Amazon EKS cluster while ensuring strong isolation, security, and governance.

Each team is treated as a **separate tenant** and is isolated using a combination of:
- Kubernetes-native controls (namespaces, RBAC, NetworkPolicies)
- AWS security mechanisms (IAM, IRSA, Security Groups)
- CI/CD access controls

This layered approach ensures **secure multi-tenancy without sacrificing cost efficiency or scalability**.

---

# Tenant Model

Each tenant is assigned:

- a dedicated Kubernetes namespace
- namespace-scoped RBAC permissions
- dedicated service accounts
- `ResourceQuota` and `LimitRange`
- default-deny `NetworkPolicies`
- isolated access to secrets and AWS resources via IRSA

### Example Namespaces

- `team-a`
- `team-b`
- `team-c`

This model allows multiple teams to share a single EKS cluster while maintaining **logical and operational separation**.

---

# Isolation Mechanisms

## 1. Namespace Isolation

Namespaces act as the primary boundary for tenant separation:

- workloads, services, and configurations are isolated per namespace
- teams cannot access resources in other namespaces by default

---

## 2. RBAC (Role-Based Access Control)

Access is controlled using Kubernetes RBAC:

- users and CI/CD pipelines are restricted to their namespace
- no cross-namespace access unless explicitly granted

### Example

- Team A → access only to `team-a`
- Team B → cannot access Team A resources

This enforces **least privilege access** inside the cluster.

---

## 3. Network Isolation

Each namespace enforces **default-deny NetworkPolicies**:

- no pod-to-pod communication allowed by default
- only explicitly allowed traffic is permitted

### Allowed Flows

- ingress controller → application pods
- application → database / external APIs
- limited intra-namespace communication (if required)

This prevents **lateral movement between tenants**.

---

## 4. Resource Isolation

Each tenant namespace includes:

- CPU and memory quotas
- pod limits
- default resource requests and limits

### Benefits

- fair resource distribution
- prevention of noisy-neighbor issues
- improved cluster stability under load

---

## 5. IAM Isolation using IRSA

AWS access is controlled using **IAM Roles for Service Accounts (IRSA)**:

- each workload gets a dedicated IAM role
- permissions are scoped to specific AWS resources

### Examples

- access only a specific S3 bucket or prefix
- access only a specific Secrets Manager path

### Key Benefit

- no static AWS credentials inside the cluster
- fine-grained, workload-level AWS access control

---

## 6. Secrets Isolation

Secrets are managed using **AWS Secrets Manager**:

- secrets are not stored in Kubernetes manifests
- each tenant accesses only its own secrets
- access is controlled via IAM roles (IRSA)

This improves both **security and auditability**.

---

# Security Model

Security is enforced across three layers:

- AWS Infrastructure Layer
- Kubernetes Cluster Layer
- CI/CD Layer

---

## 1. AWS Layer Security

- worker nodes and databases run in **private subnets**
- **Security Groups** restrict network access
- **encryption at rest** enabled (RDS, S3)
- IAM roles used instead of static credentials
- IRSA for pod-level AWS access
- Secrets Manager for secure secret storage

---

## 2. Kubernetes Layer Security

- namespace-based tenant isolation
- RBAC enforcing least privilege
- default-deny NetworkPolicies
- ResourceQuota and LimitRange enforcement
- dedicated service accounts per workload
- restricted access to system namespaces

### Optional Enhancements

- admission controllers (Kyverno / OPA)
- pod security standards (non-root, restricted capabilities)

---

## 3. CI/CD Security

- protected branches and approval workflows
- immutable image tagging (no `latest`)
- deployment permissions scoped per environment
- no secrets stored in pipeline definitions
- IAM role-based access from CI/CD (no static keys)

---

# Why This Approach Works

This multi-layered strategy provides:

- strong tenant isolation
- reduced blast radius
- secure AWS access from workloads
- protection against lateral movement
- scalable onboarding model
- production-ready security posture

---

# Summary

The platform achieves secure multi-tenancy using a combination of:

- **Kubernetes isolation mechanisms** (namespaces, RBAC, NetworkPolicies)
- **AWS security controls** (IAM, IRSA, Security Groups)
- **CI/CD governance and access control**

This layered design ensures that multiple teams can safely share a single EKS cluster while maintaining **security, scalability, and operational efficiency**.