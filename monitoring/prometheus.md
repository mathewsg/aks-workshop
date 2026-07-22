# AKS Monitoring - Prometheus & Managed Grafana

## Prerequisites

### Set Environment Variables

```powershell
$CLUSTER_NAME = "aks-mon"
$RESOURCE_GROUP = "azure-rg"
$LOCATION = "swedencentral"
$LOG_ANALYTICS_WORKSPACE = "law-aks-monitoring"
$AZURE_MONITOR_WORKSPACE = "amw-aks-monitoring"
$GRAFANA_NAME = "grafana-aks-monitoring"
```

### Pre-requisites

- Resource group and AKS cluster deployed (see [container-insights.md](container-insights.md) for base setup)
- Log Analytics Workspace deployed



## Deploy Azure Monitor Workspace (for Managed Prometheus)

```powershell
az monitor account create `
    --name $AZURE_MONITOR_WORKSPACE `
    --resource-group $RESOURCE_GROUP `
    --location $LOCATION

$AZURE_MONITOR_WORKSPACE_ID = az monitor account show `
    --name $AZURE_MONITOR_WORKSPACE `
    --resource-group $RESOURCE_GROUP `
    --query id -o tsv
```

## Enable Prometheus Add-On (Metrics)

```powershell
az aks update `
    --name $CLUSTER_NAME `
    --resource-group $RESOURCE_GROUP `
    --enable-azure-monitor-metrics `
    --azure-monitor-workspace-resource-id $AZURE_MONITOR_WORKSPACE_ID
```

### Verify Prometheus Pods

```bash
kubectl get pods -n kube-system | Select-String "ama-metrics"
```

Expected output:

```
ama-metrics-xxxxx              1/1     Running   0   2m
ama-metrics-node-xxxxx         1/1     Running   0   2m
ama-metrics-ksm-xxxxx          1/1     Running   0   2m
```

## Deploy Azure Managed Grafana

```powershell
az grafana create `
    --name $GRAFANA_NAME `
    --resource-group $RESOURCE_GROUP `
    --location $LOCATION
```

### Assign Grafana Admin Role to Current User

```powershell
$GRAFANA_ID = az grafana show --name $GRAFANA_NAME --resource-group $RESOURCE_GROUP --query id -o tsv
$CURRENT_USER_ID = az ad signed-in-user show --query id -o tsv

az role assignment create `
    --assignee $CURRENT_USER_ID `
    --role "Grafana Admin" `
    --scope $GRAFANA_ID
```

### Link Grafana to Azure Monitor Workspace

```powershell
az aks update `
    --name $CLUSTER_NAME `
    --resource-group $RESOURCE_GROUP `
    --grafana-resource-id $GRAFANA_ID
```

### Verify Grafana

```powershell
az grafana show --name $GRAFANA_NAME --resource-group $RESOURCE_GROUP --query "{name:name, endpoint:properties.endpoint, provisioningState:properties.provisioningState}" -o table
```

Once deployed, open the Grafana endpoint URL in a browser. Pre-built Kubernetes dashboards are automatically provisioned when linked to the AKS Prometheus data source.

## Summary

| Component | What It Does |
|-----------|--------------|
| Azure Monitor Workspace | Stores Prometheus metrics for querying via PromQL |
| Prometheus Add-On | Scrapes Kubernetes metrics and sends to Azure Monitor Workspace |
| Azure Managed Grafana | Visualizes Prometheus metrics via pre-built Kubernetes dashboards |
