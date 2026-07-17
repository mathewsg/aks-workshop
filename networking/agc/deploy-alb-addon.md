# Deploy Application Gateway for Containers ALB Controller - Helm

> Based on [Quickstart: Deploy Application Gateway for Containers ALB Controller - Helm](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/quickstart-deploy-application-gateway-for-containers-alb-controller-helm)

## Prerequisites

### Set Environment Variables

```powershell
$AKS_NAME = "aks-$(63533475)"
$RESOURCE_GROUP = "azure-rg"
$LOCATION = "swedencentral"
$SUBSCRIPTION_ID = "<your-subscription-id>"
$IDENTITY_RESOURCE_NAME = "azure-alb-identity"
$FEDERATED_IDENTITY_NAME = "azure-alb-identity"
$CONTROLLER_NAMESPACE = "azure-alb-system"
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
```

### Install Helm

```powershell
winget install helm.helm
```

### Enable Workload Identity on Existing Cluster

> **Requirements:** Azure CNI or Azure CNI Overlay, workload identity enabled, supported AKS Kubernetes version, and a [region where AGC is available](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/overview#supported-regions).

```powershell
az aks update -g $RESOURCE_GROUP -n $AKS_NAME --enable-oidc-issuer --enable-workload-identity --no-wait
```

## Create Managed Identity and Federate with AKS

```powershell
$mcResourceGroup = az aks show --resource-group $RESOURCE_GROUP --name $AKS_NAME --query "nodeResourceGroup" -o tsv
$mcResourceGroupId = az group show --name $mcResourceGroup --query id -o tsv

# Create the managed identity
az identity create --resource-group $RESOURCE_GROUP --name $IDENTITY_RESOURCE_NAME
$principalId = az identity show -g $RESOURCE_GROUP -n $IDENTITY_RESOURCE_NAME --query principalId -o tsv

# Wait for identity replication
Start-Sleep -Seconds 60

# Assign Reader role on the MC resource group
az role assignment create --assignee-object-id $principalId --assignee-principal-type ServicePrincipal --scope $mcResourceGroupId --role "acdd72a7-3385-48ef-bd42-f606fba81ae7"

# Set up federation with AKS OIDC issuer
$AKS_OIDC_ISSUER = az aks show -n $AKS_NAME -g $RESOURCE_GROUP --query "oidcIssuerProfile.issuerUrl" -o tsv

az identity federated-credential create --name $FEDERATED_IDENTITY_NAME `
    --identity-name $IDENTITY_RESOURCE_NAME `
    --resource-group $RESOURCE_GROUP `
    --issuer $AKS_OIDC_ISSUER `
    --subject "system:serviceaccount:${CONTROLLER_NAMESPACE}:alb-controller-sa"
```

## Install ALB Controller via Helm

```powershell
$HELM_NAMESPACE = "default"

az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_NAME

$CLIENT_ID = az identity show -g $RESOURCE_GROUP -n $IDENTITY_RESOURCE_NAME --query clientId -o tsv

helm install alb-controller oci://mcr.microsoft.com/application-lb/charts/alb-controller `
    --namespace $HELM_NAMESPACE `
    --version 1.11.3 `
    --set albController.namespace=$CONTROLLER_NAMESPACE `
    --set albController.podIdentity.clientID=$CLIENT_ID
```

## Verify the ALB Controller Installation

### Verify ALB Controller Pods

```bash
kubectl get pods -n azure-alb-system
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

You should see a condition with `message: Valid GatewayClass` and `status: "True"`.

## Uninstall ALB Controller

```powershell
helm uninstall alb-controller
kubectl delete ns azure-alb-system
kubectl delete gatewayclass azure-alb-external
```