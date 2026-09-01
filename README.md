# Platform Launchpad GitOps

Declarative Kubernetes desired state for the **Platform Launchpad** application.

This repository is the runtime source of truth for Platform Launchpad workloads deployed to Amazon EKS and reconciled by Argo CD.

It intentionally separates application delivery from application deployment:

- **Jenkins** validates, tests, builds, scans, and publishes application artifacts.
- **Amazon ECR** stores immutable container images.
- **This repository** records the approved Kubernetes desired state.
- **Argo CD** continuously reconciles that desired state into Amazon EKS.

Jenkins does **not** deploy application workloads directly to Kubernetes.

---

## Architecture

```text
┌─────────────────────────────┐
│ Platform Launchpad          │
│ Application Repository      │
│                             │
│ Backend / Worker / Frontend │
└──────────────┬──────────────┘
               │
               │ source commit
               ▼
┌─────────────────────────────┐
│ Jenkins CI                  │
│                             │
│ • Validate                  │
│ • Test                      │
│ • Build                     │
│ • Scan                      │
│ • Publish                   │
└──────────────┬──────────────┘
               │
               │ container images
               ▼
┌─────────────────────────────┐
│ Amazon ECR                  │
│                             │
│ Immutable application       │
│ artifacts                   │
└──────────────┬──────────────┘
               │
               │ SHA-256 image digests
               ▼
┌─────────────────────────────┐
│ Platform Launchpad GitOps   │
│                             │
│ Kubernetes desired state    │
│ Environment overlays        │
│ Immutable image digests     │
└──────────────┬──────────────┘
               │
               │ Git reconciliation
               ▼
┌─────────────────────────────┐
│ Argo CD                     │
│                             │
│ Drift detection             │
│ Continuous reconciliation   │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│ Amazon EKS                  │
│                             │
│ Platform Launchpad runtime  │
└─────────────────────────────┘
```

---

## Repository Responsibilities

This repository owns the declarative Kubernetes runtime state for Platform Launchpad.

### This Repository Owns

- Kubernetes application manifests
- Kubernetes Services
- Kubernetes ServiceAccounts
- Application Ingress configuration
- Environment overlays
- Immutable container image digests
- Runtime Kubernetes configuration
- Secrets Store CSI `SecretProviderClass` resources
- Argo CD Application definitions
- Environment-specific desired state
- Git-based deployment and rollback history

### This Repository Does Not Own

- Application source code
- Backend or frontend build logic
- Container image builds
- Jenkins CI pipeline implementation
- AWS infrastructure provisioning
- Amazon ECR repositories
- Amazon EKS infrastructure
- Amazon RDS
- Redis infrastructure
- IAM roles or policies
- EKS Pod Identity associations
- AWS Secrets Manager secret values
- Plaintext application credentials

AWS infrastructure remains managed through Terraform in the Platform Launchpad application/infrastructure repository.

---

## GitOps Delivery Model

Platform Launchpad uses a pull-based GitOps delivery model.

```text
Developer
    |
    | Git push / merge
    v
Application Repository
    |
    v
Jenkins
    |
    +--> Backend tests
    |
    +--> Frontend validation
    |
    +--> Build backend/worker image
    |
    +--> Build frontend image
    |
    +--> Container vulnerability scanning
    |
    +--> Assume least-privilege AWS CI role
    |
    +--> Publish images to Amazon ECR
    |
    +--> Capture immutable image digests
    |
    +--> Update approved GitOps overlay
             |
             v
       GitOps Repository
             |
             v
          Argo CD
             |
             v
         Amazon EKS
```

The CI system publishes artifacts and updates desired state.

The CI system does **not** execute direct Kubernetes deployment commands such as:

```text
kubectl apply
kubectl set image
helm upgrade
```

Argo CD is the deployment authority responsible for reconciling approved Git state into the Kubernetes cluster.

---

## Repository Structure

```text
platform-launchpad-gitops/
├── README.md
├── applications/
│
├── base/
│   ├── namespace.yaml
│   ├── ingress.yaml
│   ├── kustomization.yaml
│   │
│   ├── backend/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   │
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   │
│   ├── worker/
│   │   ├── deployment.yaml
│   │   └── kustomization.yaml
│   │
│   └── runtime/
│       ├── serviceaccounts.yaml
│       ├── secretproviderclass.yaml
│       └── kustomization.yaml
│
└── overlays/
    ├── development/
    │   ├── kustomization.yaml
    │   ├── ingress-patch.yaml
    │   └── secretproviderclass-patch.yaml
    │
    ├── staging/
    │
    └── production/
```

The `staging` and `production` overlays are reserved for future environment expansion after the corresponding infrastructure is implemented.

---

## Kubernetes Base

The `base/` directory contains reusable Kubernetes resources shared across environments.

The base intentionally avoids environment-specific container registry references.

Instead, workloads reference logical application image names.

### Backend

```yaml
image: platform-launchpad-backend
```

### Worker

```yaml
image: platform-launchpad-backend
```

### Frontend

```yaml
image: platform-launchpad-frontend
```

The backend and worker intentionally use the same backend application image.

