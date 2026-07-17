# Deploy AKS Cluster with Azure CNI Overlay

## Prerequisites

Set your environment variables:

```powershell
$CLUSTER_NAME = "aks-$(63533475)"
$RESOURCE_GROUP = "azure-rg"
$LOCATION = "swedencentral"
$VM_SKU="Standard_D2as_v5"
```

## Deploy the Cluster

```bash
az aks create --node-count 2 --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --node-vm-size $VM_SKU --location $LOCATION --network-plugin azure --network-plugin-mode overlay --pod-cidr 192.168.0.0/16 --generate-ssh-keys
```

## Connect to the Cluster

```bash
az aks get-credentials --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP
```

## List Pods

```bash
kubectl get pods --all-namespaces
```

## Deploy Sample Nginx Application

```bash
kubectl apply -f nginx-deployment.yaml
```

## Expose as LoadBalancer Service

```bash
kubectl apply -f nginx-service.yaml
```

## Verify the Deployment and Service

```bash
kubectl get pods
kubectl get service nginx
```
