# AKS Monitoring - Container Insights

## Prerequisites

### Set Environment Variables

```powershell
$AKS_NAME = "aks-mon"
$RESOURCE_GROUP = "azure-rg"
$LOCATION = "swedencentral"
$VM_SKU = "Standard_D2as_v5"
$LOG_ANALYTICS_WORKSPACE = "law-aks-monitoring"
```

### Create Resource Group

```powershell
az group create --name $RESOURCE_GROUP --location $LOCATION
```

## Deploy AKS Cluster Without Monitoring

```powershell
az aks create `
    --name $AKS_NAME `
    --resource-group $RESOURCE_GROUP `
    --location $LOCATION `
    --node-count 2 `
    --node-vm-size $VM_SKU `
    --network-plugin azure `
    --network-plugin-mode overlay `
    --pod-cidr 192.168.0.0/16 `
    --generate-ssh-keys
```

### Connect to the Cluster

```powershell
az aks get-credentials --name $AKS_NAME --resource-group $RESOURCE_GROUP
```

## Deploy Log Analytics Workspace

```powershell
az monitor log-analytics workspace create `
    --resource-group $RESOURCE_GROUP `
    --workspace-name $LOG_ANALYTICS_WORKSPACE `
    --location $LOCATION

$LOG_ANALYTICS_WORKSPACE_ID = az monitor log-analytics workspace show `
    --resource-group $RESOURCE_GROUP `
    --workspace-name $LOG_ANALYTICS_WORKSPACE `
    --query id -o tsv
```

## Enable Container Insights

```powershell
az aks enable-addons `
    --addon monitoring `
    --name $AKS_NAME `
    --resource-group $RESOURCE_GROUP `
    --workspace-resource-id $LOG_ANALYTICS_WORKSPACE_ID
```

## Verify AMA (Azure Monitor Agent) Pods

After enabling Container Insights, Azure Monitor Agent pods are deployed to the cluster:

```bash
kubectl get pods -n kube-system | Select-String "ama-"
```

Expected output — you should see `ama-logs` and `ama-metrics` pods running:

```
ama-logs-xxxxx          1/1     Running   0   2m
ama-logs-rs-xxxxx       1/1     Running   0   2m
ama-metrics-xxxxx       1/1     Running   0   2m
ama-metrics-node-xxxxx  1/1     Running   0   2m
```

To see all monitoring-related pods:

```bash
kubectl get pods -n kube-system -l component=ama-logs
kubectl get ds -n kube-system
```

## Enable Diagnostic Settings

Enable diagnostic settings to send AKS control plane logs to Log Analytics:

```powershell
$AKS_ID = az aks show --name $AKS_NAME --resource-group $RESOURCE_GROUP --query id -o tsv

az monitor diagnostic-settings create `
    --name "aks-diagnostics" `
    --resource $AKS_ID `
    --workspace $LOG_ANALYTICS_WORKSPACE_ID `
    --logs '[
        {"categoryGroup": "allLogs", "enabled": true}
    ]' `
    --metrics '[
        {"category": "AllMetrics", "enabled": true}
    ]'
```

### Verify Diagnostic Settings

```powershell
az monitor diagnostic-settings list --resource $AKS_ID -o table
```
