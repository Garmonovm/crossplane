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


# CompositionResourceDefinition for deployment and service
kubectl apply -f xrd.yaml
k get -f xrd.yaml
## Function to use the composition
kubectl apply -f function.yaml
k get -f function.yaml

## Create composition
kubectl apply -f composition.yaml

## App
kubectl apply -f app.yaml


##### Descriptions
Summary Table
File	Purpose
xrd.yaml	Defines the custom resource type/schema
composition.yaml	Maps custom resource to real resources
function.yaml	(Optional) Advanced logic/patching
app.yaml	Creates an instance of your custom resource

The XRD (CompositeResourceDefinition) is what actually creates the new custom resource type in Kubernetes (like App).
Without the XRD, Kubernetes doesn’t know your resource exists—so you can’t create or use it in app.yaml.
The Composition only tells Crossplane how to build resources for your custom type, but it needs the XRD to know what that type is.
The Function adds logic, but it also depends on the XRD-defined resource.
##


kubectl apply -f provider-s3.yaml
kubectl get providers


kubectl create -f bucket.yaml
kubectl get buckets.s3.aws.m.upbound.io


k delete bucket crossplane-bucket-wd59s