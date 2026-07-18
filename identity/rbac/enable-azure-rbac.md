# Enable Azure RBAC for Kubernetes Authorization on AKS

> Based on [Use Microsoft Entra ID Authorization for the Kubernetes API in AKS](https://learn.microsoft.com/en-us/azure/aks/entra-id-authorization?tabs=azure-cli)

## Prerequisites

### Set Environment Variables

```powershell
$AKS_NAME = "aks-$(63533475)"
$RESOURCE_GROUP = "azure-rg"
$SUBSCRIPTION_ID = "<your-subscription-id>"
```

## Connect to the Cluster

```powershell
az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_NAME
```

## Deploy Sample Namespace and Application

```bash
kubectl apply -f sample-app.yaml
```

## Verify the Deployment

```bash
kubectl get pods -n sample-app -o wide
```

## Enable Azure RBAC on an Existing AKS Cluster

```powershell
az aks update --resource-group $RESOURCE_GROUP --name $AKS_NAME --enable-aad --enable-azure-rbac
```

Verify Azure RBAC is enabled:

```powershell
az aks show --resource-group $RESOURCE_GROUP --name $AKS_NAME --query "aadProfile.enableAzureRbac" -o tsv
```

## Assign Azure Kubernetes Service RBAC Admin to Current User

```powershell
$AKS_ID = az aks show --resource-group $RESOURCE_GROUP --name $AKS_NAME --query id -o tsv
$CURRENT_USER = az ad signed-in-user show --query id -o tsv

az role assignment create --role "Azure Kubernetes Service RBAC Admin" --assignee $CURRENT_USER --scope $AKS_ID
```

## Create and Assign a Custom Role Scoped to the Namespace

The custom role `AKS Namespace Reader` grants read access to pods and deployments in the `sample-app` namespace only.

### Create the Custom Role

```powershell
az role definition create --role-definition '@namespace-reader-role.json'
```

### Assign the Custom Role to a User

```powershell
$AKS_ID = az aks show --resource-group $RESOURCE_GROUP --name $AKS_NAME --query id -o tsv
$TARGET_USER = az ad signed-in-user show --query id -o tsv

az role assignment create --role "AKS Namespace Reader" --assignee $TARGET_USER --scope "$AKS_ID/namespaces/sample-app"
```

## Clean Up

### Disable Azure RBAC

```powershell
az aks update --resource-group $RESOURCE_GROUP --name $AKS_NAME --disable-azure-rbac
```

### Delete Role Assignment

```powershell
az role assignment list --scope $AKS_ID --query [].id -o tsv
az role assignment delete --ids <ASSIGNMENT-ID>
```

### Delete Custom Role Definition

```powershell
az role definition delete --name "AKS Namespace Reader"
```
