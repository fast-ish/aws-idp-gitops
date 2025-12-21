# AWS IDP GitOps

GitOps repository for the Internal Developer Platform (IDP). This repository is managed by ArgoCD and contains all Kubernetes manifests for platform components and team applications.

## Repository Structure

```
aws-idp-gitops/
├── README.md
├── clusters/                    # Cluster-specific configurations
│   └── production/
│       ├── kustomization.yaml   # What's deployed to this cluster
│       └── cluster-config.yaml  # Cluster metadata
├── platform/                    # Platform components (shared)
│   ├── external-secrets/        # External Secrets Operator config
│   ├── kyverno/                 # Policy engine
│   │   └── policies/
│   │       ├── baseline/        # Pod Security Standards (Enforce)
│   │       ├── best-practices/  # Best practices (Audit)
│   │       └── mutations/       # Default mutations
│   ├── argo-workflows/          # CI pipeline templates
│   ├── argocd/                  # GitOps configuration
│   │   ├── applicationsets/     # ApplicationSets for teams
│   │   └── kustomization.yaml
│   ├── team-resources/          # Helm chart for team namespaces
│   │   └── templates/
│   │       ├── namespace.yaml
│   │       ├── resource-quota.yaml
│   │       ├── limit-range.yaml
│   │       ├── network-policy.yaml
│   │       └── rbac.yaml
│   └── team-project/            # Helm chart for ArgoCD AppProjects
└── teams/                       # Team configurations
    ├── platform/
    ├── backend/
    ├── frontend/
    ├── data/
    ├── ml/
    └── integrations/
```

## How It Works

1. **ArgoCD watches this repository** - Any changes merged to `main` are automatically synced to the cluster
2. **ApplicationSets generate resources** - Three ApplicationSets process `teams/*/team.yaml`:
   - `team-projects` - Creates ArgoCD AppProjects with RBAC
   - `team-namespaces` - Creates namespaces with quotas, limits, and network policies
   - `team-apps` - Deploys each team's applications
3. **Kyverno enforces policies** - Security policies are applied to all workloads
4. **External Secrets syncs credentials** - AWS Secrets Manager secrets are automatically synced

## Teams

Teams are organized by function. Each team gets:
- **Dedicated namespace** with resource quotas and limits
- **Network policies** controlling ingress/egress
- **RBAC roles** (developer, viewer)
- **ArgoCD AppProject** scoped to their namespace

| Team | Namespace | Description |
|------|-----------|-------------|
| `platform` | `team-platform` | Platform engineering - internal tools and developer experience |
| `backend` | `team-backend` | Backend services - APIs, microservices, and core business logic |
| `frontend` | `team-frontend` | Frontend team - web applications, SSR, and static sites |
| `data` | `team-data` | Data engineering - ETL pipelines, data APIs, and analytics |
| `ml` | `team-ml` | Machine learning - model training, inference, and feature engineering |
| `integrations` | `team-integrations` | Integrations - third-party APIs, webhooks, and external connectors |

## Adding a New Team

1. Create a directory under `teams/`:
   ```bash
   mkdir -p teams/my-team
   ```

2. Create a `team.yaml` file:
   ```yaml
   name: my-team
   namespace: team-my-team
   description: "My team description"

   owner:
     name: "My Team"
     email: my-team@example.com
     slack: "#my-team"

   resourceQuota:
     requests:
       cpu: "10"
       memory: "20Gi"
     limits:
       cpu: "20"
       memory: "40Gi"
     pods: "30"
     services: "10"
     configmaps: "20"
     secrets: "20"
     persistentvolumeclaims: "5"

   limitRange:
     default:
       cpu: "500m"
       memory: "512Mi"
     defaultRequest:
       cpu: "100m"
       memory: "128Mi"
     max:
       cpu: "2"
       memory: "4Gi"

   networkPolicy:
     allowIngress: true
     allowEgress: true
     allowFromNamespaces:
       - argocd
       - monitoring

   apps:
     - name: my-service
       repoURL: https://github.com/fast-ish/my-service
       path: k8s
       targetRevision: main
       environment: production
   ```

3. Open a PR - ArgoCD will preview the changes
4. Merge - ArgoCD auto-syncs within 3 minutes

## What Gets Created Per Team

When a team is added, the ApplicationSets automatically create:

### ArgoCD AppProject (`team-{name}`)
- Scoped to team's namespace only
- Source repos: `github.com/fast-ish/*` and `github.com/{team}/*`
- Allowed resources: Deployments, Services, ConfigMaps, Secrets, Ingresses, etc.
- Blocked resources: ResourceQuotas, LimitRanges, NetworkPolicies (platform-managed)
- Roles: `developer` (sync apps), `viewer` (read-only)

### Namespace Resources
- **Namespace** with Pod Security Standards labels (restricted)
- **ResourceQuota** limiting CPU, memory, pods, services, etc.
- **LimitRange** setting default/max container resources
- **NetworkPolicy** controlling traffic flow
- **ServiceAccount** for workloads
- **RBAC Roles** (developer, viewer)

### Applications
- One ArgoCD Application per entry in `apps[]`
- Auto-synced from the specified Git repository
- Deployed to the team's namespace

## Platform Components

| Component | Purpose | Namespace |
|-----------|---------|-----------|
| External Secrets | Sync AWS Secrets Manager → K8s Secrets | `external-secrets` |
| Kyverno | Policy engine for security & governance | `kyverno` |
| Argo Workflows | CI pipelines (build, test, deploy) | `argo` |
| ArgoCD | GitOps continuous deployment | `argocd` |
| Reloader | Auto-restart pods on config changes | `reloader` |

## Policy Enforcement

### Baseline (Enforced)
- No privileged containers
- No host namespaces (PID, IPC, Network)
- No host ports
- Limited capabilities (only NET_BIND_SERVICE)

### Best Practices (Audit)
- Resource requests/limits required
- Standard labels required
- Liveness/readiness probes required

### Network Policies (Per Team)
- Allow traffic within same namespace
- Allow traffic from ingress controller
- Allow DNS resolution
- Allow traffic from specified namespaces
- Allow egress to external APIs (configurable)

## Team Resource Quotas

Default quotas can be customized per team:

| Team Type | CPU Requests | Memory Requests | Pods |
|-----------|--------------|-----------------|------|
| Platform | 10 | 20Gi | 30 |
| Backend | 20 | 40Gi | 50 |
| Frontend | 8 | 16Gi | 30 |
| Data | 40 | 80Gi | 40 |
| ML | 32 | 128Gi | 25 |
| Integrations | 12 | 24Gi | 40 |

## Related Repositories

| Repository | Purpose |
|------------|---------|
| [aws-idp-infra](https://github.com/fast-ish/aws-idp-infra) | CDK infrastructure (EKS, Aurora, IAM) |
| [backstage-ext](https://github.com/fast-ish/backstage-ext) | Backstage application |
| [cdk-common](https://github.com/fast-ish/cdk-common) | Shared CDK constructs |
