# Architecture Guide

This document provides detailed architecture information for the aws-idp-gitops repository and how it integrates with the Internal Developer Platform.

## Table of Contents

- [Overview](#overview)
- [GitOps Architecture](#gitops-architecture)
- [Component Architecture](#component-architecture)
- [Multi-Tenancy Model](#multi-tenancy-model)
- [Security Architecture](#security-architecture)
- [Network Architecture](#network-architecture)
- [Integration Points](#integration-points)

---

## Overview

The aws-idp-gitops repository implements a declarative, Git-based approach to managing Kubernetes resources. It serves as the single source of truth for all platform and application configurations deployed to the EKS cluster.

### Design Principles

| Principle | Description | Implementation |
|-----------|-------------|----------------|
| **Declarative** | Desired state defined in Git | All resources defined as YAML manifests |
| **Immutable** | No manual changes to cluster | ArgoCD enforces Git as source of truth |
| **Auditable** | All changes tracked in Git history | Pull requests and commit logs provide audit trail |
| **Self-Healing** | Automatic drift detection and correction | ArgoCD continuously reconciles actual vs desired state |
| **Separation of Concerns** | Platform vs application configs | Platform in `/platform`, teams in `/teams` |

---

## GitOps Architecture

### ArgoCD Deployment Model

```mermaid
flowchart TB
    subgraph "GitOps Repository"
        MAIN[main branch]
        PLATFORM[platform/<br/>Platform Components]
        TEAMS[teams/<br/>Team Configs]
        CLUSTERS[clusters/<br/>Cluster Configs]
    end

    subgraph "ArgoCD Control Plane"
        ROOT[Root Application<br/>app-of-apps]
        PLATAPP[Platform ApplicationSet]
        TEAMAPP[Team ApplicationSet]
    end

    subgraph "Generated Applications"
        APP1[Platform Apps]
        APP2[Team Projects]
        APP3[Team Namespaces]
        APP4[Team Applications]
    end

    subgraph "Kubernetes Resources"
        POL[Policies]
        SEC[Secrets]
        NS[Namespaces]
        DEPLOY[Deployments]
    end

    MAIN --> ROOT
    ROOT --> PLATAPP
    ROOT --> TEAMAPP

    PLATAPP --> APP1
    TEAMAPP --> APP2
    TEAMAPP --> APP3
    TEAMAPP --> APP4

    APP1 --> POL
    APP1 --> SEC
    APP2 --> NS
    APP3 --> NS
    APP4 --> DEPLOY
```

### App of Apps Pattern

The repository uses the "App of Apps" pattern where a root ArgoCD Application manages other Applications and ApplicationSets:

```yaml
# Root Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: platform
  source:
    repoURL: https://github.com/fast-ish/aws-idp-gitops
    path: clusters/production
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Sync Waves

Resources are deployed in a specific order using sync waves:

| Wave | Components | Purpose |
|------|-----------|---------|
| **-5** | Namespaces | Create namespaces first |
| **-4** | CRDs | Install custom resource definitions |
| **-3** | RBAC | Set up permissions |
| **-2** | ConfigMaps, Secrets | Configuration data |
| **-1** | PVCs | Storage resources |
| **0** | Standard resources | Default deployment wave |
| **1** | Services | After deployments |
| **2** | Ingresses | After services |

---

## Component Architecture

### Platform Components Layer

```mermaid
flowchart TB
    subgraph "Core Platform"
        ARGOCD[ArgoCD<br/>GitOps Engine]
        KYVERNO[Kyverno<br/>Policy Engine]
        EXTSEC[External Secrets<br/>Secrets Sync]
    end

    subgraph "CI/CD Layer"
        WORKFLOWS[Argo Workflows<br/>Build & Deploy]
        EVENTS[Argo Events<br/>Event Triggers]
        ROLLOUTS[Argo Rollouts<br/>Progressive Delivery]
    end

    subgraph "Observability Layer"
        PROM[Prometheus<br/>Metrics]
        LOKI[Loki<br/>Logs]
        TEMPO[Tempo<br/>Traces]
        GRAFANA[Grafana<br/>Dashboards]
    end

    subgraph "Security & Compliance"
        TRIVY[Trivy Operator<br/>Vulnerability Scanning]
        FALCO[Falco<br/>Runtime Security]
        COST[OpenCost<br/>Cost Tracking]
    end

    subgraph "Operations"
        VELERO[Velero<br/>Backup]
        GOLDILOCKS[Goldilocks<br/>Resource Optimization]
        VPA[VPA<br/>Auto-scaling]
    end

    ARGOCD --> WORKFLOWS
    WORKFLOWS --> EVENTS
    EVENTS --> ROLLOUTS

    PROM --> GRAFANA
    LOKI --> GRAFANA
    TEMPO --> GRAFANA

    KYVERNO --> TRIVY
    TRIVY --> FALCO
```

### ApplicationSet Architecture

Three ApplicationSets manage team resources:

#### 1. Team Projects ApplicationSet

Creates ArgoCD AppProjects with RBAC:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: team-projects
spec:
  generators:
    - git:
        repoURL: https://github.com/fast-ish/aws-idp-gitops
        revision: main
        files:
          - path: "teams/*/team.yaml"
  template:
    metadata:
      name: '{{name}}-project'
    spec:
      project: platform
      source:
        repoURL: https://github.com/fast-ish/aws-idp-gitops
        path: platform/team-project
        helm:
          values: |
            {{values}}
```

#### 2. Team Namespaces ApplicationSet

Provisions namespace infrastructure:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: team-namespaces
spec:
  generators:
    - git:
        repoURL: https://github.com/fast-ish/aws-idp-gitops
        revision: main
        files:
          - path: "teams/*/team.yaml"
  template:
    metadata:
      name: '{{name}}-namespace'
    spec:
      project: platform
      source:
        repoURL: https://github.com/fast-ish/aws-idp-gitops
        path: platform/team-resources
        helm:
          values: |
            {{values}}
```

#### 3. Team Apps ApplicationSet

Deploys team applications:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: team-apps
spec:
  generators:
    - matrix:
        generators:
          - git:
              repoURL: https://github.com/fast-ish/aws-idp-gitops
              revision: main
              files:
                - path: "teams/*/team.yaml"
          - list:
              elements: '{{apps}}'
  template:
    metadata:
      name: '{{name}}-{{app.name}}'
    spec:
      project: 'team-{{name}}'
      source:
        repoURL: '{{app.repoURL}}'
        path: '{{app.path}}'
        targetRevision: '{{app.targetRevision}}'
```

---

## Multi-Tenancy Model

### Namespace Isolation

Each team receives a dedicated namespace with strict isolation:

```mermaid
flowchart TB
    subgraph "Team Backend Namespace"
        RQ[ResourceQuota<br/>CPU: 20, Memory: 40Gi]
        LR[LimitRange<br/>Default: 500m/512Mi]
        NP[NetworkPolicy<br/>Deny by default]
        RBAC[RBAC Roles<br/>Developer, Viewer]

        subgraph "Workloads"
            POD1[backend-api]
            POD2[backend-worker]
        end
    end

    RQ -.enforces.-> POD1
    RQ -.enforces.-> POD2
    LR -.defaults.-> POD1
    LR -.defaults.-> POD2
    NP -.controls.-> POD1
    NP -.controls.-> POD2
    RBAC -.permissions.-> POD1
    RBAC -.permissions.-> POD2
```

### Resource Hierarchy

```
AppProject (team-backend)
├── Scoped to namespace: team-backend
├── Source repositories: github.com/fast-ish/*
├── RBAC Roles
│   ├── developer: sync, update, delete
│   └── viewer: get, list
└── Resource Whitelist/Blacklist
    ├── Allowed: Deployments, Services, ConfigMaps
    └── Blocked: ResourceQuota, LimitRange, NetworkPolicy
```

---

## Security Architecture

### Defense in Depth

Multiple security layers protect the platform:

```mermaid
flowchart TB
    subgraph "Layer 1: Network"
        VPC[VPC Isolation]
        SG[Security Groups]
        NACL[Network ACLs]
        NP[Network Policies]
    end

    subgraph "Layer 2: Authentication"
        IRSA[IAM Roles for Service Accounts]
        RBAC[Kubernetes RBAC]
        PROJ[ArgoCD Projects]
    end

    subgraph "Layer 3: Authorization"
        POL[Kyverno Policies]
        PSS[Pod Security Standards]
        OPA[Policy Enforcement]
    end

    subgraph "Layer 4: Runtime"
        FALCO[Falco Runtime Security]
        TRIVY[Trivy Scanning]
        AUDIT[Audit Logs]
    end

    VPC --> IRSA
    SG --> RBAC
    NACL --> PROJ
    NP --> POL

    IRSA --> FALCO
    RBAC --> TRIVY
    PROJ --> AUDIT
    POL --> AUDIT
```

### Policy Enforcement Flow

```mermaid
sequenceDiagram
    participant User
    participant API as Kube API Server
    participant Kyverno
    participant Admission as Admission Controller
    participant etcd

    User->>API: Create Pod
    API->>Admission: Validate
    Admission->>Kyverno: Check Policies

    Kyverno->>Kyverno: Evaluate Rules

    alt Policy Pass
        Kyverno-->>Admission: Allow
        Admission-->>API: Accept
        API->>etcd: Store
        etcd-->>User: Success
    else Policy Fail
        Kyverno-->>Admission: Deny
        Admission-->>API: Reject
        API-->>User: Error
    end
```

---

## Network Architecture

### Namespace Network Isolation

```mermaid
flowchart TB
    subgraph "Internet"
        CLIENT[External Client]
    end

    subgraph "AWS VPC"
        ALB[Application Load Balancer]

        subgraph "team-backend namespace"
            API[backend-api<br/>Service]
            WORKER[backend-worker<br/>Service]
        end

        subgraph "team-frontend namespace"
            WEB[frontend-web<br/>Service]
        end

        subgraph "monitoring namespace"
            PROM[Prometheus]
        end
    end

    subgraph "AWS Services"
        RDS[(RDS Database)]
        S3[(S3 Bucket)]
        SM[Secrets Manager]
    end

    CLIENT --> ALB
    ALB --> API
    ALB --> WEB

    API --> WORKER
    API --> RDS
    WORKER --> S3

    PROM -.metrics.-> API
    PROM -.metrics.-> WEB

    API -.secrets.-> SM
```

### Default Network Policies

Each team namespace gets these policies by default:

1. **Default Deny All**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

2. **Allow DNS**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

3. **Allow Intra-Namespace**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}
```

---

## Integration Points

### AWS Integration

| Service | Purpose | Authentication |
|---------|---------|----------------|
| **Secrets Manager** | Secret storage and rotation | IRSA (IAM Roles for Service Accounts) |
| **ECR** | Container image registry | IRSA with pull permissions |
| **S3** | Backup storage (Velero), logs | IRSA with read/write permissions |
| **CloudWatch** | Logs and metrics | IRSA with write permissions |
| **EKS** | Kubernetes control plane | IAM authentication |

### External Secrets Flow

```mermaid
sequenceDiagram
    participant Pod
    participant ES as External Secrets Operator
    participant K8s as Kubernetes Secret
    participant SM as AWS Secrets Manager

    Pod->>K8s: Request Secret
    K8s->>ES: ExternalSecret exists?
    ES->>SM: Fetch Secret (IRSA)
    SM-->>ES: Return Secret Data
    ES->>K8s: Create/Update Secret
    K8s-->>Pod: Mount Secret
```

### Platform Service Integration

When deployed via the Fastish platform:

| Component | Integration Point | Purpose |
|-----------|------------------|---------|
| **Orchestrator** | CodePipeline | Automated deployment triggers |
| **Portal** | API Gateway | User management and provisioning |
| **Network** | VPC Peering | Cross-stack connectivity |
| **Reporting** | CloudWatch | Usage metrics and billing |

---

## Best Practices

### Repository Organization

1. **Keep platform and teams separate**: Platform components in `/platform`, team configs in `/teams`
2. **Use clear naming**: `team-{name}` for namespaces, `{team}-{app}` for applications
3. **Document changes**: Meaningful commit messages and PR descriptions
4. **Version control**: Use Git tags for releases

### ArgoCD Configuration

1. **Use ApplicationSets**: Dynamic generation from team configs
2. **Enable auto-sync**: For non-production environments
3. **Set sync waves**: Control deployment order
4. **Health checks**: Define custom health assessments

### Security Best Practices

1. **Principle of least privilege**: Minimal RBAC permissions
2. **Network segmentation**: NetworkPolicies for all namespaces
3. **Secret management**: Never commit secrets to Git
4. **Regular scanning**: Enable Trivy for vulnerability detection

---

## Related Documentation

- [README](../README.md) - Main documentation
- [Policy Guide](POLICY-GUIDE.md) - Policy enforcement details
- [Team Onboarding](TEAM-ONBOARDING.md) - Team onboarding process
- [Security](SECURITY.md) - Security best practices
