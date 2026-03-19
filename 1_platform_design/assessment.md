# Architecture Document

## 1. Overview

This document describes a multi-tenant application platform on AWS designed for 20+ engineering teams (250+ engineers) with a scaling path to 50+ teams. The platform is built around Amazon EKS, shared platform services, standardized CI/CD, and centralized observability.

The design goals are:
- enable fast onboarding for many teams
- provide strong tenant isolation in a shared platform
- maintain security and least privilege by default
- scale operationally and technically from 20 to 50+ teams
- keep costs efficient in the initial phase
- provide a clear path for stronger isolation later

## 2. High-Level Architecture

Reference diagram:
- `diagrams/multi_tenant_platform_architecture.png`

### Platform summary
The platform uses:
- Amazon VPC across 3 Availability Zones in eu-central-1
- public subnets for ALB / ingress and controlled internet-facing entry
- private subnets for EKS worker nodes and data services
- Amazon EKS as the shared multi-tenant container platform
- Amazon ECR for image storage
- Amazon RDS for shared platform metadata or application relational workloads
- Amazon S3 for artifact storage, backup exports, and shared object storage
- AWS Secrets Manager for secret management
- IAM Roles for Service Accounts (IRSA) for pod-level AWS access
- OpenTelemetry Collector for telemetry export
- New Relic for monitoring, dashboards, and alerting

### Request flow
1. Engineering teams push code through CI/CD.
2. CI/CD builds and pushes images into Amazon ECR.
3. Deployments are applied into the target team namespace in EKS.
4. External traffic enters through ALB and the ingress controller.
5. Applications access AWS services securely using IRSA and Secrets Manager.
6. Telemetry flows through OpenTelemetry Collector into New Relic.

## 3. Multi-Tenancy Isolation Strategy

In this platform, a tenant is an engineering team or application group using the shared platform.

### Tenant model
Each team receives:
- a dedicated Kubernetes namespace
- namespace-scoped RBAC
- dedicated service accounts
- ResourceQuota and LimitRange
- default-deny NetworkPolicies
- isolated secrets access
- team-level labels for observability and governance

Examples:
- `team-a`
- `team-b`
- `team-c`

### Isolation controls

#### Namespace isolation
Every team deploys into its own namespace. This creates the first logical boundary and separates workloads, services, and secrets.

#### RBAC isolation
Users and CI service accounts are restricted to their namespace. Team A cannot manage Team B’s workloads by default.

#### Network isolation
Each tenant namespace uses a default-deny NetworkPolicy. Only required traffic is explicitly allowed, such as:
- ingress controller to application pods
- application to approved platform services
- application to database endpoints if required

This prevents uncontrolled east-west communication across tenants.

#### Resource isolation
Each namespace gets:
- CPU quota
- memory quota
- pod count quota
- default requests and limits

This prevents noisy-neighbor issues and protects platform stability.

#### IAM isolation with IRSA
Applications access AWS services using service-account-bound IAM roles instead of static credentials. This allows:
- one tenant to read only its own secret
- one app to access only its own S3 bucket or prefix
- better auditability and least privilege

#### Secrets isolation
Secrets are stored in AWS Secrets Manager and exposed only to the correct workload or namespace. Secrets are never shared broadly across tenants.

### Isolation tiers for future growth
The platform starts with logical isolation and evolves to stronger isolation when needed:

- **Tier 1: Shared**
  - shared cluster
  - shared node groups
  - namespace-level isolation

- **Tier 2: Isolated node group**
  - dedicated node group for noisy or sensitive tenants
  - taints/tolerations and node selectors

- **Tier 3: Dedicated cluster**
  - for regulated, high-risk, or highly critical tenants

This allows cost-efficient operations initially, while preserving a clear path to stronger isolation later.

## 4. Security Model

The security model follows least privilege, default deny, and separation of duties.

### AWS layer
- worker nodes and databases run in private subnets
- security groups restrict application-to-database and north-south traffic
- S3 and RDS encryption at rest enabled
- IAM roles used instead of long-lived AWS keys
- IRSA provides pod-level AWS permissions
- Secrets Manager stores secrets outside manifests

### Kubernetes layer
- namespace-based tenant separation
- RBAC scoped per team
- NetworkPolicies for east-west control
- ResourceQuota and LimitRange for fairness and stability
- dedicated service accounts per workload
- restricted access to shared platform namespaces

### CI/CD layer
- protected branches
- immutable image tags
- controlled deployment permissions by environment
- audit trail through Git and pipeline logs
- no hardcoded secrets in pipeline definitions

### Security outcome
This design provides strong logical isolation for shared-cluster multi-tenancy and can be extended to stronger physical separation where needed.

## 5. Scalability Approach (20 → 50+ Teams)

