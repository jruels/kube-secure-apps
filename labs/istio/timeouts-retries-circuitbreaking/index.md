# Network Resiliency - Timeouts, Retries, and Circuit Breaking

This sub-module demonstrates advanced network resiliency features in Istio: **Timeouts**, **Retries**, and **Circuit Breaking**. These features protect your microservices from cascading failures and overload conditions.

---

## What You'll Learn

- How to configure timeouts to prevent slow services from blocking your application
- How to implement automatic retries for failed requests
- How to use circuit breakers to protect services from overload
- How to test and validate each resiliency feature

---

## Why These Features Matter

| Feature | Problem it Solves | How it Helps |
|---------|------------------|--------------|
| **Timeouts** | Slow services blocking your application | Fail fast instead of waiting indefinitely |
| **Retries** | Transient network failures | Automatically retry failed requests |
| **Circuit Breaking** | Overloaded services crashing | Stop sending traffic to struggling services |

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

3. **For advanced scenarios, install istioctl:**
   - See [Module 3 Prerequisites](../README.md#prerequisites) for installation instructions

---

## Table of Contents

1. [Timeouts](#scenario-1-timeouts)
2. [Retries](#scenario-2-retries)
3. [Circuit Breaking](#scenario-3-circuit-breaking)

---

## Scenario 1: Timeouts

Timeouts prevent slow services from blocking your entire application. In this scenario:
- `catalogdetail` has a **5-second delay**
- `productcatalog` calls `catalogdetail` with a **2-second timeout**
- Result: Request fails fast after 2 seconds instead of waiting 5 seconds

**Use case:** Database queries that take too long shouldn't block your API responses.

### Step 1: Apply Delay to catalogdetail

First, we inject a 5-second delay into the catalogdetail service:

```bash
#Apply delay configuration
kubectl apply -f manifests/timeouts/catalogdetail-virtualservice.yaml
```

**Expected output:**
```
virtualservice.networking.istio.io/catalogdetail configured
```

### Step 2: Test the Delay (Before Timeout)

Let's verify the 5-second delay is working:

```bash
# Get frontend pod name
export FE_POD_NAME=$(kubectl get pods -n workshop -l app=frontend -o jsonpath='{.items[].metadata.name}')

# Exec into the frontend container
kubectl exec -it ${FE_POD_NAME} -n workshop -c frontend -- bash

# Test the delay
curl http://productcatalog:5000/products/ -s -o /dev/null -w "Time taken to start transfer: %{time_starttransfer}\n"
```

**Expected output:**
```
Time taken to start transfer: 5.024133
```
✅ **5-second delay is working** (productcatalog waits for catalogdetail)

Exit the container:
```bash
exit
```

### Step 3: Apply Timeout to productcatalog

Now add a 2-second timeout to productcatalog:

```bash
# Apply timeout configuration
kubectl apply -f manifests/timeouts/productcatalog-virtualservice.yaml
```

**Expected output:**
```
virtualservice.networking.istio.io/productcatalog configured
```

### Step 4: Validate Timeout Configuration

Check the VirtualService configuration:

```bash
kubectl get virtualservice productcatalog -n workshop -o yaml
```

**Look for this in the output:**
```yaml
spec:
  hosts:
  - productcatalog
  http:
  - route:
    - destination:
        host: productcatalog
    timeout: 2s          # Fail after 2 seconds
```

### Step 5: Test the Timeout

```bash
# Exec into the frontend container
kubectl exec -it ${FE_POD_NAME} -n workshop -c frontend -- bash

# Test with timeout (should fail after 2 seconds)
curl http://productcatalog:5000/products/ -s -o /dev/null -w "Time taken to start transfer: %{time_starttransfer}\n"
```

**Expected output:**
```
Time taken to start transfer: 2.006172
```
✅ **Timeout is working** (fails after 2 seconds instead of waiting 5 seconds)

Exit the container:
```bash
exit
```

### What Just Happened?

1. `productcatalog` makes a request to `catalogdetail`
2. `catalogdetail` has a 5-second delay configured
3. `productcatalog` has a 2-second timeout configured
4. After 2 seconds, Istio's Envoy proxy **cancels the request** and returns an error
5. This prevents the slow catalogdetail service from blocking productcatalog

```
┌──────────────┐    2s timeout    ┌──────────────┐    5s delay    ┌──────────────┐
│ProductCatalog│─────────────────▶│Envoy Sidecar │───────────────▶│CatalogDetail │
└──────────────┘                  └──────────────┘                └──────────────┘
       │                                 │
       │◀────── Error after 2s ──────────│
       │                                 │
       │                          (Request canceled,
       │                           catalogdetail never
       │                           receives the request)
```

### Reset the Environment

```bash
kubectl apply -f manifests/setup-mesh-resources/
```

---

## Scenario 2: Retries

Retries automatically retry failed requests, which is useful for handling transient network failures.

In this scenario:
- `productcatalog` deployment is broken (sleeps instead of serving requests)
- `productcatalog` VirtualService is configured to retry 2 times
- You'll see Envoy making 3 total attempts (1 initial + 2 retries)

**Use case:** Network blips or temporary service unavailability often succeed on retry.

### Step 1: Configure Retries

Apply retry configuration to the productcatalog VirtualService:

```bash
#Apply retry configuration
kubectl apply -f manifests/retries/productcatalog-virtualservice.yaml
```

**Expected output:**
```
virtualservice.networking.istio.io/productcatalog configured
```

### Step 2: Validate Retry Configuration

```bash
kubectl get virtualservice productcatalog -n workshop -o yaml
```

**Look for this in the output:**
```yaml
spec:
  hosts:
  - productcatalog
  http:
  - route:
    - destination:
        host: productcatalog
    retries:
      attempts: 2             # Retry up to 2 times
      perTryTimeout: 2s       # Each attempt has a 2s timeout
```

### Step 3: Break the productcatalog Deployment

We'll modify the productcatalog deployment to sleep instead of serving requests:

```bash
# This command modifies the deployment to run 'sleep 1h' instead of the app
kubectl get deployment -n workshop productcatalog -o json | \
jq '.spec.template.spec.containers[0].readinessProbe={exec:{command:["sh","-c","echo hello"]}}
| .spec.template.spec.containers[0].livenessProbe={exec:{command:["sh","-c","echo hello"]}}
| .spec.template.spec.containers[0]+={command:["sh","-c","sleep 1h"]}' | \
kubectl apply --force=true -f -
```

**What this does:**
- Changes the container command to `sleep 1h` (doesn't serve HTTP requests)
- Changes health probes to always succeed (pod appears healthy but doesn't work)

**Wait for the pod to restart:**
```bash
kubectl rollout status deployment/productcatalog -n workshop
```

### Step 4: Enable Debug Logging (Optional)

In a **second terminal**, enable debug logging for productcatalog's Envoy proxy:

```bash
# This requires istioctl to be installed
istioctl pc log --level debug -n workshop deploy/productcatalog
```

### Step 5: Watch Retry Attempts

In your **first terminal**, watch the Envoy logs for retry attempts:

```bash
kubectl -n workshop logs -l app=productcatalog -c istio-proxy -f | grep "x-envoy-attempt-count"
```

**Keep this running.** Open a **third terminal** for the next step.

### Step 6: Trigger Retries

In a **third terminal**, make a request that will fail and trigger retries:

```bash
export FE_POD_NAME=$(kubectl get pods -n workshop -l app=frontend -o jsonpath='{.items[].metadata.name}')
kubectl exec -it ${FE_POD_NAME} -n workshop -c frontend -- bash

# Make a request (will fail and retry)
curl http://productcatalog:5000/products/ -s -o /dev/null
```

### Step 7: Observe Retries

Go back to your **first terminal** (the one watching logs). You should see:

```
'x-envoy-attempt-count', '1'
'x-envoy-attempt-count', '1'
'x-envoy-attempt-count', '2'
'x-envoy-attempt-count', '2'
'x-envoy-attempt-count', '3'
'x-envoy-attempt-count', '3'
```

✅ **Retries are working!**
- Attempt 1: Initial request (fails)
- Attempt 2: First retry (fails)
- Attempt 3: Second retry (fails)

**Note:** We see **3 attempts** total because Istio counts the initial request plus the 2 retries.

### What Just Happened?

1. Frontend calls productcatalog
2. productcatalog is broken (sleeping, not serving requests)
3. Istio's Envoy proxy detects the failure
4. Envoy automatically retries the request 2 times (as configured)
5. All 3 attempts fail, and the final error is returned to the frontend

### Reset the Environment

Restore productcatalog to normal operation:

```bash
# Delete the broken deployment
kubectl delete deployment productcatalog -n workshop

# Redeploy from Module 1
cd ../../01-getting-started
helm upgrade mesh-basic . -n workshop

# Or reset mesh resources
cd ../03-network-resiliency
kubectl apply -f manifests/setup-mesh-resources/
```

---

## Scenario 3: Circuit Breaking

Circuit breakers protect services from being overwhelmed with traffic. When a service is overloaded, the circuit breaker "trips" and stops sending requests.

In this scenario:
- `catalogdetail` is configured to accept only **1 concurrent connection** and **1 pending request**
- When we send 3 concurrent requests, 1 succeeds and 2 fail with HTTP 503 (circuit breaker tripped)

**Use case:** Protect your database or backend service from being overwhelmed during traffic spikes.

### Step 1: Apply Circuit Breaker Configuration

```bash
#Apply circuit breaker configuration
kubectl apply -f manifests/circuitbreaking/catalogdetail-destinationrule.yaml
```

**Expected output:**
```
destinationrule.networking.istio.io/catalogdetail configured
```

### Step 2: Validate Circuit Breaker Configuration

```bash
kubectl get destinationrule catalogdetail -n workshop -o yaml
```

**Look for this in the output:**
```yaml
spec:
  host: catalogdetail.workshop.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1                 # Only 1 concurrent connection
      http:
        http1MaxPendingRequests: 1        # Only 1 request can wait
        maxRequestsPerConnection: 1
```

**What this configuration does:**
- **maxConnections: 1** - Only 1 active connection allowed at a time
- **http1MaxPendingRequests: 1** - Only 1 request can wait in the queue
- When these limits are exceeded, Istio returns **HTTP 503** immediately

---

### Testing Circuit Breaker

Choose the instructions for your platform:

<details>
<summary><b>🐧 Linux/Mac - Using Fortio</b></summary>

#### Step 1: Create fortio Pod

```bash
kubectl run fortio --image=fortio/fortio:latest_release -n workshop --annotations='proxy.istio.io/config=proxyStatsMatcher:
  inclusionPrefixes:
  - "cluster.outbound"
  - "cluster_manager"
  - "listener_manager"
  - "server"
  - "cluster.xds-grpc"'
```

**Wait for pod to be ready:**
```bash
kubectl wait --for=condition=Ready pod/fortio -n workshop --timeout=60s
```

#### Step 2: Test Single Request (Should Succeed)

```bash
kubectl exec fortio -n workshop -c fortio -- /usr/bin/fortio \
curl http://catalogdetail.workshop.svc.cluster.local:3000/catalogDetail
```

**Expected output:**
```
HTTP/1.1 200 OK
...
{"version":"1","vendors":["ABC.com"]}
```
✅ Single request succeeds

#### Step 3: Test with 2 Concurrent Connections

```bash
kubectl exec fortio -n workshop -c fortio -- \
/usr/bin/fortio load -c 2 -qps 0 -n 20 -loglevel Warning \
http://catalogdetail.workshop.svc.cluster.local:3000/catalogDetail
```

**Expected output:**
```
Code 200 : 15-18 (75-90%)
Code 503 : 2-5  (10-25%)
```
✅ Some requests fail with HTTP 503 (circuit breaker starting to trip)

#### Step 4: Test with 3 Concurrent Connections (Trip Circuit Breaker)

```bash
kubectl exec fortio -n workshop -c fortio -- \
/usr/bin/fortio load -c 3 -qps 0 -n 30 -loglevel Warning \
http://catalogdetail.workshop.svc.cluster.local:3000/catalogDetail
```

**Expected output:**
```
Code 200 : 12 (40.0 %)
Code 503 : 18 (60.0 %)
```
✅ **Circuit breaker is working!** 60% of requests fail because the service is overloaded.

#### Step 5: Verify Circuit Breaker Stats

```bash
kubectl exec fortio -n workshop -c istio-proxy -- \
pilot-agent request GET stats | grep catalogdetail | grep pending
```

**Look for:**
```
cluster.outbound|3000||catalogdetail.workshop.svc.cluster.local.upstream_rq_pending_overflow: 17
```
✅ **17 requests were rejected** by the circuit breaker

#### Clean Up

```bash
kubectl delete pod fortio -n workshop
```

</details>

<details open>
<summary><b>🪟 Windows - Using Hey</b></summary>

#### Step 1: Create hey Pod

```bash
# Create hey pod YAML if it doesn't exist
cat > hey-pod.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: hey-loadtest
  namespace: workshop
  labels:
    app: hey
spec:
  containers:
  - name: hey
    image: williamyeh/hey:latest
    command: ["sleep", "3600"]
  restartPolicy: Never
EOF

# Apply the pod
kubectl apply -f manifests/hey-pod.yaml
```

**Wait for pod to be ready:**
```bash
kubectl wait --for=condition=Ready pod/hey-loadtest -n workshop --timeout=60s
```

#### Step 2: Test Single Request (Should Succeed)

```bash
kubectl exec hey-loadtest -n workshop -- /hey -n 1 \
http://catalogdetail.workshop.svc.cluster.local:3000/catalogDetail
```

**Expected output:**
```
Status code distribution:
  [200] 1 responses
```
✅ Single request succeeds

#### Step 3: Test with 2 Concurrent Workers

```bash
kubectl exec hey-loadtest -n workshop -- /hey -n 6 -c 2 -t 5 \
http://catalogdetail.workshop.svc.cluster.local:3000/catalogDetail
```

**Expected output:**
```
Status code distribution:
  [200] 4-5 responses  (~70-80%)
  [503] 1-2 responses  (~20-30%)
```
✅ Some requests fail with HTTP 503 (circuit breaker starting to trip)

#### Step 4: Test with 3 Concurrent Workers (Trip Circuit Breaker)

```bash
kubectl exec hey-loadtest -n workshop -- /hey -n 10 -c 3 -t 5 \
http://catalogdetail.workshop.svc.cluster.local:3000/catalogDetail
```

**Expected output:**
```
Status code distribution:
  [200] 3 responses   (30%)
  [503] 6 responses   (60%)
```
✅ **Circuit breaker is working!** 60% of requests fail because the service is overloaded.

**Explanation:**
- `-n 10`: Send 10 total requests
- `-c 3`: Use 3 concurrent workers
- `-t 5`: 5-second timeout per request
- Result: Only 3 succeed because catalogdetail only accepts 1 connection at a time

#### Step 5: Verify Circuit Breaker Stats

```bash
kubectl exec hey-loadtest -n workshop -c istio-proxy -- \
pilot-agent request GET stats | grep catalogdetail | grep pending
```

**Look for:**
```
cluster.outbound|3000||catalogdetail.workshop.svc.cluster.local.upstream_rq_pending_overflow: 6
```
✅ **6 requests were rejected** by the circuit breaker

#### Clean Up

```bash
kubectl delete -f hey-pod.yaml
```

</details>

---

### What Just Happened?

1. We configured catalogdetail to accept only **1 concurrent connection**
2. We sent **3 concurrent requests** using fortio/hey
3. Istio's Envoy proxy:
   - Accepted the 1st request (succeeds)
   - Rejected requests 2 and 3 with **HTTP 503** (circuit breaker tripped)
4. This protects catalogdetail from being overwhelmed

```
┌───────────┐    3 concurrent     ┌──────────────┐         ┌──────────────┐
│   Client  │────requests────────▶│Envoy Sidecar │────1───▶│CatalogDetail │
└───────────┘                     └──────────────┘  request └──────────────┘
                                         │
                                         │ Circuit Breaker:
                                         │ - maxConnections: 1
                                         │ - Only 1 allowed
                                         │
                                         ▼
                                  HTTP 503 x 2
                                  (Requests 2 & 3 rejected)
```

### Reset the Environment

```bash
kubectl apply -f manifests/setup-mesh-resources/
```

---

## Understanding the Components

### Timeout Configuration

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: productcatalog
spec:
  hosts:
  - productcatalog
  http:
  - route:
    - destination:
        host: productcatalog
    timeout: 2s              # Fail after 2 seconds
```

### Retry Configuration

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: productcatalog
spec:
  hosts:
  - productcatalog
  http:
  - route:
    - destination:
        host: productcatalog
    retries:
      attempts: 2            # Retry up to 2 times
      perTryTimeout: 2s      # Timeout per attempt
      retryOn: 5xx           # Retry on 5xx errors
```

### Circuit Breaker Configuration

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: catalogdetail
spec:
  host: catalogdetail.workshop.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1                 # Max concurrent connections
      http:
        http1MaxPendingRequests: 1        # Max pending requests
        maxRequestsPerConnection: 1       # Max requests per connection
```

---

## Troubleshooting

### Timeout not working

```bash
# Verify delay is configured on catalogdetail
kubectl get virtualservice catalogdetail -n workshop -o yaml | grep delay

# Verify timeout is configured on productcatalog
kubectl get virtualservice productcatalog -n workshop -o yaml | grep timeout

# Ensure delay > timeout
```

### Retries not visible

```bash
# Check that productcatalog deployment is broken
kubectl get pods -n workshop -l app=productcatalog

# Check retry configuration
kubectl get virtualservice productcatalog -n workshop -o yaml | grep -A 3 retries

# Enable debug logging
istioctl pc log --level debug -n workshop deploy/productcatalog
```

### Circuit breaker not working

```bash
# Check DestinationRule configuration
kubectl get destinationrule catalogdetail -n workshop -o yaml

# Ensure you're sending enough concurrent requests
# Circuit breaker requires c > maxConnections to trip

# Verify pod is running
kubectl get pod fortio -n workshop  # or hey-loadtest
```

### fortio/hey pod not found

```bash
# Check if pod exists
kubectl get pods -n workshop

# Recreate the pod
# For Linux/Mac:
kubectl run fortio --image=fortio/fortio:latest_release -n workshop

# For Windows:
kubectl apply -f manifests/hey-pod.yaml
```

---

## Key Takeaways

✅ **Timeouts** prevent slow services from blocking your application - fail fast!
✅ **Retries** handle transient failures automatically - network blips often succeed on retry
✅ **Circuit Breakers** protect services from overload - stop sending traffic to struggling services
✅ **Observability** - Envoy logs show retry attempts and circuit breaker stats

These three features work together to build **resilient microservices** that gracefully handle failures.

---

## Next Steps

Continue to **[Rate Limiting](../rate-limiting/README.md)** to learn how to:
- Control the rate of requests to your services
- Protect against traffic spikes and abuse
- Implement fair resource allocation

Or proceed to **[Module 4 - Security](../../04-security/)** to learn about:
- Mutual TLS (mTLS) encryption
- Authentication and authorization
- Security policies

---

## Additional Resources

- [Istio Timeouts Documentation](https://istio.io/latest/docs/concepts/traffic-management/#timeouts)
- [Istio Retries Documentation](https://istio.io/latest/docs/concepts/traffic-management/#retries)
- [Istio Circuit Breaking Documentation](https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/)
- [Envoy Circuit Breaking](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking)
