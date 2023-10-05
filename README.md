# Glue things together

Keeping an eye on the CNCF ecosystem is a full time job, but if you are serious about adopting Kubernetes you want to stay up to date to make sure that you levarage what these projects are doing, so you don't need to build your in-house solution. 

In this section will we look at creating our Platform using a set of tools that accomodate different teams with different expectations. 

For this we will install the following tools into our Kubernetes Cluster that we will call the Platform Cluster: 

- [Backstage](https://backstage.io)
- [Crossplane](https://crossplane.io)
- [ArgoCD](https://argo-cd.readthedocs.io/en/stable/)

These three very popular tools provide a set of key features that enable us to build more complex platforms on top of Kubernetes. 

```
cat <<EOF | kind create cluster --name platform --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 31080 # expose port 31380 of the node to port 80 on the host, later to be use by kourier or contour ingress
    listenAddress: 127.0.0.1
    hostPort: 80
EOF
```

### Internal Developer Portal

In order to keep track of all of the new services, applications, documentation, resources, infrastructure, etc. that your teams will continue to create throughout the lifecycles of your intiatives - having an internal developer portal may be a very valuable addition to your platform.

Now let's deploy Backstage onto the kind cluster

```
kubectl create namespace backstage
kubectl create serviceaccount backstage -n backstage

kubectl apply -f backstage/kubernetes/rbac.yaml
kubectl apply -f backstage/kubernetes/backstage-secret.yaml
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
kubectl create configmap backstage-cm -n backstage --from-literal=ENDPOINT=$APISERVER

kubectl apply -f backstage/kubernetes/backstage-service.yaml
kubectl apply -f backstage/kubernetes/backstage.yaml
```

We'll port forward in order to get to the Backstage UI

```
kubectl port-forward --namespace=backstage svc/backstage 5434:80
```

Access http://localhost:5434/ on your browser and do some initial exploring.

<!-- Next we're going to integrate Backstage with dapr resources on our Kubernetes cluster:

```
kubectl patch deployment/dapr-sidecar-injector -n dapr-system \
-p '{"spec":{"template":{"metadata":{"labels":{"backstage.io/kubernetes-id":"sample-app"}}}}}'

kubectl patch deployment/dapr-dashboard -n dapr-system \
-p '{"spec":{"template":{"metadata":{"labels":{"backstage.io/kubernetes-id":"sample-app"}}}}}'

kubectl patch deployment/dapr-sentry -n dapr-system \
-p '{"spec":{"template":{"metadata":{"labels":{"backstage.io/kubernetes-id":"sample-app"}}}}}'

kubectl patch deployment/dapr-operator -n dapr-system \
-p '{"spec":{"template":{"metadata":{"labels":{"backstage.io/kubernetes-id":"sample-app"}}}}}'
``` -->

### ArgoCD

1.  Deploy ArgoCD Server to kind cluster

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

2. Run the following command to get the ArgoCD local admim password (in the case of Github auth failures).

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

3. We'll port forward in order to get to the ArgoCD UI

```
kubectl port-forward --namespace=argocd svc/argocd-server 5435:80
```

Access http://localhost:5435/ on your browser, use "admin" for the usernmae and the previously pulled admin secret for the password. 

## Crossplane Environment Composition

This tutorial focus on creating a basic AWS EKS cluster

This Crossplane Composite resource creates the following resources:
- EKS Cluster
- NodeGroup

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

```
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/upbound/provider-aws:latest
EOF
```

```
cat <<EOF | kubectl apply -f -                  
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-helm
spec:
  package: crossplanecontrib/provider-helm:master
EOF
```

`kubectl get provider`

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


5. Enable custom resource tracking on the ArgoCD configmap

```bash
kubectl edit configmap argocd-cm -n argocd
```

```
apiVersion: v1
data:
  application.resourceTrackingMethod: annotation
kind: ConfigMap
```

6. Apply Crossplane configuration through ArgoCD

```bash
kubectl apply -f argocd/applicationset/crossplane/configuration/crossplane-config.yaml
```