### Phase 1: 20+ teams
Start with:
- one shared EKS cluster
- namespace per team
- shared ingress and shared observability stack
- common CI/CD templates
- autoscaling worker nodes

Why this works:
- lower initial cost
- simpler operations
- fast onboarding
- standardized controls

### Phase 2: growth to 50+ teams
Scale in multiple dimensions:

#### Compute scaling
- add more worker nodes
- use Cluster Autoscaler or Karpenter
- introduce workload-based node groups

Examples:
- general application workloads
- compute-heavy workloads
- sensitive workloads

#### Operational scaling
Standardize:
- namespace creation
- RBAC templates
- quota templates
- policy templates
- CI/CD onboarding
- observability labels and dashboards

This reduces manual work for the platform team.

#### Isolation scaling
Not every team needs its own cluster. Instead:
- standard teams stay on shared node groups
- noisy tenants move to dedicated node groups
- regulated or business-critical tenants move to dedicated clusters

### Long-term scaling model
The platform uses a shared-first strategy with selective isolation. This avoids over-engineering on day one while preserving a safe path to 50+ teams.

## 6. Cost Estimation (AWS eu-central-1)

This is a baseline estimate for a modest shared platform starting point.

### Assumptions
- 1 EKS cluster
- 3 worker nodes
- 1 NAT Gateway
- 1 ALB
- modest S3 usage
- 1 small PostgreSQL RDS instance
- observability collector running in cluster

### Estimated monthly baseline
- EKS control plane: ~73 USD/month
- worker nodes: ~250–450 USD/month
- NAT Gateway: ~35 USD/month plus data processing
- ALB: ~20–40 USD/month
- RDS: ~80–180 USD/month depending on HA choice
- S3: ~20–50 USD/month
- observability: variable, depends on telemetry volume

### Estimated total
**~500 to 900 USD/month** for a minimal but production-minded shared baseline.

### Cost optimization strategy
- start with one shared cluster instead of one cluster per team
- use autoscaling and right-sizing
- keep only sensitive tenants on isolated infrastructure
- use S3 lifecycle policies
- reduce unnecessary NAT traffic
- adopt savings plans when usage stabilizes
- upgrade EKS regularly to avoid extended-support pricing

## 7. Key Design Trade-Offs and Rationale

### Shared cluster vs dedicated cluster per team
**Chosen:** shared EKS cluster with namespace-based multi-tenancy

**Why:** much lower cost, easier onboarding, simpler operations

**Trade-off:** weaker isolation than one cluster per team

**Rationale:** for 20–50 internal teams, shared cluster plus strong namespace controls gives the best balance of cost and operational efficiency.

### EKS vs ECS
**Chosen:** EKS

**Why:** stronger namespace-based multi-tenancy, mature RBAC, NetworkPolicies, and better fit for platform engineering patterns

**Trade-off:** more operational complexity than ECS

**Rationale:** the assignment emphasizes multi-tenancy and platform controls, which are clearer to demonstrate on EKS.

### Shared node groups vs dedicated node groups
**Chosen:** start shared, then isolate selectively

**Why:** avoids early over-provisioning and reduces waste

**Trade-off:** shared pools can have noisy-neighbor effects

**Rationale:** stronger isolation should be introduced only where business or technical requirements justify it.

### Centralized observability vs per-team tooling
**Chosen:** centralized observability with tenant-aware labels

**Why:** lower operational overhead, consistent dashboards, easier platform-wide visibility

**Trade-off:** requires careful tagging and access design

**Rationale:** centralized telemetry is the most practical model for a shared internal platform.

### Shared data services vs per-tenant dedicated databases
**Chosen:** shared baseline with option for dedicated data services later

**Why:** lower cost and simpler initial operations

**Trade-off:** some teams may later require stronger isolation

**Rationale:** not every tenant needs dedicated data infrastructure on day one.

## 8. Disaster Recovery Considerations

To strengthen the design and address production-readiness:

- deploy across 3 Availability Zones in a single region
- keep worker nodes in private subnets
- enable backups for RDS
- version and lifecycle-manage S3 data
- keep infrastructure as code in Git
- use immutable images in ECR
- store secrets in AWS Secrets Manager
- document recovery runbooks for:
  - cluster recreation
  - namespace re-onboarding
  - secret restoration
  - database restore and failover

Future enhancements:
- Multi-AZ RDS for higher availability
- cross-region backup replication
- secondary-region disaster recovery strategy for critical tenants
- GitOps-based recovery of cluster state

## 9. Final Design Rationale

This architecture is designed to start simple, secure, and cost-efficient for 20+ teams while preserving a clean path to scale to 50+ teams. It uses shared infrastructure where practical, strong logical isolation by default, and selective physical isolation only when justified by performance, compliance, or business criticality.