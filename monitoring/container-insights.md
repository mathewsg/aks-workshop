# AKS Monitoring - Container Insights

## Prerequisites

### Set Environment Variables

```powershell
$AKS_NAME = "aks-mon"
$RESOURCE_GROUP = "azure-rg"
$LOCATION = "swedencentral"
$VM_SKU = "Standard_D2as_v5"
$LOG_ANALYTICS_WORKSPACE_NAME = "law-aks-monitoring"
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
    --workspace-name $LOG_ANALYTICS_WORKSPACE_NAME `
    --location $LOCATION

$LOG_ANALYTICS_WORKSPACE_NAME_ID = az monitor log-analytics workspace show `
    --resource-group $RESOURCE_GROUP `
    --workspace-name $LOG_ANALYTICS_WORKSPACE_NAME `
    --query id -o tsv
```

## Enable Container Insights

```powershell
az aks enable-addons `
    --addon monitoring `
    --name $AKS_NAME `
    --resource-group $RESOURCE_GROUP `
    --workspace-resource-id $LOG_ANALYTICS_WORKSPACE_NAME_ID
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

## Deploy a Log-Generating Sample App

Deploy a simple pod that continuously writes timestamped messages to stdout:

```powershell
kubectl run log-generator --image=busybox --restart=Never -- /bin/sh -c 'i=0; while true; do echo "$i [INFO] Heartbeat from log-generator pod - count=$i"; i=$((i+1)); sleep 5; done'
```

### Verify the Pod is Running and Producing Logs

```powershell
kubectl get pod log-generator
kubectl logs log-generator
```

Expected output:

```
2026-07-23 10:00:00 [INFO] Heartbeat from log-generator pod - count=0
2026-07-23 10:00:05 [INFO] Heartbeat from log-generator pod - count=1
2026-07-23 10:00:10 [INFO] Heartbeat from log-generator pod - count=2
```

## Query ContainerLogV2 for Stdout

Wait ~5 minutes for logs to flow into Log Analytics, then navigate to **Log Analytics Workspace > Logs** in the Azure Portal and run:

```kql
ContainerLogV2
| where PodName == "log-generator"
| where LogSource == "stdout"
| project TimeGenerated, PodName, ContainerName, LogMessage
| order by TimeGenerated desc
| take 50
```

### Cleanup

```powershell
kubectl delete pod log-generator
```
