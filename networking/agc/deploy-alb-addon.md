# Deploy Application Gateway for Containers ALB Controller - AKS Add-On

> Based on [Quickstart: Deploy Application Gateway for Containers ALB Controller - AKS Add-on](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/quickstart-deploy-application-gateway-for-containers-alb-controller-addon)

## Prerequisites

### Set Environment Variables

```powershell
$AKS_NAME = "<your-cluster-name>"
$RESOURCE_GROUP = "<your-resource-group>"
$LOCATION = "<your-region>"
$SUBSCRIPTION_ID = "<your-subscription-id>"
```

### Sign In and Set Subscription

```powershell
az login
az account set --subscription $SUBSCRIPTION_ID
```

### Register Required Resource Providers

```powershell
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.NetworkFunction
az provider register --namespace Microsoft.ServiceNetworking
```

### Install Azure CLI Extensions

```powershell
az extension add --name alb
az extension add --name aks-preview
```

### Register Preview Features

```powershell
az feature register --namespace "Microsoft.ContainerService" --name "ManagedGatewayAPIPreview"
az feature register --namespace "Microsoft.ContainerService" --name "ApplicationLoadBalancerPreview"
```

## Set Up AKS Cluster with the AKS Add-On

> **Requirements:** Azure CNI or Azure CNI Overlay, workload identity enabled, supported AKS Kubernetes version, and a [region where AGC is available](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/overview#supported-regions).

### Existing Cluster

#### Enable Workload Identity

```powershell
az aks update -g $RESOURCE_GROUP -n $AKS_NAME --enable-oidc-issuer --enable-workload-identity --no-wait
```

#### Install ALB Controller Add-On

```powershell
az aks update --name $AKS_NAME --resource-group $RESOURCE_GROUP --enable-gateway-api --enable-application-load-balancer
```

## Verify the ALB Controller Installation

### Verify ALB Controller Pods

```bash
kubectl get pods -n kube-system | grep alb-controller
```

Expected output — two pods in `Running` state:

```
alb-controller-6648c5d5c-sdd9t   1/1     Running   0   4d6h
alb-controller-6648c5d5c-au234   1/1     Running   0   4d6h
```

### Verify GatewayClass

```bash
kubectl get gatewayclass azure-alb-external -o yaml
```

You should see a condition with `message: Valid GatewayClass` and `status: "True"`