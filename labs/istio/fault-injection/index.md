# Network Resiliency - Fault Injection

This sub-module demonstrates Istio's fault injection capabilities for testing application resilience. You'll learn how to inject **delays** and **aborts** (HTTP errors) into requests to see how your microservices handle failures.

---

## What You'll Learn

- How to inject artificial delays into HTTP requests
- How to inject HTTP errors (aborts) to simulate service failures
- How to use header-based routing to inject faults for specific users
- How to test application behavior under failure conditions

---

## Why Fault Injection?

Fault injection is a **chaos engineering** technique that helps you:

- **Test resilience**: Verify that your application gracefully handles slow or failing services
- **Find weaknesses**: Discover bugs and bottlenecks before they occur in production
- **Safe testing**: Inject faults without actually breaking your services
- **Targeted testing**: Apply faults only to specific users or a percentage of traffic

---

## Prerequisites

1. **Complete [Module 3 Initial Setup](../README.md#initial-setup---configure-mesh-resources)**
   - Mesh resources applied
   - All pods running with 2/2 containers

2. **Verify workshop namespace:**
   ```bash
   kubectl get pods -n workshop
   # All pods should show 2/2 READY
   ```

---

## Table of Contents

1. [Delay Injection](#scenario-1-injecting-delay-fault-into-http-requests)
2. [Abort Injection](#scenario-2-injecting-abort-fault-into-http-requests)

---

## Scenario 1: Injecting Delay Fault into HTTP Requests

In this scenario, we inject a **15-second delay** for requests with a specific header (`user: internal`). All other requests proceed normally.

**Use case:** Test how your application behaves when a downstream service is slow (e.g., database query taking too long).

### Deploy

```bash
# Enter the lab directory
cd labs/istio/modules/fault-injection
# Apply delay configuration
kubectl apply -f manifests/delay/catalogdetail-virtualservice.yaml
```

**Expected output:**
```
virtualservice.networking.istio.io/catalogdetail configured
```

### Validate

Verify the delay configuration in the VirtualService:

```bash
kubectl get virtualservice catalogdetail -n workshop -o yaml
```

**Look for this in the output:**
```yaml
spec:
  hosts:
  - catalogdetail
  http:
  - fault:
      delay:
        fixedDelay: 15s        # 15-second delay
        percentage:
          value: 100            # 100% of matching requests
    match:
    - headers:
        user:
          exact: internal       # Only for requests with user: internal header
    route:
    - destination:
        host: catalogdetail
        port:
          number: 3000
  - route:                      # Default route (no delay)
    - destination:
        host: catalogdetail
        port:
          number: 3000
```

**What this configuration does:**
- **First rule**: Matches requests with `user: internal` header and injects a 15-second delay
- **Second rule**: All other requests proceed normally (no delay)

### Test

Test the delay by making requests from within the mesh:

**Step 1: Get the frontend pod name**
```bash
export FE_POD_NAME=$(kubectl get pods -n workshop -l app=frontend -o jsonpath='{.items[].metadata.name}')
echo "Frontend pod: $FE_POD_NAME"
```

**Step 2: Exec into the frontend container**
```bash
kubectl exec -it ${FE_POD_NAME} -n workshop -c frontend -- bash
```

You should now see a shell prompt inside the frontend container:
```
root@frontend-xxxxxxxxx-xxxxx:/app#
```

**Step 3: Test internal user (should have 15-second delay)**
```bash
curl http://catalogdetail:3000/catalogdetail/ -s -H "user: internal" -o /dev/null \
-w "Time taken to start transfer: %{time_starttransfer}\n"
```

**Expected output:**
```
Time taken to start transfer: 15.009529
```
* **15-second delay is injected for internal user**

**Step 4: Test external user (should be fast)**

```bash
curl http://catalogdetail:3000/catalogdetail/ -s -H "user: external" -o /dev/null \
-w "Time taken to start transfer: %{time_starttransfer}\n"
```

**Expected output:**
```
Time taken to start transfer: 0.006548
```
* **No delay for external user**

**Step 5: Exit the container**

```bash
exit
```

### What Just Happened?

1. Istio's Envoy proxy intercepted the request from frontend to catalogdetail
2. It matched the `user: internal` header in the request
3. It injected a 15-second delay before forwarding the request to catalogdetail
4. Requests without the header or with different headers were not delayed

---

## Scenario 2: Injecting Abort Fault into HTTP Requests

In this scenario, we inject an **HTTP 500 error** for requests with the `user: internal` header. This simulates a service failure.

**Use case:** Test how your application handles downstream service failures (e.g., database connection error).

### Deploy

```bash
# Apply abort configuration
kubectl apply -f manifests/abort/catalogdetail-virtualservice.yaml
```

**Expected output:**
```
virtualservice.networking.istio.io/catalogdetail configured
```

### Validate

Verify the abort configuration:

```bash
kubectl get virtualservice catalogdetail -n workshop -o yaml
```

**Look for this in the output:**
```yaml
spec:
  hosts:
  - catalogdetail
  http:
  - fault:
      abort:
        httpStatus: 500         # Return HTTP 500 error
        percentage:
          value: 100            # 100% of matching requests
    match:
    - headers:
        user:
          exact: internal       # Only for requests with user: internal header
    route:
    - destination:
        host: catalogdetail
        port:
          number: 3000
  - route:                      # Default route (no abort)
    - destination:
        host: catalogdetail
        port:
          number: 3000
```

**What this configuration does:**
- **First rule**: Matches requests with `user: internal` header and returns HTTP 500 error
- **Second rule**: All other requests proceed normally (no error)

### Test

Test the abort fault:

**Step 1: Exec into the frontend container**
```bash
export FE_POD_NAME=$(kubectl get pods -n workshop -l app=frontend -o jsonpath='{.items[].metadata.name}')
kubectl exec -it ${FE_POD_NAME} -n workshop -c frontend -- bash
```

**Step 2: Test internal user (should get HTTP 500)**
```bash
curl http://catalogdetail:3000/catalogdetail/ -s -H "user: internal" -o /dev/null \
-w "HTTP Response: %{http_code}\n"
```

**Expected output:**
```
HTTP Response: 500
```
* **HTTP 500 error is returned for internal user**

**Step 3: Test external user (should get HTTP 200)**

```bash
curl http://catalogdetail:3000/catalogdetail/ -s -H "user: external" -o /dev/null \
-w "HTTP Response: %{http_code}\n"
```

**Expected output:**
```
HTTP Response: 200
```
* **Normal response (200 OK) for external user**

**Step 4: Exit the container**
```bash
exit
```

### What Just Happened?

1. Istio's Envoy proxy intercepted the request from frontend to catalogdetail
2. It matched the `user: internal` header in the request
3. It immediately returned an HTTP 500 error **without** forwarding the request to catalogdetail
4. Requests without the header or with different headers were forwarded normally

---

## Understanding Fault Injection

### How Does Istio Inject Faults?

Istio uses the **Envoy sidecar proxy** to intercept traffic between services. When a VirtualService defines fault injection rules:

1. The Envoy proxy inspects each request
2. If the request matches the fault injection criteria (e.g., specific header)
3. The proxy applies the fault (delay or abort) before forwarding the request

```
┌─────────────┐        ┌──────────────────┐        ┌──────────────┐
│  Frontend   │───────▶│  Envoy Sidecar   │───────▶│ CatalogDetail│
│  Container  │        │  (Fault Inject)  │        │   Container  │
└─────────────┘        └──────────────────┘        └──────────────┘
                              │
                              │ Delay: Wait 15s
                              │ Abort: Return HTTP 500
                              ▼
                       Fault Injection Rules
                       from VirtualService
```

### Delay vs Abort

| Feature | Delay | Abort |
|---------|-------|-------|
| **What it does** | Adds artificial latency | Returns HTTP error code |
| **Use case** | Test slow services | Test service failures |
| **Example** | Database query taking 15 seconds | Database connection refused |
| **HTTP status** | Eventually returns real response | Immediately returns error (e.g., 500) |

---

## Advanced Fault Injection

### Inject Faults for a Percentage of Traffic

Instead of 100% of matching requests, you can inject faults for a percentage:

```yaml
fault:
  delay:
    fixedDelay: 5s
    percentage:
      value: 50          # Only 50% of requests get delayed
```

### Combine Delay and Abort

You can apply both delay **and** abort to the same request:

```yaml
fault:
  delay:
    fixedDelay: 3s
  abort:
    httpStatus: 503
    percentage:
      value: 20          # 20% get both delay + abort
```

---

## Reset the Environment

After testing fault injection, reset to the initial state:

```bash
# Reset to initial mesh resources
kubectl apply -f manifests/setup-mesh-resources/
```

**Expected output:**
```
virtualservice.networking.istio.io/catalogdetail configured
destinationrule.networking.istio.io/catalogdetail unchanged
virtualservice.networking.istio.io/frontend unchanged
virtualservice.networking.istio.io/productcatalog unchanged
```

This removes the fault injection rules and returns to baseline configuration.

---

## Troubleshooting

### Delay not working

```bash
# Check VirtualService configuration
kubectl get virtualservice catalogdetail -n workshop -o yaml

# Verify the fault configuration is present
# Look for: fault.delay.fixedDelay

# Check that you're using the correct header
# Verify: match.headers.user.exact: internal
```

### Abort not working

```bash
# Check VirtualService configuration
kubectl get virtualservice catalogdetail -n workshop -o yaml

# Verify the fault configuration is present
# Look for: fault.abort.httpStatus

# Ensure you're sending the request with the correct header
```

### Can't exec into frontend pod

```bash
# Check if frontend pod exists
kubectl get pods -n workshop -l app=frontend

# If pod is not running, redeploy Module 1
cd ../../01-getting-started
helm install mesh-basic . -n workshop
```

---

## Key Takeaways

* **Delay injection** simulates slow downstream services
* **Abort injection** simulates service failures with HTTP errors
* **Header-based matching** allows targeted fault injection for specific users
* **Percentage-based faults** enable gradual rollout of chaos tests
* **Safe testing** - faults are injected by the proxy, not your actual services

---

## Next Steps

Continue to **[Timeouts, Retries, and Circuit Breaking](../timeouts-retries-circuitbreaking/README.md)** to learn how to:
- Set timeouts to prevent slow requests from blocking your application
- Automatically retry failed requests
- Protect services from overload with circuit breakers

---

## Additional Resources

- [Istio Fault Injection Documentation](https://istio.io/latest/docs/tasks/traffic-management/fault-injection/)
- [Envoy Fault Injection](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/fault_filter)
- [Chaos Engineering Principles](https://principlesofchaos.org/)