This means a single backend image digest update can consistently promote both workloads.

---

## Environment Overlays

Environment-specific configuration is maintained under:

```text
overlays/
```

The current implemented environment is:

```text
overlays/development/
```

The development overlay controls:

- Backend/worker ECR image digest
- Frontend ECR image digest
- ECR repository mapping
- Application hostname
- AWS Secrets Manager runtime secret reference

This keeps the reusable Kubernetes base independent from a specific deployment environment.

---

## Development Environment

The development environment currently targets:

```text
AWS Region:
us-east-1
```

Application container repository:

```text
.dkr.ecr.us-east-1.amazonaws.com/platform-launchpad-development-application
```

Application hostname:

```text
launchpad.christineadelusi.com
```

Runtime secret reference:

```text
platform-launchpad/development/runtime
```

The Kubernetes manifests reference the external secret through the AWS Secrets Store CSI integration.

Secret values are never stored in this repository.

---

## Immutable Image Deployment

Runtime workloads are deployed using immutable container image digests rather than mutable deployment tags.

Conceptually:

```text
platform-launchpad-development-application@sha256:<digest>
```

The development overlay maintains separate artifact identities for:

```text
Backend / Worker
    |
    └── sha256:<backend-digest>

Frontend
    |
    └── sha256:<frontend-digest>
```

The backend digest is applied to both:

- Backend Deployment
- Worker Deployment

The frontend digest is applied independently to:

- Frontend Deployment

This ensures that the exact artifact approved by CI is the artifact reconciled by Argo CD.

---

## Why Digests Instead of Tags?

Container tags are useful for human-readable traceability, but tags can be mutable.

A digest identifies the exact immutable image content.

For example:

```text
backend-sha-a1b2c3d4e5f6
```

may identify a build in ECR, while Kubernetes ultimately consumes:

```text
.dkr.ecr.us-east-1.amazonaws.com/platform-launchpad-development-application@sha256:<digest>
```

This provides:

- Deterministic deployments
- Artifact immutability
- Reliable rollback
- Stronger auditability
- Protection against accidental tag reassignment

---

## CI/CD Responsibility Boundary

Platform Launchpad intentionally separates CI from deployment reconciliation.

### Jenkins Responsibilities

Jenkins is responsible for:

1. Checking out application source
2. Running backend validation
3. Running automated backend tests
4. Running frontend validation
5. Building the backend/worker container image
6. Building the frontend container image
7. Verifying built images
8. Scanning container images for vulnerabilities
9. Assuming the least-privilege CI delivery role
10. Authenticating to Amazon ECR
11. Publishing approved images
12. Capturing immutable ECR image digests
13. Updating approved GitOps desired state

### Argo CD Responsibilities

Argo CD is responsible for:

1. Monitoring this repository
2. Detecting desired-state changes
3. Comparing Git state with cluster state
4. Applying approved Kubernetes changes
5. Detecting runtime drift
6. Reconciling drift back toward Git
7. Reporting synchronization and application health

### Terraform Responsibilities

Terraform remains responsible for infrastructure such as:

- VPC networking
- Public and private subnets
- Security groups
- VPC endpoints
- Amazon EKS
- EKS node groups
- IAM
- EKS Pod Identity
- Amazon ECR
- Amazon RDS
- Redis
- AWS Secrets Manager
- AWS Load Balancer Controller infrastructure

This separation prevents multiple systems from competing for ownership of the same resources.

---

## Security Model

The delivery model follows least-privilege and separation-of-responsibilities principles.

### Jenkins

Jenkins receives only the permissions required to perform CI and artifact publication.

AWS access uses a dedicated CI delivery role rather than broad infrastructure credentials.

Jenkins does not require Kubernetes deployment credentials.

### Argo CD

Argo CD is responsible for Kubernetes reconciliation.

This prevents Jenkins from becoming both:

- The artifact build system
- The Kubernetes deployment authority

### Secrets

Plaintext secrets must never be committed to this repository.

Runtime secrets are stored externally in AWS Secrets Manager and exposed to workloads through Kubernetes Secrets Store CSI integration.

The GitOps repository contains only references to those external secrets.

---

## Runtime Secret Flow

```text
AWS Secrets Manager
        |
        | platform-launchpad/development/runtime
        v
EKS Pod Identity
        |
        v
Secrets Store CSI Driver
        |
        v
SecretProviderClass
        |
        +--> Backend
        |
        └--> Worker
```

The GitOps repository defines the Kubernetes reference.

Terraform defines and manages the AWS-side infrastructure and authorization.

---

## Ingress Routing

Platform Launchpad uses a shared Kubernetes Ingress.

The development hostname is:

```text
launchpad.christineadelusi.com
```

Application routing follows this model:

```text
/
    -> frontend:3000

/api
    -> backend:8000

/docs
    -> backend:8000

/redoc
    -> backend:8000

/openapi.json
    -> backend:8000

/health
    -> backend:8000
```

The AWS Load Balancer Controller translates the Kubernetes Ingress configuration into AWS Application Load Balancer resources.

---

## Local Manifest Validation

