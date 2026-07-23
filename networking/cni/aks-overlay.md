# Deploy AKS Cluster with Azure CNI Overlay

## Prerequisites

Set your environment variables:

```powershell
$AKS_NAME = "aks-$(63533475)"
$RESOURCE_GROUP = "azure-rg"
$LOCATION = "swedencentral"
$VM_SKU="Standard_D2as_v5"
```

## Create Resource Group

```powershell
az group create --name $RESOURCE_GROUP --location $LOCATION
```

## Deploy the Cluster

```bash
az aks create --node-count 2 --name $AKS_NAME --resource-group $RESOURCE_GROUP --node-vm-size $VM_SKU --location $LOCATION --network-plugin azure --network-plugin-mode overlay --pod-cidr 192.168.0.0/16 --generate-ssh-keys
```

## Connect to the Cluster

```bash
az aks get-credentials --name $AKS_NAME --resource-group $RESOURCE_GROUP
```


## Deploy Sample Nginx Application

```bash
kubectl apply -f nginx-deployment.yaml
```

## List Pods in nginx-app Namespace

```bash
kubectl get pods -n nginx-app -o wide
```

## Expose as LoadBalancer Service

```bash
kubectl apply -f nginx-service.yaml
```

## Verify the Deployment and Service

```bash
kubectl get service nginx -n nginx-app
```
