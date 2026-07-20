# Traffic Splitting with Application Gateway for Containers - Gateway API

> Based on [Traffic Splitting with Application Gateway for Containers - Gateway API](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/how-to-traffic-splitting-gateway-api?tabs=alb-managed)

## Prerequisites

- ALB Controller deployed and Application Gateway for Containers provisioned via `ApplicationLoadBalancer` custom resource. See [create-agc-managed.md](create-agc-managed.md).

## Deploy Sample Backend Application

Deploys two backend services (`backend-v1` and `backend-v2`) in the `test-infra` namespace:

```bash
kubectl apply -f traffic-split-deployment.yaml
```

Verify pods are running:

```bash
kubectl get pods -n test-infra
```

## Deploy the Gateway

```bash
kubectl apply -f traffic-split-gateway.yaml
```

Verify the gateway status:

```bash
kubectl get gateway gateway-01 -n test-infra -o yaml
```

Wait until the gateway shows `status: "True"` for both `Accepted` and `Programmed` conditions, and an address is assigned.

## Deploy the HTTPRoute with Traffic Splitting

The route splits traffic 50/50 between `backend-v1` and `backend-v2`:

```bash
kubectl apply -f traffic-split-httproute.yaml
```

Verify the HTTPRoute status:

```bash
kubectl get httproute traffic-split-route -n test-infra -o yaml
```

Ensure the route shows `Accepted` and `Programmed` conditions with `status: "True"`.

## Test Access to the Application

Get the FQDN assigned to the gateway:

```powershell
$fqdn = kubectl get gateway gateway-01 -n test-infra -o jsonpath='{.status.addresses[0].value}'
```

Send traffic and observe responses from both backends:

```powershell
curl http://$fqdn
```

Approximately 50% of responses will come from `backend-v1` and 50% from `backend-v2`.

## Adjusting Traffic Weights

To change the traffic split (e.g., 80/20), edit `traffic-split-httproute.yaml` and update the `weight` values:

```yaml
  - backendRefs:
    - name: backend-v1
      port: 8080
      weight: 80
    - name: backend-v2
      port: 8080
      weight: 20
```

Then reapply:

```bash
kubectl apply -f traffic-split-httproute.yaml
```
