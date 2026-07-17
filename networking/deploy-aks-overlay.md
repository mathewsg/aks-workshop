# Deploy AKS Cluster with Azure CNI Overlay

## Prerequisites

Set your environment variables:

```powershell
$CLUSTER_NAME = "aks-$(63533420)"
$RESOURCE_GROUP = "azure-rg"
$REGION = "swedencentral"
```

## Deploy the Cluster

```bash
az aks create --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --location $REGION --network-plugin azure --network-plugin-mode overlay --pod-cidr 192.168.0.0/16 --generate-ssh-keys
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
