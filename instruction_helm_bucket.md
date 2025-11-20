# Crossplane + Helm: Complete Bucket Management Guide

A step-by-step guide to set up Crossplane with Helm for managing S3 buckets across multiple AWS accounts and regions. Developers can create/delete buckets by simply editing YAML files in Git.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Step 1: Set Up Crossplane Providers](#step-1-set-up-crossplane-providers)
4. [Step 2: Configure Provider Authentication](#step-2-configure-provider-authentication)
5. [Step 3: Create Helm Chart](#step-3-create-helm-chart)
6. [Step 4: Set Up Git Repository](#step-4-set-up-git-repository)
7. [Step 5: Deploy with ArgoCD](#step-5-deploy-with-argocd)
8. [Step 6: Developer Workflow](#step-6-developer-workflow)
9. [Multi-Account & Multi-Region Setup](#step-7-multi-account--multi-region-setup)
10. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
Developer's Git Workflow:
┌──────────────────────────────────────────────────────────────┐
│ 1. Developer edits values-dev.yaml                           │
│ 2. Commits and pushes to Git                                 │
│ 3. ArgoCD detects changes                                    │
│ 4. ArgoCD triggers Helm rendering                            │
│ 5. Helm generates Crossplane manifests                       │
│ 6. Crossplane reads ProviderConfig                           │
│ 7. Crossplane creates S3 bucket in AWS                       │
└──────────────────────────────────────────────────────────────┘

Infrastructure Stack:
┌─────────────────────────┐
│   AWS Accounts          │
│  (Prod/Dev/Multi-Acc)   │
└────────────┬────────────┘
             ▲
             │ Creates resources
┌────────────┴────────────┐
│   Crossplane Providers  │
│  (AWS Family Provider)  │
└────────────┬────────────┘
             ▲
             │ Reads config
┌────────────┴────────────────────┐
│   ProviderConfigs (multiple)    │
│  - prod-account-us-east-1       │
│  - dev-account-us-west-2        │
│  - staging-account-eu-west-1    │
└────────────┬────────────────────┘
             ▲
             │ Deploys via Helm
┌────────────┴────────────────────┐
│   Helm Chart (bucket template)  │
└────────────┬────────────────────┘
             ▲
             │ Syncs
┌────────────┴────────────────────┐
│   ArgoCD Application             │
└────────────┬────────────────────┘
             ▲
             │ Watches
┌────────────┴────────────────────┐
│   Git Repository                │
│  (values-dev.yaml, etc.)        │
└─────────────────────────────────┘
```

---

## Prerequisites

Before starting, ensure you have:

```bash
# 1. Kubernetes cluster running (EKS, Minikube, etc.)
kubectl cluster-info

# 2. Helm installed
helm version

# 3. ArgoCD installed
kubectl create namespace argocd
# Add ArgoCD Helm repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install ArgoCD
helm install argocd argo/argo-cd \
  --namespace argocd \
  --version 5.46.8 \
  --set server.service.type=LoadBalancer

kubectl get pods -n argocd

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

admin
o93IhtmmO7UjeBOU

# For Minikube, get the service URL
minikube service argocd-server -n argocd
# Or use port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Then open: https://localhost:8080



# 4. AWS CLI configured with credentials
aws sts get-caller-identity

# 5. Multiple AWS accounts (optional, for multi-account setup)
# - Have AWS account IDs and credentials for each account
```

---

## Step 1: Set Up Crossplane Providers

### 1.1 Install Crossplane Core
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update


helm install crossplane \
  --namespace crossplane-system \
  --create-namespace \
  crossplane-stable/crossplane \
  --version 2.1.1


kubectl get pods -n crossplane-system


### 1.2 Install AWS Family Provider

helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update


helm install crossplane \
  --namespace crossplane-system \
  --create-namespace \
  crossplane-stable/crossplane \
  --version 2.1.1


kubectl get pods -n crossplane-system -w

# Wait until both pods are Running
# crossplane-xxxxx Running
# crossplane-rbac-manager-xxxxx Running

# Create AWS credentials file
cat > /tmp/aws-credentials.ini <<EOF
[default]
aws_access_key_id = YOUR_AWS_ACCESS_KEY_ID
aws_secret_access_key = YOUR_AWS_SECRET_ACCESS_KEY
EOF

# Create Kubernetes secret
kubectl create secret generic aws-secret \
  --namespace=crossplane-system \
  --from-file=creds=./aws-credentials.ini


# Verify secret was created
kubectl get secret -n crossplane-system
kubectl describe secret aws-secret -n crossplane-system

# Clean up the credentials file
rm /tmp/aws-credentials.txt


**For Production Account (Optional):**
** OPTIONAL FOR MULTIACCOUT TEST**
```bash
cat > /tmp/aws-prod-credentials.ini <<EOF
[default]
aws_access_key_id = YOUR_PROD_ACCOUNT_ACCESS_KEY_ID
aws_secret_access_key = YOUR_PROD_ACCOUNT_SECRET_ACCESS_KEY
EOF

kubectl create secret generic aws-dev-credentials \
  -n crossplane-system \
  --from-file=creds=aws-dev-credentials.ini

rm /tmp/aws-prod-credentials.txt
```

Verify secrets:

```bash
kubectl get secrets -n crossplane-system | grep aws
```

### 2.2 Create ProviderConfigs

Create file: `crossplane/providers/provider-config-dev.yaml`

```yaml
# Development Account - US East 1
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: dev-eu-central-1
  namespace: crossplane-system
spec:
  region: eu-central-1
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-dev-credentials
      key: creds
---
**OPTIONAL** Second account
# Production Account - US East 1 (Optional)
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: prod-us-east-1
  namespace: crossplane-system
spec:
  region: us-east-1
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-prod-credentials
      key: creds
```

Apply provider configs:

```bash
kubectl apply -f crossplane/providers/provider-config-dev.yaml

# Verify
kubectl get providerconfig -n crossplane-system
```




**Key Naming Convention:**

- `dev-eu-central-1` = Development account, eu-central-1
- `prod-us-east-1` = Production account, US East 1 region

---

## Step 3: Create Helm Chart

### 3.1 Create Chart Structure

```bash
mkdir -p crossplane/helm-buckets/{templates,envs}
```

Directory structure:

```
crossplane/
├── providers/
│   ├── provider-family-aws.yaml
│   └── provider-config-dev.yaml
└── helm-buckets/
    ├── Chart.yaml
    ├── values.yaml
    ├── values-dev.yaml
    ├── templates/
    │   └── bucket.yaml
    └── envs/
        └── kustomization.yaml
```

### 3.2 Create Chart.yaml

File: `crossplane/helm-buckets/Chart.yaml`

```yaml
apiVersion: v2
name: crossplane-s3-buckets
description: Helm chart for managing S3 buckets via Crossplane
type: application
version: 1.0.0
appVersion: "1.0.0"
keywords:
  - crossplane
  - s3
  - aws
  - buckets
maintainers:
  - name: Maxim Garmonov
    email: devops@example.com
```

### 3.3 Create Default values.yaml

File: `crossplane/helm-buckets/values.yaml`

```yaml
# Default values for crossplane-s3-buckets
# This is a YAML-formatted file.

# Empty by default - override in environment-specific values files
buckets: []

# Default settings applied to all buckets (can be overridden per bucket)
defaults:
  versioning: true
  encryption: true
  publicAccessBlock: true
  region: us-east-1
  providerConfigRef: dev-us-east-1
```

### 3.4 Create Environment-Specific values-dev.yaml

File: `crossplane/helm-buckets/values-dev.yaml`

```yaml
# Development environment buckets
# Buckets are created in DEV AWS account

buckets:
  # Raw data ingestion bucket
  - name: data-platform-dev-raw
    region: us-east-1
    providerConfigRef: dev-us-east-1
    versioning: true
    encryption: true
    publicAccessBlock: true
    tags:
      team: data-platform
      env: dev
      owner: dev-team
      cost-center: engineering
    description: "Raw data ingestion for data platform"

  # Processed data bucket
  - name: data-platform-dev-processed
    region: us-east-1
    providerConfigRef: dev-us-east-1
    versioning: true
    encryption: true
    publicAccessBlock: true
    tags:
      team: data-platform
      env: dev
      owner: dev-team
      cost-center: engineering
    description: "Processed data for data platform"

  # ML models bucket (US West for closer to ML workloads)
  - name: ml-models-dev
    region: us-west-2
    providerConfigRef: dev-us-west-2
    versioning: true
    encryption: true
    publicAccessBlock: true
    tags:
      team: ml
      env: dev
      owner: ml-team
      cost-center: ml
    description: "ML model storage and versioning"

  # Analytics bucket
  - name: analytics-dev-results
    region: us-east-1
    providerConfigRef: dev-us-east-1
    versioning: false
    encryption: true
    publicAccessBlock: true
    tags:
      team: analytics
      env: dev
      owner: analytics-team
      cost-center: analytics
    description: "Analytics query results"
```

**For Production (Optional):** Create `values-prod.yaml`

```yaml
# Production environment buckets
# Buckets are created in PROD AWS account

buckets:
  - name: data-platform-prod-raw
    region: us-east-1
    providerConfigRef: prod-us-east-1
    versioning: true
    encryption: true
    publicAccessBlock: true
    tags:
      team: data-platform
      env: prod
      owner: prod-team
      cost-center: engineering
    description: "Production raw data ingestion"

  - name: data-platform-prod-processed
    region: us-east-1
    providerConfigRef: prod-us-east-1
    versioning: true
    encryption: true
    publicAccessBlock: true
    tags:
      team: data-platform
      env: prod
      owner: prod-team
      cost-center: engineering
    description: "Production processed data"
```

### 3.5 Create Helm Template

File: `crossplane/helm-buckets/templates/bucket.yaml`

```yaml
{{- range .Values.buckets }}
---
# S3 Bucket Resource
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: {{ .name }}
  namespace: default
  labels:
    app: crossplane-bucket
    team: {{ .tags.team | default "unknown" }}
    env: {{ .tags.env | default "unknown" }}
    managed-by: crossplane
    managed-by-helm: crossplane-s3-buckets
  annotations:
    description: {{ .description | default "S3 bucket managed by Crossplane" }}
spec:
  forProvider:
    region: {{ .region | default "us-east-1" }}
    tags:
      {{- toYaml .tags | nindent 6 }}
  providerConfigRef:
    name: {{ .providerConfigRef | default "dev-us-east-1" }}

{{- if .versioning }}
---
# Enable Versioning
apiVersion: s3.aws.upbound.io/v1beta1
kind: BucketVersioningV2
metadata:
  name: {{ .name }}-versioning
  namespace: default
  labels:
    app: crossplane-bucket
    bucket: {{ .name }}
spec:
  forProvider:
    bucketRef:
      name: {{ .name }}
    versioningConfiguration:
    - status: Enabled
  providerConfigRef:
    name: {{ .providerConfigRef | default "dev-us-east-1" }}
{{- end }}

{{- if .encryption }}
---
# Enable Server-Side Encryption
apiVersion: s3.aws.upbound.io/v1beta1
kind: BucketServerSideEncryptionConfigurationV2
metadata:
  name: {{ .name }}-encryption
  namespace: default
  labels:
    app: crossplane-bucket
    bucket: {{ .name }}
spec:
  forProvider:
    bucketRef:
      name: {{ .name }}
    rule:
    - applyServerSideEncryptionByDefault:
      - sseAlgorithm: AES256
  providerConfigRef:
    name: {{ .providerConfigRef | default "dev-us-east-1" }}
{{- end }}

{{- if .publicAccessBlock }}
---
# Block Public Access
apiVersion: s3.aws.upbound.io/v1beta1
kind: BucketPublicAccessBlock
metadata:
  name: {{ .name }}-public-block
  namespace: default
  labels:
    app: crossplane-bucket
    bucket: {{ .name }}
spec:
  forProvider:
    bucketRef:
      name: {{ .name }}
    blockPublicAcls: true
    blockPublicPolicy: true
    ignorePublicAcls: true
    restrictPublicBuckets: true
  providerConfigRef:
    name: {{ .providerConfigRef | default "dev-us-east-1" }}
{{- end }}
{{- end }}
```

---

## Step 4: Set Up Git Repository

### 4.1 Directory Structure

Create this structure in your Git repository:

```
infrastructure-repo/
├── README.md
├── crossplane/
│   ├── providers/
│   │   ├── provider-family-aws.yaml
│   │   └── provider-config-dev.yaml
│   └── helm-buckets/
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-dev.yaml
│       ├── values-prod.yaml (optional)
│       └── templates/
│           └── bucket.yaml
└── argocd/
    └── applications/
        ├── crossplane-buckets-dev.yaml
        └── crossplane-buckets-prod.yaml (optional)
```

### 4.2 Commit to Git

```bash
git add crossplane/
git commit -m "Add Crossplane provider config and Helm chart for S3 buckets"
git push origin main
```

---

## Step 5: Deploy with ArgoCD

### 5.1 Create ArgoCD Application

File: `argocd/applications/crossplane-buckets-dev.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-buckets-dev
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  # Source: Git repository
  source:
    repoURL: https://github.com/your-org/infrastructure-repo
    targetRevision: main
    path: crossplane/helm-buckets
    helm:
      releaseName: crossplane-buckets-dev
      valueFiles:
        - values-dev.yaml
      # Merge values with defaults
      values: |
        # Empty - all values come from values-dev.yaml

  # Destination: Current cluster, default namespace
  destination:
    server: https://kubernetes.default.svc
    namespace: default

  # Sync policy: automatic
  syncPolicy:
    automated:
      prune: true # Delete if removed from Git
      selfHeal: true # Reconcile if manually changed
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

---
# Optional: Production Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-buckets-prod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://github.com/your-org/infrastructure-repo
    targetRevision: main
    path: crossplane/helm-buckets
    helm:
      releaseName: crossplane-buckets-prod
      valueFiles:
        - values-prod.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: default

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### 5.2 Deploy ArgoCD Application

```bash
# Apply the ArgoCD application
kubectl apply -f argocd/applications/crossplane-buckets-dev.yaml

# Watch the application sync
kubectl get applications -n argocd -w

# Check application status
kubectl describe application crossplane-buckets-dev -n argocd
```

---

## Step 6: Developer Workflow

### 6.1 For Developers: Add a New Bucket

**Scenario:** Developer needs a new bucket for their team

**Steps:**

1. **Edit the values file** (e.g., `crossplane/helm-buckets/values-dev.yaml`):

```yaml
buckets:
  # ... existing buckets ...

  # NEW BUCKET - Add this section
  - name: my-team-data-bucket
    region: us-east-1
    providerConfigRef: dev-us-east-1
    versioning: true
    encryption: true
    publicAccessBlock: true
    tags:
      team: my-team
      env: dev
      owner: john-doe
      cost-center: engineering
    description: "Data bucket for my team"
```

2. **Commit and push**:

```bash
git add crossplane/helm-buckets/values-dev.yaml
git commit -m "Add bucket: my-team-data-bucket for data storage"
git push origin main
```

3. **ArgoCD will automatically detect the change** within 2-3 minutes

4. **Verify bucket creation**:

```bash
# Check if Crossplane resource is created
kubectl get bucket my-team-data-bucket

# Check status
kubectl describe bucket my-team-data-bucket

# Check AWS
aws s3 ls | grep my-team-data-bucket
```

### 6.2 Modify an Existing Bucket

Edit the bucket config in `values-dev.yaml` and commit. ArgoCD will automatically apply the changes.

### 6.3 Delete a Bucket

Remove the bucket entry from `values-dev.yaml` and commit. Crossplane will delete it from AWS.

```yaml
# Remove this section entirely:
# - name: old-bucket-name
#   region: us-east-1
#   ...

git add crossplane/helm-buckets/values-dev.yaml
git commit -m "Remove bucket: old-bucket-name"
git push origin main
```

---

## Step 7: Multi-Account & Multi-Region Setup

### 7.1 Understanding Multi-Account Setup

```
Your Kubernetes Cluster (Runs Crossplane)
│
├─ ProviderConfig: dev-us-east-1
│  └─ Credentials: aws-dev-credentials
│     └─ Creates buckets in: DEV account, US-EAST-1 region
│
├─ ProviderConfig: dev-us-west-2
│  └─ Credentials: aws-dev-credentials
│     └─ Creates buckets in: DEV account, US-WEST-2 region
│
├─ ProviderConfig: prod-us-east-1
│  └─ Credentials: aws-prod-credentials
│     └─ Creates buckets in: PROD account, US-EAST-1 region
│
└─ ProviderConfig: prod-eu-west-1
   └─ Credentials: aws-prod-credentials
      └─ Creates buckets in: PROD account, EU-WEST-1 region
```

### 7.2 Add More AWS Accounts

**Step 1: Create new AWS credentials secret**

```bash
# For Staging account
cat > /tmp/aws-staging-credentials.txt <<EOF
[default]
aws_access_key_id = YOUR_STAGING_ACCOUNT_ACCESS_KEY_ID
aws_secret_access_key = YOUR_STAGING_ACCOUNT_SECRET_ACCESS_KEY
EOF

kubectl create secret generic aws-staging-credentials \
  -n crossplane-system \
  --from-file=creds=/tmp/aws-staging-credentials.txt

rm /tmp/aws-staging-credentials.txt
```

**Step 2: Add ProviderConfigs**

Add to `crossplane/providers/provider-config-dev.yaml`:

```yaml
---
# Staging Account - US East 1
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: staging-us-east-1
  namespace: crossplane-system
spec:
  region: us-east-1
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-staging-credentials
      key: creds

---
# Staging Account - EU Central 1
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: staging-eu-central-1
  namespace: crossplane-system
spec:
  region: eu-central-1
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-staging-credentials
      key: creds
```

Apply:

```bash
kubectl apply -f crossplane/providers/provider-config-dev.yaml
```

**Step 3: Use in bucket definitions**

Update `values-dev.yaml`:

```yaml
buckets:
  - name: staging-team-bucket
    region: eu-central-1
    providerConfigRef: staging-eu-central-1 # Points to staging account
    versioning: true
    encryption: true
    tags:
      team: staging-team
      env: staging
```

### 7.3 Multi-Region Deployment for DR

Create a bucket in multiple regions for disaster recovery:

```yaml
buckets:
  # Primary region (US East 1)
  - name: data-platform-primary
    region: us-east-1
    providerConfigRef: dev-us-east-1
    versioning: true
    encryption: true
    tags:
      team: data-platform
      env: dev
      backup: primary

  # Backup region (EU West 1)
  - name: data-platform-backup-eu
    region: eu-west-1
    providerConfigRef: dev-eu-west-1 # Different region config
    versioning: true
    encryption: true
    tags:
      team: data-platform
      env: dev
      backup: secondary
```

### 7.4 Naming Convention for ProviderConfigs

Use this naming convention for clarity:

```
{environment}-{region}

Examples:
- dev-us-east-1
- dev-us-west-2
- dev-eu-west-1
- prod-us-east-1
- prod-eu-central-1
- staging-us-east-1
- staging-ap-southeast-1
```

---

## Troubleshooting

### Issue 1: Bucket Not Being Created

**Symptoms:** Bucket resource shows `SYNCED=False` or `READY=False`

**Diagnosis:**

```bash
# Check resource status and conditions
kubectl get bucket -o yaml

# Check detailed status
kubectl describe bucket <bucket-name>

# Check Crossplane logs
kubectl logs -n crossplane-system -l app=crossplane -f
```

**Common Causes:**

- ProviderConfig not found or incorrect name
- AWS credentials invalid
- Missing IAM permissions
- Provider pod not healthy

**Solution:**

```bash
# Verify ProviderConfig exists
kubectl get providerconfig -n crossplane-system

# Check if referenced ProviderConfig matches spec.providerConfigRef.name
kubectl get bucket <bucket-name> -o yaml | grep providerConfigRef

# Verify AWS credentials secret
kubectl get secret aws-dev-credentials -n crossplane-system
```

### Issue 2: ArgoCD Not Syncing

**Symptoms:** Application shows `OutOfSync` or sync fails

**Diagnosis:**

```bash
# Check ArgoCD application status
kubectl describe application crossplane-buckets-dev -n argocd

# Check ArgoCD application controller logs
kubectl logs -n argocd deployment/argocd-application-controller -f
```

**Common Causes:**

- Git credentials not configured
- Helm rendering errors
- Invalid YAML in values file

**Solution:**

```bash
# Validate Helm rendering locally
helm template crossplane-buckets-dev \
  crossplane/helm-buckets \
  -f crossplane/helm-buckets/values-dev.yaml

# Check git credentials in ArgoCD
kubectl get secret -n argocd | grep repository

# Manually trigger sync
argocd app sync crossplane-buckets-dev
```

### Issue 3: Provider Not Healthy

**Symptoms:** `kubectl get providers` shows `HEALTHY=False`

**Diagnosis:**

```bash
# Check provider pod
kubectl get pods -n crossplane-system

# Check provider logs
kubectl logs -n crossplane-system <provider-pod-name>
```

**Common Causes:**

- CRDs not installed
- Crossplane RBAC issues
- Provider pod crashing

**Solution:**

```bash
# Reinstall provider
kubectl delete provider upbound-provider-family-aws
kubectl apply -f crossplane/providers/provider-family-aws.yaml

# Wait for CRDs
sleep 60

# Check CRDs
kubectl get crds | grep aws
```

### Issue 4: Permission Denied on AWS

**Symptoms:** Error message in logs: "AccessDenied" or "UnauthorizedOperation"

**Diagnosis:**

```bash
# Check provider logs
kubectl logs -n crossplane-system <provider-pod-name> | grep -i permission

# Verify credentials
kubectl get secret aws-dev-credentials -n crossplane-system -o jsonpath='{.data.creds}' | base64 -d
```

**Common Causes:**

- IAM user/role lacks S3 permissions
- Wrong credentials in secret
- Credentials expired

**Solution:**

Ensure AWS credentials have these permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:CreateBucket",
        "s3:DeleteBucket",
        "s3:GetBucketPolicy",
        "s3:PutBucketPolicy",
        "s3:DeleteBucketPolicy",
        "s3:GetBucketVersioning",
        "s3:PutBucketVersioning",
        "s3:GetBucketEncryption",
        "s3:PutEncryptionConfiguration",
        "s3:GetBucketPublicAccessBlock",
        "s3:PutBucketPublicAccessBlock",
        "s3:GetBucketTagging",
        "s3:PutBucketTagging",
        "s3:ListBucket"
      ],
      "Resource": "*"
    }
  ]
}
```

### Issue 5: "Timeout waiting for ProviderConfig Informer to sync"

**Cause:** ProviderConfig CRD not yet available

**Solution:**

Wait for provider installation to complete:

```bash
# Watch provider status
kubectl get provider -w

# Wait until HEALTHY=True

# Then create ProviderConfig
kubectl apply -f crossplane/providers/provider-config-dev.yaml
```

---

## Useful Commands

### Check All Crossplane Resources

```bash
# All buckets
kubectl get bucket -A

# All ProviderConfigs
kubectl get providerconfig -A

# All providers
kubectl get providers

# All Crossplane CRDs
kubectl get crds | grep crossplane
```

### Monitor Bucket Creation

```bash
# Real-time watch
kubectl get bucket -w

# Detailed info
kubectl describe bucket <bucket-name>

# View YAML
kubectl get bucket <bucket-name> -o yaml
```

### Monitor ArgoCD Sync

```bash
# Watch application
kubectl get applications -n argocd -w

# Sync status
kubectl get application crossplane-buckets-dev -n argocd

# Sync logs
argocd app logs crossplane-buckets-dev -n argocd
```

### Debug Provider

```bash
# Provider status
kubectl describe provider upbound-provider-family-aws

# Provider logs
kubectl logs -n crossplane-system -l pkg.crossplane.io/provider

# Check CRDs
kubectl get crds | grep aws | wc -l
```

---

## Complete Checklist for Setup

- [ ] Crossplane core installed
- [ ] AWS family provider installed and healthy
- [ ] AWS credentials secrets created
- [ ] ProviderConfigs created for each account/region
- [ ] Helm chart created with templates
- [ ] Values files created (values-dev.yaml, values-prod.yaml)
- [ ] Git repository set up with correct structure
- [ ] ArgoCD application deployed
- [ ] First bucket created successfully
- [ ] Team trained on workflow

---

## Next Steps

1. **Scale buckets:** Add more buckets to `values-dev.yaml`
2. **Add environments:** Create `values-prod.yaml` for production
3. **Add regions:** Create ProviderConfigs for additional regions
4. **Automate:** Create script to add buckets without manual editing
5. **Monitor:** Set up CloudWatch monitoring for bucket activity
6. **Backup:** Enable S3 replication between regions
7. **Security:** Enable bucket logging and access control

---

## Additional Resources

- [Crossplane Documentation](https://docs.crossplane.io)
- [Upbound AWS Provider](https://marketplace.upbound.io/providers/upbound/provider-family-aws)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io)
- [Helm Documentation](https://helm.sh/docs)
