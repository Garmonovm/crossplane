# Crossplane SQS Queues Helm Chart

Helm chart for managing AWS SQS queues with Dead Letter Queues (DLQ) using Crossplane.

## Prerequisites

- Crossplane core installed
- AWS Family Provider (crossplane-contrib/provider-family-aws) v2.2.0+
- ClusterProviderConfig configured with AWS credentials
- ArgoCD (optional, for GitOps deployment)

## Features

- ✅ SQS Queue creation with configurable parameters
- ✅ Dead Letter Queue (DLQ) support
- ✅ QueueRedrivePolicy for automatic DLQ linking
- ✅ Long polling configuration
- ✅ Custom retention and visibility timeout
- ✅ Tag-based organization
- ✅ Multi-environment support (dev, staging, prod)

## Chart Structure

```
helm-sqs/
├── Chart.yaml
├── values.yaml              # Default values
├── values-dev.yaml          # Development environment
└── templates/
    └── queue.yaml           # Queue, DLQ, and RedrivePolicy templates
```

## Usage

### Local Helm Install

```bash
# Install with dev values
helm install crossplane-sqs-dev ./helm-sqs -f ./helm-sqs/values-dev.yaml

# Upgrade
helm upgrade crossplane-sqs-dev ./helm-sqs -f ./helm-sqs/values-dev.yaml

# Uninstall
helm uninstall crossplane-sqs-dev
```

### ArgoCD Deployment

```bash
# Deploy ArgoCD application
kubectl apply -f argocd/applications/crossplane-sqs-dev.yaml

# Check sync status
kubectl get application crossplane-sqs-dev -n argocd
```

## Configuration

### Queue Parameters

| Parameter                  | Description                | Default            |
| -------------------------- | -------------------------- | ------------------ |
| `name`                     | Queue name                 | Required           |
| `region`                   | AWS region                 | `eu-central-1`     |
| `providerConfigRef`        | ClusterProviderConfig name | `dev-eu-central-1` |
| `visibilityTimeoutSeconds` | Visibility timeout         | `30`               |
| `messageRetentionSeconds`  | Message retention period   | `345600` (4 days)  |
| `receiveWaitTimeSeconds`   | Long polling wait time     | `0` (disabled)     |
| `hasDLQ`                   | Enable DLQ                 | `false`            |
| `maxReceiveCount`          | Max retries before DLQ     | `3`                |

### Example Configuration

```yaml
queues:
  - name: my-app-queue
    region: eu-central-1
    providerConfigRef: dev-eu-central-1
    visibilityTimeoutSeconds: 60
    messageRetentionSeconds: 345600
    receiveWaitTimeSeconds: 20 # Enable long polling
    hasDLQ: true
    maxReceiveCount: 3
    tags:
      team: platform
      env: dev
```

## Monitoring

### Check Queue Status

```bash
# List all queues
kubectl get queue.sqs.aws.m.upbound.io

# Check specific queue
kubectl describe queue.sqs.aws.m.upbound.io app-events-dev

# Check DLQ
kubectl describe queue.sqs.aws.m.upbound.io app-events-dev-dlq

# Check redrive policy
kubectl get queueredrivepolicy.sqs.aws.m.upbound.io
```

### Verify in AWS

```bash
# List queues
aws sqs list-queues --region eu-central-1

# Get queue URL
aws sqs get-queue-url --queue-name app-events-dev --region eu-central-1

# Check queue attributes
aws sqs get-queue-attributes \
  --queue-url https://sqs.eu-central-1.amazonaws.com/ACCOUNT_ID/app-events-dev \
  --attribute-names All
```

## Troubleshooting

### Queue Not Created

**Check Crossplane provider:**

```bash
kubectl get providers
kubectl logs -n crossplane-system -l pkg.crossplane.io/provider=provider-family-aws
```

**Check ClusterProviderConfig:**

```bash
kubectl get clusterproviderconfig dev-eu-central-1 -o yaml
```

### DLQ Not Linked

The QueueRedrivePolicy uses `queueUrlSelector` which requires:

1. Main queue must have `queue-type: main` label
2. DLQ must exist and be ready
3. AWS Account ID must be correct in ARN

**Note:** You may need to manually update the redrivePolicy with your AWS Account ID.

## Important Notes for Minikube

⚠️ **Minikube Limitations:**

- This setup works on Minikube for learning/testing
- Real AWS credentials required (ClusterProviderConfig)
- SQS queues are created in actual AWS account
- Consider costs for queue usage

## Cleanup

### Remove Queues via Git

Remove queue entries from `values-dev.yaml` and commit. ArgoCD will automatically delete them.

### Manual Cleanup

```bash
# Delete specific queue (will also delete DLQ and redrive policy)
kubectl delete queue.sqs.aws.m.upbound.io app-events-dev

# Delete all queues
kubectl delete queue.sqs.aws.m.upbound.io -l managed-by-helm=crossplane-sqs-queues
```

⚠️ **Finalizers:** Kubernetes resources won't be deleted until AWS resources are cleaned up. This is expected behavior.

## Next Steps

1. Monitor DLQ message count (set up CloudWatch alarms)
2. Create IAM roles for queue access (see helm-iam chart)
3. Configure queue encryption with KMS
4. Set up cross-region replication for DR
5. Add visibility timeout based on processing time

## References

- [Crossplane AWS Provider](https://marketplace.upbound.io/providers/upbound/provider-family-aws)
- [AWS SQS Documentation](https://docs.aws.amazon.com/sqs/)
- [SQS Dead Letter Queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
