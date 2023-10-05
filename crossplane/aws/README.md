# Crossplane Environment Composition for platform building

This tutorial focus on creating a basic AWS EKS cluster

This Crossplane Composite resource creates the following resources:
- EKS Cluster
- NodeGroup
<!-- - Helm Provider Config
- Helm Release inside the created cluster -->

## Installation

In an EKS Kubernetes Cluster install the following components.

Let's install [Upbound's Crossplane CLI](https://docs.upbound.io/cli/):

```
curl -sL "https://cli.upbound.io" | sh

```

Download [Upbound Universal Crossplane](https://docs.upbound.io/uxp/install/?ref=upbound-blog#install-upbound-universal-crossplane)

```
# Make sure your ~/.kube/config file points to your cluster

up uxp install


## Install & Configure AWS provider

### Generate an AWS key-pair file

Create a text file containing the AWS account aws_access_key_id and aws_secret_access_key. The AWS documentation provides information on how to generate these keys.

```
[default]
aws_access_key_id = <aws_access_key>
aws_secret_access_key = <aws_secret_key>
```
Save this text file as aws-credentials.txt

### Create a Kubernetes secret with AWS credentials
Use kubectl create secret -n upbound-system to generate a Kubernetes secret object inside the Kubernetes cluster.

```
kubectl create secret \
generic aws-secret \
-n upbound-system \
--from-file=creds=./aws-credentials.txt
```

### Create a ProviderConfig

Create a ProviderConfig Kubernetes configuration file to attach the AWS credentials to the installed official provider.

```
cat <<EOF | kubectl apply -f -
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: upbound-system
      name: aws-secret
      key: creds
EOF
```

## Install Environment Composite Resource (XRD)

```
kubectl apply -f team-d-env-aws.yaml
```