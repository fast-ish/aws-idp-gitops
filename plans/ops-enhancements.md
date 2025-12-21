# Operational Enhancements Plan

## Overview
Add Governance & Compliance, Resource Optimization, Observability, and Backup/Restore enhancements.

---

## Helm Charts Required

| Chart | Repository | Version | Purpose |
|-------|------------|---------|---------|
| goldilocks | https://charts.fairwinds.com/stable | 8.0.2 | VPA recommendations dashboard |
| velero | https://vmware-tanzu.github.io/helm-charts | 6.0.0 | Backup & disaster recovery |
| aws-node-termination-handler | https://aws.github.io/eks-charts | 0.21.0 | Spot interruption handling |

**Already Installed:** Kyverno, Prometheus, Grafana, Loki, Tempo, VPA, Karpenter

---

## 1. Governance & Compliance

### 1.1 Drift Detection
| File | Purpose |
|------|---------|
| `platform/governance/drift-detection/prometheus-rules.yaml` | Drift alerts (5min, 30min windows) |
| `platform/governance/drift-detection/remediation-workflow.yaml` | Argo Workflow for auto-remediation |
| `platform/kyverno/policies/compliance/audit-resource-changes.yaml` | Track manual kubectl changes |

### 1.2 Audit Trail
| File | Purpose |
|------|---------|
| `platform/argocd/notifications-audit.yaml` | Audit log templates |
| `platform/observability/grafana/dashboards/audit-trail.json` | Who/what/when dashboard |

### 1.3 Deployment Windows
| File | Purpose |
|------|---------|
| `platform/governance/deployment-windows/configmap.yaml` | Freeze status config |
| `platform/governance/deployment-windows/cronjobs.yaml` | Automated freeze scheduler |
| `platform/kyverno/policies/compliance/deployment-windows.yaml` | Enforce freeze policy |

**ConfigMap States:**
- `status: open` - Deployments allowed
- `status: frozen` - Deployments blocked (bypass: `fasti.sh/bypass-freeze=true`)

---

## 2. Resource Optimization

### 2.1 Goldilocks (VPA Dashboard)
| File | Purpose |
|------|---------|
| `platform/goldilocks/kustomization.yaml` | Kustomize config |
| `platform/goldilocks/values.yaml` | Helm values |
| `platform/kyverno/policies/mutations/add-goldilocks-label.yaml` | Auto-label team namespaces |
| `platform/observability/alerting/rightsizing-alerts.yaml` | Over/under provisioned alerts |

### 2.2 Spot Instance Automation
| File | Purpose |
|------|---------|
| `platform/spot-handler/kustomization.yaml` | Kustomize config |
| `platform/spot-handler/values.yaml` | AWS NTH Helm values |
| `platform/spot-handler/prometheus-rules.yaml` | Interruption alerts |
| `platform/kyverno/policies/generators/generate-pdb.yaml` | Auto-create PDBs |

### 2.3 Idle Resource Detection
| File | Purpose |
|------|---------|
| `platform/observability/alerting/idle-resource-alerts.yaml` | Detect unused resources |
| `platform/argo-workflows/templates/idle-resource-report.yaml` | Weekly report |

---

## 3. Observability

### 3.1 Golden Signals Dashboards
| File | Purpose |
|------|---------|
| `platform/observability/grafana/dashboards/golden-signals-template.json` | Per-service dashboard |
| `platform/kyverno/policies/generators/generate-service-dashboard.yaml` | Auto-create dashboards |

### 3.2 SLO Burn Rate Alerts
| File | Purpose |
|------|---------|
| `platform/observability/alerting/slo-burn-rate-multiwindow.yaml` | Multi-window burn rate |

| Alert | Burn Rate | Windows | Action |
|-------|-----------|---------|--------|
| Critical | 14.4x | 5m + 1h | Page (2% budget/hour) |
| High | 6x | 30m + 6h | Ticket (5% budget/6h) |
| Elevated | 1x | 1d + 3d | Info (on track to exhaust) |

### 3.3 Trace-Deployment Correlation
| File | Purpose |
|------|---------|
| `platform/observability/tempo/metrics-generator.yaml` | Span metrics with deployment labels |
| `platform/observability/grafana/dashboards/deployment-traces.json` | Correlation dashboard |
| `platform/observability/alerting/trace-alerts.yaml` | Deployment-introduced errors |
| `platform/argo-rollouts/analysis-templates/trace-analysis.yaml` | Rollout analysis |

