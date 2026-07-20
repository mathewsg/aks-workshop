# Create Application Gateway for Containers - Managed by ALB Controller

> Based on [Quickstart: Create Application Gateway for Containers managed by ALB Controller](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/quickstart-create-application-gateway-for-containers-managed-by-alb-controller)

## Prerequisites

### Set Environment Variables

```powershell
$AKS_NAME = "aks-$(63533475)"
$RESOURCE_GROUP = "azure-rg"
$LOCATION = "swedencentral"
$IDENTITY_RESOURCE_NAME = "azure-alb-identity"
$FEDERATED_IDENTITY_NAME = "azure-alb-identity"
$CONTROLLER_NAMESPACE = "azure-alb-system"
$ALB_SUBNET_NAME = "subnet-alb"
$SUBNET_ADDRESS_PREFIX = "10.225.0.0/24"
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

## Deploy ALB Controller on Existing Cluster

### Enable Workload Identity

> **Requirements:** Azure CNI or Azure CNI Overlay, workload identity enabled, supported AKS Kubernetes version, and a [region where AGC is available](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/overview#supported-regions).

```powershell
az aks update -g $RESOURCE_GROUP -n $AKS_NAME --enable-oidc-issuer --enable-workload-identity --no-wait
```

### Create Managed Identity and Federate with AKS

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

### Install ALB Controller via Helm

```powershell
$HELM_NAMESPACE = "azure-alb-system"

az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_NAME

$CLIENT_ID = az identity show -g $RESOURCE_GROUP -n $IDENTITY_RESOURCE_NAME --query clientId -o tsv

helm install alb-controller oci://mcr.microsoft.com/application-lb/charts/alb-controller `
    --create-namespace `
    --namespace $HELM_NAMESPACE `
    --version 1.11.3 `
    --set albController.namespace=$CONTROLLER_NAMESPACE `
    --set albController.podIdentity.clientID=$CLIENT_ID
```

### Verify ALB Controller Pods

```bash
kubectl get pods -n azure-alb-system
```

### Verify GatewayClass

```bash
kubectl get gatewayclass azure-alb-external -o yaml
```

## Prepare Subnet for Application Gateway for Containers

```powershell
$MC_RESOURCE_GROUP = az aks show --name $AKS_NAME --resource-group $RESOURCE_GROUP --query "nodeResourceGroup" -o tsv
$CLUSTER_SUBNET_ID = az vmss list --resource-group $MC_RESOURCE_GROUP --query '[0].virtualMachineProfile.networkProfile.networkInterfaceConfigurations[0].ipConfigurations[0].subnet.id' -o tsv
$VNET_ID = ($CLUSTER_SUBNET_ID -split '/subnets/')[0]
$VNET_NAME = ($VNET_ID -split '/')[-1]
$VNET_RESOURCE_GROUP = ($VNET_ID -split '/')[4]

# Create subnet with delegation for Application Gateway for Containers
az network vnet subnet create `
    --resource-group $VNET_RESOURCE_GROUP `
    --vnet-name $VNET_NAME `
    --name $ALB_SUBNET_NAME `
    --address-prefixes $SUBNET_ADDRESS_PREFIX `
    --delegations "Microsoft.ServiceNetworking/trafficControllers"

$ALB_SUBNET_ID = az network vnet subnet show --name $ALB_SUBNET_NAME --resource-group $VNET_RESOURCE_GROUP --vnet-name $VNET_NAME --query id -o tsv
```

## Delegate Permissions to Managed Identity

```powershell
$mcResourceGroupId = az group show --name $MC_RESOURCE_GROUP --query id -o tsv
$principalId = az identity show -g $RESOURCE_GROUP -n $IDENTITY_RESOURCE_NAME --query principalId -o tsv

# Delegate AppGw for Containers Configuration Manager role to MC resource group
az role assignment create --assignee-object-id $principalId --assignee-principal-type ServicePrincipal --scope $mcResourceGroupId --role "fbc52c3f-28ad-4303-a892-8a056630b8f1"

# Delegate Network Contributor role to the ALB subnet
az role assignment create --assignee-object-id $principalId --assignee-principal-type ServicePrincipal --scope $ALB_SUBNET_ID --role "4d97b98b-1d4f-4787-a291-c67834d212e7"
```

## Create ApplicationLoadBalancer Kubernetes Resource

> The ALB Controller will create the Application Gateway for Containers resource in Azure and an association to the specified subnet.

Replace the placeholder in the YAML file with the actual subnet ID and apply:

```powershell
(Get-Content alb-resource.yaml) -replace 'ALB_SUBNET_ID_PLACEHOLDER', $ALB_SUBNET_ID | kubectl apply -f -
```

## Validate Creation of Application Gateway for Containers

It can take 5-6 minutes for the resources to be created.

```bash
kubectl get applicationloadbalancer alb-test -n alb-test-infra -o yaml -w
```

Expected output when provisioning is complete:

```yaml
status:
  conditions:
  - lastTransitionTime: "2023-06-19T21:03:29Z"
    message: Valid Application Gateway for Containers resource
    observedGeneration: 1
    reason: Accepted
    status: "True"
    type: Accepted
  - lastTransitionTime: "2023-06-19T21:03:29Z"
    message: alb-id=/subscriptions/xxx/resourceGroups/yyy/providers/Microsoft.ServiceNetworking/trafficControllers/alb-zzz
    observedGeneration: 1
    reason: Ready
    status: "True"
    type: Deployment
```

## Next Steps

Try out sample scenarios:

- [Backend MTLS](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/how-to-backend-mtls-gateway-api?tabs=alb-managed)
- [SSL/TLS Offloading](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/how-to-ssl-offloading-gateway-api?tabs=alb-managed)
- [Traffic Splitting / Weighted Round Robin](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/how-to-traffic-splitting-gateway-api?tabs=alb-managed)