The reusable base can be rendered locally with:

```bash
kubectl kustomize base
```

The development environment can be rendered with:

```bash
kubectl kustomize overlays/development
```

Rendering does not require a connection to the Kubernetes cluster.

To inspect development image references:

```bash
kubectl kustomize overlays/development \
  | grep 'image:'
```

To inspect environment-specific configuration:

```bash
kubectl kustomize overlays/development \
  | grep -E 'host:|objectName:'
```

---

## Deployment

Application deployment is intentionally not performed manually from this repository.

The expected deployment lifecycle is:

```text
Application change
      |
      v
CI validation
      |
      v
Container build
      |
      v
Security scan
      |
      v
ECR publication
      |
      v
Immutable digest captured
      |
      v
GitOps desired state updated
      |
      v
Argo CD detects change
      |
      v
EKS reconciled
```

Git is therefore the authoritative record of what should be running.

---

## Drift Management

Manual changes made directly to Kubernetes are considered runtime drift.

Argo CD compares the live cluster against this repository and identifies differences between:

```text
Desired State
     |
     | compare
     v
Live Cluster State
```

Where automated reconciliation is enabled, Argo CD restores the approved Git state.

Permanent changes should therefore be made through Git rather than directly against the cluster.

---

## Rollback

Rollback is Git-driven.

A failed application release can be reverted by:

1. Reverting the GitOps commit that introduced the problematic image digest, or
2. Creating a corrective commit restoring a previously known-good digest.

Argo CD then reconciles the restored desired state.

```text
Bad Release
     |
     v
Git Revert
     |
     v
Previous Image Digest
     |
     v
Argo CD
     |
     v
Known-Good Runtime
```

This provides an auditable rollback path without requiring Jenkins to manipulate the cluster directly.

---

## Environment Promotion

The repository structure is designed to support controlled promotion through:

```text
development
     |
     v
staging
     |
     v
production
```

Promotion should move an already validated immutable image digest between environment overlays rather than rebuilding the application for each environment.

This ensures the artifact tested earlier in the delivery process is the same artifact promoted later.

Staging and production will be implemented when their corresponding infrastructure environments are introduced.

---

## Git as the Runtime Source of Truth

The Kubernetes cluster is not considered the authoritative source of application configuration.

Git is.

The intended relationship is:

```text
Git desired state
      |
      v
Argo CD reconciliation
      |
      v
Kubernetes runtime state
```

Operational changes that must persist should be represented as commits to this repository.

This provides:

- Version history
- Change review
- Auditability
- Reproducibility
- Drift detection
- Controlled rollback

---

## Design Principles

This repository follows several platform engineering principles:

### Declarative Delivery

Runtime state is expressed declaratively through Kubernetes manifests and Kustomize overlays.

### Separation of Concerns

Application development, infrastructure provisioning, CI, artifact storage, desired state, and runtime reconciliation have distinct responsibilities.

### Immutable Artifacts

Application deployments reference immutable SHA-256 container digests.

### Least Privilege

CI receives artifact publication permissions without broad Kubernetes deployment authority.

### Git-Based Auditability

Runtime changes are represented through Git commits.

### Reconciliation

Argo CD continuously compares desired and actual state.

### Reproducibility

Kustomize allows the complete environment configuration to be rendered from version-controlled files.

### Environment Promotion

The architecture supports promoting the same immutable artifact across development, staging, and production.

---

## Current Implementation Status

The GitOps repository currently includes:

- [x] Reusable Kubernetes base
- [x] Backend Deployment
- [x] Backend Service
- [x] Worker Deployment
- [x] Frontend Deployment
- [x] Frontend Service
- [x] Shared application Ingress
- [x] Backend and worker ServiceAccounts
- [x] Secrets Store CSI `SecretProviderClass`
- [x] Development Kustomize overlay
- [x] Environment-specific ingress configuration
- [x] Environment-specific Secrets Manager reference
- [x] Immutable image digest structure
- [x] Local Kustomize rendering validation
- [ ] Jenkins automated GitOps update
- [ ] Argo CD Application definition
- [ ] Argo CD reconciliation validation
- [ ] End-to-end GitOps deployment
- [ ] Staging environment
- [ ] Production environment

---

## Related Platform Launchpad Components

Platform Launchpad is designed as a production-style platform engineering project demonstrating:

- AWS infrastructure as code
- Amazon EKS
- Kubernetes
- Terraform
- Jenkins CI
- GitOps
- Argo CD
- Docker
- Amazon ECR
- FastAPI
- Next.js
- PostgreSQL
- Redis
- AWS Secrets Manager
- EKS Pod Identity
- Secrets Store CSI
- Application Load Balancing
- Container vulnerability scanning
- Immutable artifact delivery
- Environment promotion
- Drift detection
- Git-based rollback

---

## Project Goal

Platform Launchpad demonstrates a secure self-service application delivery platform where application teams can request and manage deployments while the underlying platform enforces consistent infrastructure, security, CI/CD, and GitOps practices.

The GitOps repository provides the declarative runtime boundary between validated application artifacts and the Kubernetes platform responsible for running them.