---

## 4. Backup & Restore (Velero)

### 4.1 Velero Installation
| File | Purpose |
|------|---------|
| `platform/velero/kustomization.yaml` | Kustomize config |
| `platform/velero/namespace.yaml` | Velero namespace |
| `platform/velero/values.yaml` | Helm values |
| `platform/velero/aws-credentials.yaml` | ExternalSecret for S3 |

### 4.2 Backup Schedules
| Schedule | Namespaces | Retention | Frequency |
|----------|------------|-----------|-----------|
| daily-cluster-backup | All (excl. system) | 30 days | Daily 3 AM |
| hourly-critical-backup | prod-*, team-* (label: backup=critical) | 7 days | Hourly |
| pre-deploy-snapshot | Target namespace | 48 hours | Before each deploy |

### 4.3 Restore Workflows
| File | Purpose |
|------|---------|
| `platform/velero/restore-workflow.yaml` | Argo Workflow for restore |
| `platform/velero/disaster-recovery-runbook.yaml` | DR runbook ConfigMap |
| `platform/observability/alerting/velero-alerts.yaml` | Backup failure alerts |

### 4.4 AWS Infrastructure (CDK)
| Resource | Purpose |
|----------|---------|
| S3 Bucket | `fasti-sh-velero-backups` with versioning |
| IAM Role | IRSA for Velero service account |
| KMS Key | Encryption for backup data |

---

## Directory Structure

```
platform/
├── governance/
│   ├── drift-detection/
│   │   ├── prometheus-rules.yaml
│   │   └── remediation-workflow.yaml
│   └── deployment-windows/
│       ├── configmap.yaml
│       └── cronjobs.yaml
├── goldilocks/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   └── values.yaml
├── spot-handler/
│   ├── kustomization.yaml
│   ├── values.yaml
│   └── prometheus-rules.yaml
├── velero/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── values.yaml
│   ├── aws-credentials.yaml
│   ├── restore-workflow.yaml
│   └── disaster-recovery-runbook.yaml
```

---

## Summary

| Category | Components | New Files |
|----------|------------|-----------|
| Governance | Drift Detection, Audit Trail, Deployment Windows | 8 |
| Resource Optimization | Goldilocks, Spot Handler, Idle Detection | 10 |
| Observability | Golden Signals, SLO Burn Rate, Trace Correlation | 7 |
| Backup & Restore | Velero, Schedules, Workflows, Alerts | 8 |
| **Total** | **12 features** | **~33 files** |

| Helm Chart | Version | Namespace |
|------------|---------|-----------|
| goldilocks | 8.0.2 | goldilocks |
| velero | 6.0.0 | velero |
| aws-node-termination-handler | 0.21.0 | kube-system |

---

## 5. CDK Infrastructure Changes

### 5.1 Add Helm Charts to EKS Addons

**File:** `aws-idp-infra/src/main/java/fasti/sh/idp/stack/CoreAddonsNestedStack.java`

Add the following Helm chart configurations:

```java
// Velero - Backup & Disaster Recovery
HelmChart velero = HelmChart.Builder.create(this, "velero")
    .cluster(cluster)
    .chart("velero")
    .repository("https://vmware-tanzu.github.io/helm-charts")
    .version("6.0.0")
    .namespace("velero")
    .createNamespace(true)
    .values(loadValues("velero.mustache"))
    .build();

// Goldilocks - VPA Recommendations Dashboard
HelmChart goldilocks = HelmChart.Builder.create(this, "goldilocks")
    .cluster(cluster)
    .chart("goldilocks")
    .repository("https://charts.fairwinds.com/stable")
    .version("8.0.2")
    .namespace("goldilocks")
    .createNamespace(true)
    .values(loadValues("goldilocks.mustache"))
    .build();

// AWS Node Termination Handler - Spot Interruption Handling
HelmChart nodeTerminationHandler = HelmChart.Builder.create(this, "aws-node-termination-handler")
    .cluster(cluster)
    .chart("aws-node-termination-handler")
    .repository("https://aws.github.io/eks-charts")
    .version("0.21.0")
    .namespace("kube-system")
    .values(loadValues("node-termination-handler.mustache"))
    .build();
```

