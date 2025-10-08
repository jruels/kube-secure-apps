# Module 2 - Traffic Management

This module demonstrates Istio's powerful traffic routing capabilities on Amazon EKS. You'll learn how to control traffic flow between service versions using VirtualServices and DestinationRules.

---

## What You'll Learn

- How to route traffic to specific service versions
- Weight-based traffic splitting (canary deployments)
- Path-based routing
- Header-based routing
- Traffic mirroring (shadow traffic)

---

## Prerequisites

1. **Complete [Module 1 - Getting Started](../01-getting-started/README.md)**
   - EKS cluster with Istio installed
   - Workshop namespace with sample applications deployed
   - Applications accessible via the ingress gateway

2. **Verify Module 1 is still running:**
   ```bash
   kubectl get pods -n workshop
   # All pods should show 2/2 READY
   
   kubectl get gateway,virtualservice -n workshop
   # Should see productapp-gateway and productapp virtualservice
   ```

---

## Lab Scenarios

This module contains 5 traffic management scenarios:

| Scenario | Description | Use Case |
|----------|-------------|----------|
| [1. Version Routing](#scenario-1-route-traffic-to-specific-version) | Route 100% traffic to v1 | Blue/green deployments |
| [2. Weight-based Routing](#scenario-2-weight-based-routing-canary) | Split traffic 90% v1, 10% v2 | Canary deployments |
| [3. Path-based Routing](#scenario-3-path-based-routing) | Route by URI path | API versioning |
| [4. Header-based Routing](#scenario-4-header-based-routing) | Route by HTTP header | A/B testing, internal previews |
| [5. Traffic Mirroring](#scenario-5-traffic-mirroring) | Mirror traffic to v2 | Testing in production |

---

## Initial Setup - Configure Mesh Resources

Before testing traffic management, apply the mesh resources that define subsets for version routing:

```bash
# Ensure you're in the module 2 directory
cd istio/modules/traffic-management

# Apply mesh resources
kubectl apply -f manifests/setup-mesh-resources/
```

**Expected output:**
```
destinationrule.networking.istio.io/catalogdetail created
virtualservice.networking.istio.io/catalogdetail created
virtualservice.networking.istio.io/frontend created
virtualservice.networking.istio.io/productcatalog created
```

**What this does:**
- Creates a **DestinationRule** for `catalogdetail` with two subsets: `v1` and `v2` (based on version labels)
- Creates **VirtualServices** for catalogdetail, frontend, and productcatalog
- Initially routes traffic evenly (~50/50) to both v1 and v2 of catalogdetail

**Verify the resources:**
```bash
kubectl get Gateway,VirtualService,DestinationRule -n workshop
```

**Expected output:**
```
NAME                                             AGE
gateway.networking.istio.io/productapp-gateway   [age]

NAME                                                GATEWAYS                 HOSTS                AGE
virtualservice.networking.istio.io/catalogdetail                             ["catalogdetail"]    [age]
virtualservice.networking.istio.io/frontend                                  ["frontend"]         [age]
virtualservice.networking.istio.io/productapp       ["productapp-gateway"]   ["*"]                [age]
virtualservice.networking.istio.io/productcatalog                            ["productcatalog"]   [age]

NAME                                                HOST                                       AGE
destinationrule.networking.istio.io/catalogdetail   catalogdetail.workshop.svc.cluster.local   [age]
```

**Observe initial traffic distribution (optional):**

If you have Kiali running from Module 1, generate some traffic and observe the 50/50 split:

```bash
# Linux/Mac
ISTIO_INGRESS_URL=$(kubectl get svc istio-ingress -n istio-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')
siege http://$ISTIO_INGRESS_URL -c 5 -d 10 -t 1M

# Windows
ISTIO_INGRESS_URL=$(kubectl get svc istio-ingress -n istio-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')
hey -z 1m -c 5 -q 1 "http://$ISTIO_INGRESS_URL"
```

In Kiali (http://localhost:20001), you should see traffic split approximately evenly between catalogdetail v1 and v2.

---

## Scenario 1: Route Traffic to Specific Version

In this scenario, we route 100% of traffic to the v1 version of the catalogdetail service.

**Use case:** Blue/green deployment - route all production traffic to a specific version.

### Deploy

```bash
kubectl apply -f manifests/route-traffic-to-version-v1/catalogdetail-virtualservice.yaml
```

**Expected output:**
```
virtualservice.networking.istio.io/catalogdetail configured
```

### Validate

Check that the VirtualService is configured to route only to v1:

```bash
kubectl describe VirtualService catalogdetail -n workshop
```

**Look for this in the output:**
```
Spec:
  Hosts:
    catalogdetail
  Http:
    Route:
      Destination:
        Host:  catalogdetail
        Port:
          Number:  3000
        Subset:    v1      # <-- All traffic goes to v1
```

### Test

**Generate traffic and observe in Kiali:**

```bash
# Linux/Mac
ISTIO_INGRESS_URL=$(kubectl get svc istio-ingress -n istio-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')
siege http://$ISTIO_INGRESS_URL -c 5 -d 10 -t 2M

# Windows
ISTIO_INGRESS_URL=$(kubectl get svc istio-ingress -n istio-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')
hey -z 2m -c 5 -q 1 "http://$ISTIO_INGRESS_URL"
```

**In Kiali:** You should now see 100% of traffic going to catalogdetail v1 only.

---

## Scenario 2: Weight-based Routing (Canary)

Gradually shift traffic to a new version by sending 90% to v1 and 10% to v2.

**Use case:** Canary deployment - test a new version with a small percentage of production traffic.

### Deploy

```bash
kubectl apply -f manifests/weight-based-routing/catalogdetail-virtualservice.yaml
```

**Expected output:**
```
virtualservice.networking.istio.io/catalogdetail configured
```

### Validate

```bash
kubectl describe VirtualService catalogdetail -n workshop
```

**Look for weights in the output:**
```
Http:
  Route:
    Destination:
      Host:  catalogdetail
      Subset:    v1
    Weight:      90      # <-- 90% to v1
    Destination:
      Host:  catalogdetail
      Subset:    v2
    Weight:      10      # <-- 10% to v2
```

### Test

Generate traffic and observe the weighted distribution:

```bash
# Windows
hey -z 2m -c 5 -q 1 "http://$ISTIO_INGRESS_URL"
```

**In Kiali:** Traffic distribution should show approximately 90% to v1 and 10% to v2.

**Tip:** You can gradually increase the weight to v2 (e.g., 70/30, 50/50, 30/70, 0/100) for progressive rollouts.

---

## Scenario 3: Path-based Routing

Route traffic based on the request URI path. Requests to `/v2/catalogDetail` go to v2, while `/v1/catalogDetail` goes to v1.

**Use case:** API versioning - maintain multiple API versions simultaneously.

### Deploy

```bash
# Update the VirtualService with path-based routing
kubectl apply -f manifests/path-based-routing/catalogdetail-virtualservice.yaml

# Update the productcatalog environment variable to call the v2 endpoint
kubectl set env deployment/productcatalog -n workshop \
  AGG_APP_URL=http://catalogdetail.workshop.svc.cluster.local:3000/v2/catalogDetail
```

**Expected output:**
```
virtualservice.networking.istio.io/catalogdetail configured
deployment.apps/productcatalog env updated
```

**Wait for productcatalog to roll out the change:**
```bash
kubectl rollout status deployment/productcatalog -n workshop
```

### Validate

Check the VirtualService configuration:

```bash
kubectl describe VirtualService catalogdetail -n workshop
```

**Look for URI matching rules:**
```
Http:
  Match:
    Uri:
      Exact:  /v2/catalogDetail     # <-- Match v2 path
  Rewrite:
    Uri:  /catalogDetail            # <-- Rewrite to actual endpoint
  Route:
    Destination:
      Host:  catalogdetail
      Subset:    v2                 # <-- Route to v2
  Match:
    Uri:
      Exact:  /v1/catalogDetail     # <-- Match v1 path
  Rewrite:
    Uri:  /catalogDetail
  Route:
    Destination:
      Host:  catalogdetail
      Subset:    v1                 # <-- Route to v1
```

### Test

Generate traffic - all requests should now go to v2:

```bash
# Windows
hey -z 2m -c 5 -q 1 "http://$ISTIO_INGRESS_URL"
```

**In Kiali:** 100% of traffic should flow to catalogdetail v2.

### Test v1 Path

Now switch to the v1 path:

```bash
kubectl set env deployment/productcatalog -n workshop \
  AGG_APP_URL=http://catalogdetail.workshop.svc.cluster.local:3000/v1/catalogDetail

# Wait for rollout
kubectl rollout status deployment/productcatalog -n workshop
```

Generate traffic again - should now go to v1.

### Revert to Original

```bash
kubectl set env deployment/productcatalog -n workshop \
  AGG_APP_URL=http://catalogdetail.workshop.svc.cluster.local:3000/catalogDetail
```

---

## Scenario 4: Header-based Routing

Route traffic based on HTTP headers. Requests with `user-type: internal` go to v2, all others go to v1.

**Use case:** A/B testing, internal previews - route specific users to new features.

### Deploy

This scenario uses an EnvoyFilter to inject the `user-type` header randomly:

```bash
kubectl apply -f manifests/header-based-routing/
```

**Expected output:**
```
virtualservice.networking.istio.io/catalogdetail configured
envoyfilter.networking.istio.io/productcatalog created
Warning: EnvoyFilter exposes internal implementation details...
```

**Note:** The warning is expected - EnvoyFilter is an advanced feature that modifies Envoy proxy configuration.

**What this does:**
- The **VirtualService** routes traffic with `user-type: internal` header to v2, all other traffic to v1
- The **EnvoyFilter** randomly sets the `user-type` header to `internal` 30% of the time, `external` 70% of the time

### Validate

Check the VirtualService:

```bash
kubectl describe VirtualService catalogdetail -n workshop
```

**Look for header matching:**
```
Http:
  Match:
    Headers:
      User - Type:
        Exact:  internal        # <-- Match internal users
  Route:
    Destination:
      Host:  catalogdetail
      Subset:    v2             # <-- Route to v2
  Route:
    Destination:
      Host:  catalogdetail
      Subset:    v1             # <-- Default route to v1
```

Check the EnvoyFilter:

```bash
kubectl describe EnvoyFilter productcatalog -n workshop
```

### Test

Generate traffic:

```bash
# Windows
hey -z 2m -c 5 -q 1 "http://$ISTIO_INGRESS_URL"
```

**In Kiali:** You should see approximately 70% traffic to v1 and 30% to v2 (based on the random header injection).

### Clean Up

Remove the EnvoyFilter when done:

```bash
kubectl delete -f ./header-based-routing/productcatalog-envoyfilter.yaml
```

---

## Scenario 5: Traffic Mirroring

Send all live traffic to v1, but also mirror (shadow) 50% of the traffic to v2 for testing. The responses from v2 are discarded.

**Use case:** Test a new version in production with real traffic without affecting users.

### Deploy

```bash
kubectl apply -f manifests/traffic-mirroring/catalogdetail-virtualservice.yaml
```

**Expected output:**
```
virtualservice.networking.istio.io/catalogdetail configured
```

### Validate

```bash
kubectl describe VirtualService catalogdetail -n workshop
```

**Look for mirror configuration:**
```
Http:
  Mirror:
    Host:  catalogdetail
    Subset:    v2                # <-- Mirror destination
  Mirror Percentage:
    Value:  50                   # <-- Mirror 50% of traffic
  Route:
    Destination:
      Host:  catalogdetail
      Subset:    v1              # <-- Primary destination
    Weight:      100
```

### Test

Generate traffic:

```bash
# Windows
hey -z 2m -c 5 -q 1 "http://$ISTIO_INGRESS_URL"
```

**In Kiali:**
- 100% of production traffic goes to v1
- 50% of that traffic is also mirrored to v2
- Responses from v2 are discarded (users only see v1 responses)

**You can check v2 logs to see it's receiving traffic:**
```bash
kubectl logs -n workshop -l app=catalogdetail,version=v2 --tail=20
```

---

## Understanding the Components

### DestinationRule

Defines subsets (versions) of a service:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: catalogdetail
spec:
  host: catalogdetail.workshop.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1      # Pods with this label belong to subset v1
  - name: v2
    labels:
      version: v2      # Pods with this label belong to subset v2
```

### VirtualService

Routes traffic to specific subsets:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: catalogdetail
spec:
  hosts:
  - catalogdetail
  http:
  - route:
    - destination:
        host: catalogdetail
        subset: v1      # Route to v1 subset
      weight: 90
    - destination:
        host: catalogdetail
        subset: v2      # Route to v2 subset
      weight: 10
```

---

## Clean Up

To reset for the next module or scenario:

```bash
# Revert to initial mesh resources (50/50 split)
kubectl apply -f manifests/setup-mesh-resources/

# Or clean up entirely
helm uninstall mesh-basic -n workshop
kubectl delete namespace workshop
```

---

## Troubleshooting

### Traffic not routing as expected

```bash
# Check VirtualService configuration
kubectl get virtualservice catalogdetail -n workshop -o yaml

# Check DestinationRule subsets
kubectl get destinationrule catalogdetail -n workshop -o yaml

# Verify pod labels match subset labels
kubectl get pods -n workshop --show-labels | grep catalogdetail
```

### EnvoyFilter not working

```bash
# Check EnvoyFilter exists
kubectl get envoyfilter -n workshop

# Check productcatalog pods restarted after EnvoyFilter was applied
kubectl get pods -n workshop -l app=productcatalog
```

### Kiali not showing expected traffic

- Wait 30-60 seconds for metrics to update
- Refresh the Kiali page
- Ensure traffic is being generated
- Check that Prometheus is running: `kubectl get pods -n istio-system | grep prometheus`

---

## Key Takeaways

* **VirtualServices** control how requests are routed to services

*  **DestinationRules** define service subsets based on labels
* **Weight-based routing** enables canary and blue/green deployments
* **Path and header routing** enable API versioning and A/B testing
* **Traffic mirroring** allows safe testing with production traffic

---

## Next Steps

Continue to **[Module 3 - Network Resiliency](../03-network-resiliency/)** to learn about:
- Fault injection
- Timeouts and retries
- Circuit breaking
- Rate limiting
