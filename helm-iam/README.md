# Crossplane IAM Roles Helm Chart

Helm chart for managing AWS IAM roles and policies using Crossplane.

## Prerequisites

- Crossplane core installed
- AWS Family Provider (crossplane-contrib/provider-family-aws) v2.2.0+
- ClusterProviderConfig configured with AWS credentials
- ArgoCD (optional, for GitOps deployment)

## Features

- ✅ IAM Role creation with custom trust policies
- ✅ Inline IAM policies
- ✅ RolePolicy attachment using selectors
- ✅ Tag-based organization
- ✅ Multi-environment support (dev, staging, prod)

## Chart Structure

```
helm-iam/
├── Chart.yaml
├── values.yaml              # Default values
├── values-dev.yaml          # Development environment
└── templates/
    └── role.yaml            # Role and RolePolicy templates
```

## Usage

### Local Helm Install

```bash
# Install with dev values
helm install crossplane-iam-dev ./helm-iam -f ./helm-iam/values-dev.yaml

# Upgrade
helm upgrade crossplane-iam-dev ./helm-iam -f ./helm-iam/values-dev.yaml

# Uninstall
helm uninstall crossplane-iam-dev
```

### ArgoCD Deployment

```bash
# Deploy ArgoCD application
kubectl apply -f argocd/applications/crossplane-iam-dev.yaml

# Check sync status
kubectl get application crossplane-iam-dev -n argocd
```

## Configuration

### Role Parameters

| Parameter             | Description                    | Default            |
| --------------------- | ------------------------------ | ------------------ |
| `name`                | Role name                      | Required           |
| `providerConfigRef`   | ClusterProviderConfig name     | `dev-eu-central-1` |
| `description`         | Role description               | Required           |
| `assumeRolePolicy`    | Trust policy JSON              | Required           |
| `maxSessionDuration`  | Max session duration (seconds) | `3600` (1 hour)    |
| `inlinePolicy.name`   | Policy name                    | Optional           |
| `inlinePolicy.policy` | Policy JSON                    | Optional           |

### Example Configuration

```yaml
roles:
  - name: sqs-processor-dev
    providerConfigRef: dev-eu-central-1
    description: "IAM role for SQS processing"
    maxSessionDuration: 3600
    assumeRolePolicy: |
      {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Effect": "Allow",
            "Principal": {
              "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
          }
        ]
      }
    inlinePolicy:
      name: sqs-access
      policy: |
        {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Effect": "Allow",
              "Action": ["sqs:ReceiveMessage"],
              "Resource": "arn:aws:sqs:*:*:my-queue"
            }
          ]
        }
    tags:
      team: platform
      env: dev
```

## IRSA (IAM Roles for Service Accounts)

⚠️ **Important for Minikube Users:**

The current trust policy uses `ec2.amazonaws.com` as principal, which is suitable for:

- EC2 instances
- Local development/testing on Minikube
- Understanding IAM role concepts

**For EKS (Production)**, you need IRSA with OIDC provider:

```yaml
assumeRolePolicy: |
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/oidc.eks.REGION.amazonaws.com/id/OIDC_ID"
        },
        "Action": "sts:AssumeRoleWithWebIdentity",
        "Condition": {
          "StringEquals": {
            "oidc.eks.REGION.amazonaws.com/id/OIDC_ID:sub": "system:serviceaccount:NAMESPACE:SERVICE_ACCOUNT"
          }
        }
      }
    ]
  }
```

### Converting to IRSA

1. Get your EKS OIDC provider:

```bash
aws eks describe-cluster --name YOUR_CLUSTER --query "cluster.identity.oidc.issuer"
```

2. Update `assumeRolePolicy` in `values-dev.yaml`

3. Annotate Kubernetes ServiceAccount:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sqs-processor
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/sqs-processor-dev
```

## Monitoring

### Check Role Status

```bash
# List all roles
kubectl get role.iam.aws.m.upbound.io

# Check specific role
kubectl describe role.iam.aws.m.upbound.io sqs-processor-dev

# Check inline policies
kubectl get rolepolicy.iam.aws.m.upbound.io
```

### Verify in AWS

```bash
# List roles
aws iam list-roles | grep sqs-processor

# Get role details
aws iam get-role --role-name sqs-processor-dev

# List inline policies
aws iam list-role-policies --role-name sqs-processor-dev

# Get policy document
aws iam get-role-policy \
  --role-name sqs-processor-dev \
  --policy-name sqs-access-policy
```

## Troubleshooting

### Role Not Created

**Check Crossplane provider:**

```bash
kubectl get providers
kubectl logs -n crossplane-system -l pkg.crossplane.io/provider=provider-family-aws
```

**Check ClusterProviderConfig:**

```bash
kubectl get clusterproviderconfig dev-eu-central-1 -o yaml
```

### Policy Not Attached

The RolePolicy uses `roleSelector` which requires:

1. Role must exist and be ready
2. Labels must match (app, team, env)
3. Policy JSON must be valid

**Debug:**

```bash
kubectl describe rolepolicy.iam.aws.m.upbound.io sqs-processor-dev-sqs-access-policy
```

## Security Best Practices

1. **Principle of Least Privilege**: Grant only necessary permissions
2. **Resource-specific ARNs**: Avoid wildcards in Resource fields
3. **Condition Keys**: Add conditions to restrict usage
4. **Audit**: Enable CloudTrail for role usage monitoring
5. **Rotation**: Regularly review and update policies

## Common Use Cases

### SQS Access Role

```yaml
- name: sqs-consumer
  assumeRolePolicy: # EC2 or IRSA trust policy
  inlinePolicy:
    name: sqs-access
    policy: |
      {
        "Statement": [{
          "Effect": "Allow",
          "Action": [
            "sqs:ReceiveMessage",
            "sqs:DeleteMessage",
            "sqs:GetQueueAttributes"
          ],
          "Resource": "arn:aws:sqs:*:*:my-queue*"
        }]
      }
```

### S3 Access Role

```yaml
- name: s3-reader
  inlinePolicy:
    policy: |
      {
        "Statement": [{
          "Effect": "Allow",
          "Action": ["s3:GetObject", "s3:ListBucket"],
          "Resource": [
            "arn:aws:s3:::my-bucket",
            "arn:aws:s3:::my-bucket/*"
          ]
        }]
      }
```

## Cleanup

### Remove Roles via Git

Remove role entries from `values-dev.yaml` and commit. ArgoCD will automatically delete them.

### Manual Cleanup

```bash
# Delete specific role (will also delete inline policies)
kubectl delete role.iam.aws.m.upbound.io sqs-processor-dev

# Delete all roles
kubectl delete role.iam.aws.m.upbound.io -l managed-by-helm=crossplane-iam-roles
```

⚠️ **Finalizers:** Kubernetes resources won't be deleted until AWS resources are cleaned up.

## Next Steps

1. Convert to IRSA for EKS deployment
2. Add managed policy attachments (RolePolicyAttachment)
3. Set up permission boundaries
4. Configure session tags for auditing
5. Enable AWS CloudTrail logging

## References

- [Crossplane AWS Provider](https://marketplace.upbound.io/providers/upbound/provider-family-aws)
- [AWS IAM Roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)
- [EKS IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