### 5.2 Create Mustache Templates

**File:** `aws-idp-infra/src/main/resources/production/v1/helm/velero.mustache`

```yaml
configuration:
  backupStorageLocation:
    - name: default
      provider: aws
      bucket: {{deployment:id}}-velero-backups
      config:
        region: {{deployment:region}}
  volumeSnapshotLocation:
    - name: default
      provider: aws
      config:
        region: {{deployment:region}}

serviceAccount:
  server:
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::{{deployment:account}}:role/{{deployment:id}}-velero

initContainers:
  - name: velero-plugin-for-aws
    image: velero/velero-plugin-for-aws:v1.9.0
    volumeMounts:
      - mountPath: /target
        name: plugins

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
```

**File:** `aws-idp-infra/src/main/resources/production/v1/helm/goldilocks.mustache`

```yaml
vpa:
  enabled: false  # Use existing VPA

dashboard:
  enabled: true
  ingress:
    enabled: true
    ingressClassName: alb
    annotations:
      alb.ingress.kubernetes.io/scheme: internal
      alb.ingress.kubernetes.io/target-type: ip
    hosts:
      - host: goldilocks.{{deployment:domain}}
        paths:
          - path: /
            pathType: Prefix

controller:
  labelSelector:
    matchLabels:
      goldilocks.fairwinds.com/enabled: "true"

serviceMonitor:
  enabled: true
```

**File:** `aws-idp-infra/src/main/resources/production/v1/helm/node-termination-handler.mustache`

```yaml
enableSpotInterruptionDraining: true
enableRebalanceMonitoring: true
enableScheduledEventDraining: true
enablePrometheusServer: true
prometheusServerPort: 9092

nodeSelector:
  karpenter.sh/capacity-type: spot

tolerations:
  - operator: Exists

serviceMonitor:
  create: true

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 100m
    memory: 128Mi
```

### 5.3 Create IAM Policies

**File:** `aws-idp-infra/src/main/resources/production/v1/policy/velero.mustache`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVolumes",
        "ec2:DescribeSnapshots",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:CreateSnapshot",
        "ec2:DeleteSnapshot"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:PutObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": "arn:aws:s3:::{{deployment:id}}-velero-backups/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::{{deployment:id}}-velero-backups"
    }
  ]
}
```

### 5.4 Create S3 Bucket for Velero

**File:** `aws-idp-infra/src/main/java/fasti/sh/idp/stack/IdpSetupNestedStack.java`

```java
// Velero Backup Bucket
Bucket veleroBucket = Bucket.Builder.create(this, "VeleroBackupBucket")
    .bucketName(deploymentId + "-velero-backups")
    .versioned(true)
    .encryption(BucketEncryption.S3_MANAGED)
    .blockPublicAccess(BlockPublicAccess.BLOCK_ALL)
    .lifecycleRules(List.of(
        LifecycleRule.builder()
            .id("delete-old-backups")
            .expiration(Duration.days(90))
            .noncurrentVersionExpiration(Duration.days(30))
            .build()
    ))
    .removalPolicy(RemovalPolicy.RETAIN)
    .build();
```

### 5.5 Update Existing Files

**Modify:** `aws-idp-infra/src/main/resources/production/v1/eks/addons.mustache`

Add after existing addons:

```yaml
# Velero for backup/restore
- name: velero
  namespace: velero
  chart: velero
  repository: https://vmware-tanzu.github.io/helm-charts
  version: "6.0.0"
  values: {{> helm/velero.mustache}}

# Goldilocks for VPA recommendations
- name: goldilocks
  namespace: goldilocks
  chart: goldilocks
  repository: https://charts.fairwinds.com/stable
  version: "8.0.2"
  values: {{> helm/goldilocks.mustache}}

# AWS Node Termination Handler
- name: aws-node-termination-handler
  namespace: kube-system
  chart: aws-node-termination-handler
  repository: https://aws.github.io/eks-charts
  version: "0.21.0"
  values: {{> helm/node-termination-handler.mustache}}
```

---

## CDK Execution Order

1. **Create S3 bucket** for Velero backups (IdpSetupNestedStack)
2. **Create IAM policies** for Velero IRSA
3. **Deploy Helm charts** via CoreAddonsNestedStack
4. **Sync GitOps** configurations via ArgoCD
