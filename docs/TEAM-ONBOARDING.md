# Team Onboarding Guide

This guide provides step-by-step instructions for onboarding a new development team to the Internal Developer Platform.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Onboarding Process](#onboarding-process)
- [Team Configuration](#team-configuration)
- [Application Deployment](#application-deployment)
- [Access Management](#access-management)
- [Best Practices](#best-practices)
- [Common Workflows](#common-workflows)
- [FAQ](#faq)

---

## Overview

The onboarding process provisions a complete isolated environment for your team, including:

- Dedicated Kubernetes namespace with resource quotas
- Network policies for secure communication
- RBAC roles for team members
- ArgoCD project for application management
- Integration with observability and security tooling

### Onboarding Timeline

| Phase | Duration | Activities |
|-------|----------|------------|
| **Preparation** | 1-2 days | Requirements gathering, resource planning |
| **Configuration** | 1 day | Create team.yaml, submit PR |
| **Review** | 1-2 days | Platform team review and approval |
| **Provisioning** | 5-10 minutes | Automated resource creation |
| **Validation** | 1 day | Test deployments, access verification |

---

## Prerequisites

Before starting the onboarding process, ensure you have:

- [ ] GitHub account with access to the organization
- [ ] AWS SSO access to the platform account
- [ ] kubectl installed and configured
- [ ] ArgoCD CLI installed
- [ ] Understanding of your team's resource requirements
- [ ] List of applications to deploy

### Required Information

Gather the following information:

| Information | Example | Purpose |
|-------------|---------|---------|
| Team Name | `data-platform` | Identifier for namespace and resources |
| Team Email | `data-platform@company.com` | Contact for notifications |
| Slack Channel | `#data-platform` | Team communication channel |
| Resource Estimates | 20 cores, 40Gi memory | Initial quota allocation |
| Application Repositories | `github.com/org/data-api` | Git repos to deploy |

---

## Onboarding Process

### Step 1: Fork and Clone Repository

```bash
# Fork the repository on GitHub
# Then clone your fork
git clone https://github.com/<your-username>/aws-idp-gitops.git
cd aws-idp-gitops

# Add upstream remote
git remote add upstream https://github.com/fast-ish/aws-idp-gitops.git
```

### Step 2: Create Feature Branch

```bash
# Create a branch for your team
git checkout -b onboard-team-<your-team-name>
```

### Step 3: Create Team Directory

```bash
# Create team directory
mkdir -p teams/<your-team-name>
```

### Step 4: Create Team Configuration

Create `teams/<your-team-name>/team.yaml`:

```yaml
# Basic Information
name: data-platform
namespace: team-data-platform
description: "Data platform team - ETL pipelines, data APIs, and analytics"

# Team Contacts
owner:
  name: "Data Platform Team"
  email: data-platform@company.com
  slack: "#data-platform"

# Resource Quotas
resourceQuota:
  requests:
    cpu: "20"              # Total CPU requests
    memory: "40Gi"         # Total memory requests
  limits:
    cpu: "40"              # Total CPU limits
    memory: "80Gi"         # Total memory limits
  pods: "50"               # Maximum number of pods
  services: "15"           # Maximum number of services
  configmaps: "30"         # Maximum number of ConfigMaps
  secrets: "30"            # Maximum number of Secrets
  persistentvolumeclaims: "10"  # Maximum number of PVCs

# Container Resource Defaults
limitRange:
  default:                 # Default limits if not specified
    cpu: "1"
    memory: "1Gi"
  defaultRequest:          # Default requests if not specified
    cpu: "200m"
    memory: "256Mi"
  max:                     # Maximum per container
    cpu: "4"
    memory: "8Gi"

# Network Access
networkPolicy:
  allowIngress: true       # Allow external traffic
  allowEgress: true        # Allow outbound traffic
  allowFromNamespaces:     # Allow traffic from these namespaces
    - argocd
    - monitoring
    - team-backend         # If you need to access backend services

# Applications to Deploy
apps:
  - name: data-api
    repoURL: https://github.com/fast-ish/data-api
    path: k8s/overlays/production
    targetRevision: main
    environment: production
    syncPolicy:
      automated:
        prune: true        # Auto-delete resources removed from Git
        selfHeal: true     # Auto-sync on drift

  - name: etl-pipeline
    repoURL: https://github.com/fast-ish/etl-pipeline
    path: k8s
    targetRevision: main
    environment: production
```

### Step 5: Commit and Push

```bash
# Add your changes
git add teams/<your-team-name>/

# Commit with descriptive message
git commit -m "Add team configuration for <your-team-name>

- Initial resource quotas: 20 CPU, 40Gi memory
- Applications: data-api, etl-pipeline
- Network policies: allow from monitoring and backend"

# Push to your fork
git push origin onboard-team-<your-team-name>
```

### Step 6: Create Pull Request

1. Go to GitHub and create a PR from your fork to the main repository
2. Fill out the PR template with:
   - Team name and purpose
   - Resource requirements and justification
   - List of applications
   - Expected traffic patterns
   - Any special requirements

### Step 7: PR Review

The platform team will review:

- [ ] Resource quotas are reasonable
- [ ] Network policies follow security guidelines
- [ ] Application repositories exist and are accessible
- [ ] Team configuration follows naming conventions
- [ ] No security violations

### Step 8: Merge and Provisioning

Once approved and merged:

1. ArgoCD detects the change (within 3 minutes)
2. ApplicationSets generate resources:
   - ArgoCD AppProject
   - Kubernetes namespace
   - ResourceQuota and LimitRange
   - NetworkPolicies
   - RBAC roles
   - Applications

Monitor the provisioning:

```bash
# Watch for new resources
watch kubectl get appproject -n argocd

# Check namespace creation
kubectl get namespace team-<your-team-name>

# View applications
kubectl get applications -n argocd -l team=<your-team-name>
```

---

## Team Configuration

### Resource Quota Guidelines

Choose resource quotas based on your workload type:

| Workload Type | CPU Requests | Memory Requests | Recommended For |
|---------------|--------------|-----------------|-----------------|
| **Lightweight APIs** | 5-10 cores | 10-20Gi | Simple REST APIs, webhooks |
| **Standard Services** | 10-20 cores | 20-40Gi | Microservices, web applications |
| **Data Processing** | 30-50 cores | 60-100Gi | ETL pipelines, batch processing |
| **ML Workloads** | 40-100 cores | 100-200Gi | Model training, inference |

### Network Policy Configuration

Common network policy patterns:

#### 1. Public-Facing Service

```yaml
networkPolicy:
  allowIngress: true
  allowEgress: true
  allowFromNamespaces:
    - ingress-nginx      # Allow ALB traffic
    - monitoring         # Allow metrics scraping
```

#### 2. Internal Service

```yaml
networkPolicy:
  allowIngress: true
  allowEgress: true
  allowFromNamespaces:
    - team-frontend      # Allow frontend to call this service
    - team-backend       # Allow backend to call this service
    - monitoring         # Allow metrics scraping
```

#### 3. Batch Job

```yaml
networkPolicy:
  allowIngress: false    # No incoming traffic
  allowEgress: true      # Need to call external APIs
  allowFromNamespaces:
    - monitoring         # Allow metrics scraping
```

### Application Configuration

Each application entry supports:

```yaml
apps:
  - name: my-app
    repoURL: https://github.com/org/repo
    path: k8s/overlays/production
    targetRevision: main
    environment: production

    # Optional: Sync policy
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
      syncOptions:
        - CreateNamespace=false
        - PruneLast=true

    # Optional: Ignore differences
    ignoreDifferences:
      - group: apps
        kind: Deployment
        jsonPointers:
          - /spec/replicas

    # Optional: Custom health check
    health:
      - group: argoproj.io
        kind: Rollout
        check: |
          health_status = {}
          if obj.status.phase == "Healthy" then
            health_status.status = "Healthy"
          end
          return health_status
```

---

## Application Deployment

### Preparing Your Application Repository

Your application repository should have Kubernetes manifests in one of these formats:

#### Option 1: Plain YAML

```
my-app/
└── k8s/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    └── configmap.yaml
```

#### Option 2: Kustomize

```
my-app/
└── k8s/
    ├── base/
    │   ├── kustomization.yaml
    │   ├── deployment.yaml
    │   └── service.yaml
    └── overlays/
        ├── development/
        │   └── kustomization.yaml
        └── production/
            └── kustomization.yaml
```

#### Option 3: Helm

```
my-app/
└── helm/
    ├── Chart.yaml
    ├── values.yaml
    ├── values-production.yaml
    └── templates/
        ├── deployment.yaml
        └── service.yaml
```

### Required Kubernetes Manifests

At minimum, your application should have:

1. **Deployment or StatefulSet**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
    version: v1.0.0
    team: data-platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
        version: v1.0.0
        team: data-platform
    spec:
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          image: my-app:v1.0.0
          ports:
            - containerPort: 8080
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 1
              memory: 1Gi
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
```

2. **Service**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: my-app
```

3. **Ingress** (if publicly accessible)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - host: my-app.company.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

---

## Access Management

### Accessing Your Namespace

```bash
# Configure kubectl context
aws eks update-kubeconfig --name <cluster-name> --region <region>

# Verify access
kubectl auth can-i get pods -n team-<your-team>

# View your resources
kubectl get all -n team-<your-team>
```

### ArgoCD Access

```bash
# Login to ArgoCD
argocd login argocd.company.com --sso

# List your applications
argocd app list --project team-<your-team>

# Get application status
argocd app get team-<your-team>-<app-name>

# Sync application
argocd app sync team-<your-team>-<app-name>
```

### RBAC Roles

Your team has two predefined roles:

| Role | Permissions | Who Should Have It |
|------|-------------|-------------------|
| **Developer** | Create, update, delete resources in namespace; Sync ArgoCD apps | Developers who deploy code |
| **Viewer** | Read-only access to namespace; View ArgoCD apps | QA, stakeholders, observers |

Request access by opening a ticket with the platform team.

---

## Best Practices

### 1. Resource Management

- Start with conservative quotas and increase as needed
- Set resource requests equal to typical usage
- Set resource limits at 2x requests for bursting
- Use VPA recommendations to right-size containers

### 2. Security

- Always run as non-root user
- Use read-only root filesystem
- Drop all capabilities
- Enable Pod Security Standards
- Never commit secrets to Git
- Use External Secrets for sensitive data

### 3. Observability

- Add standard labels: `app`, `version`, `team`
- Implement health check endpoints
- Configure liveness and readiness probes
- Export Prometheus metrics
- Use structured logging

### 4. High Availability

- Run at least 3 replicas for production
- Set Pod Disruption Budgets
- Use anti-affinity rules
- Configure autoscaling (HPA/VPA)

### 5. GitOps

- Use separate branches for environments
- Tag releases in Git
- Write descriptive commit messages
- Use PR reviews for changes
- Enable auto-sync for non-prod environments

---

## Common Workflows

### Deploying a New Application

1. Add application entry to `team.yaml`
2. Create PR with the change
3. Wait for approval and merge
4. ArgoCD automatically deploys the app

### Updating Resource Quotas

1. Edit `resourceQuota` in `team.yaml`
2. Create PR explaining why increase is needed
3. Platform team reviews and approves
4. Merge triggers quota update

### Adding Network Access

1. Edit `networkPolicy.allowFromNamespaces` in `team.yaml`
2. Create PR explaining the connectivity need
3. Security review and approval
4. Merge applies new network policy

### Debugging Failed Deployments

```bash
# Check application status
argocd app get team-<your-team>-<app>

# View sync errors
argocd app sync team-<your-team>-<app> --dry-run

# Check pod status
kubectl get pods -n team-<your-team>

# View pod logs
kubectl logs -n team-<your-team> <pod-name>

# Describe pod for events
kubectl describe pod -n team-<your-team> <pod-name>

# Check policy violations
kubectl get policyreport -n team-<your-team>
```

---

## FAQ

### How long does onboarding take?

After PR approval, resources are typically provisioned within 3-5 minutes.

### Can I have multiple namespaces?

Each team gets one namespace. If you need isolation, use labels and network policies within your namespace.

### How do I request more resources?

Edit your `team.yaml` and create a PR. Include justification for the increase.

### Can I deploy to multiple clusters?

Yes, but each cluster requires separate configuration. Contact the platform team.

### How do I manage secrets?

Use External Secrets to sync from AWS Secrets Manager. Never commit secrets to Git.

### Can I use custom domains?

Yes, configure Ingress resources in your application manifests. DNS must be configured separately.

### How do I enable autoscaling?

Add HPA (Horizontal Pod Autoscaler) manifests to your application repository.

### Can I run privileged containers?

No, privileged mode is blocked by policy. Contact the platform team if you have a special use case.

### How do I get help?

- Check documentation: https://docs.company.com
- Slack: #platform-support
- Email: platform-team@company.com
- Office hours: Tuesdays 2-3 PM

---

## Related Documentation

- [README](../README.md) - Main documentation
- [Architecture Guide](ARCHITECTURE.md) - Platform architecture
- [Policy Guide](POLICY-GUIDE.md) - Policy enforcement
- [Security Guide](SECURITY.md) - Security best practices
