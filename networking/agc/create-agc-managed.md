# Create Application Gateway for Containers - Managed by ALB Controller

> Based on [Quickstart: Create Application Gateway for Containers managed by ALB Controller](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/quickstart-create-application-gateway-for-containers-managed-by-alb-controller)

## Prerequisites

Ensure ALB Controller is already deployed on your cluster. See [deploy-alb-addon.md](deploy-alb-addon.md).

### Set Environment Variables

```powershell
$AKS_NAME = "aks-$(63533475)"
$RESOURCE_GROUP = "azure-rg"
$IDENTITY_RESOURCE_NAME = "azure-alb-identity"
$ALB_SUBNET_NAME = "subnet-alb"
$SUBNET_ADDRESS_PREFIX = "10.225.0.0/24"
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

### Create Namespace

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: alb-test-infra
EOF
```

### Create the ApplicationLoadBalancer Resource

> The ALB Controller will create the Application Gateway for Containers resource in Azure and an association to the specified subnet.

```bash
kubectl apply -f - <<EOF
apiVersion: alb.networking.azure.io/v1
kind: ApplicationLoadBalancer
metadata:
  name: alb-test
  namespace: alb-test-infra
spec:
  associations:
  - $ALB_SUBNET_ID
EOF
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
