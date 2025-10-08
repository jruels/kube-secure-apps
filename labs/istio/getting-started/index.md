# Module 1 - Getting Started

This module demonstrates how to deploy microservices as part of an Istio service mesh on Amazon EKS. You will deploy a sample application consisting of multiple microservices that communicate through the Istio service mesh.

---

## What You'll Learn

- How to deploy applications into an Istio service mesh
- How to configure Istio Gateway and VirtualService resources
- How to verify mesh traffic using Kiali
- How service-to-service communication works in Istio

---

## Architecture

The application consists of four microservices:
- **frontend** - Web UI for the application
- **productcatalog** - Manages product listings
- **catalogdetail** - Provides detailed product information (deployed with v1 and v2 versions)

All services are automatically injected with Istio sidecar proxies, enabling mesh features like traffic management, security, and observability.

---

## Prerequisites

### 1. EKS Cluster with Istio Installed

In VS Code open a new terminal and select the `GitBash` profile.

Follow these steps: 

Install Istio via Helm:
```bash
# Add Istio Helm repository
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

# Install Istio base components
kubectl create namespace istio-system
helm install istio-base istio/base -n istio-system --set defaultRevision=default

# Install Istio control plane (istiod)
helm install istiod istio/istiod -n istio-system --wait

# Install Istio ingress gateway
kubectl create namespace istio-ingress
kubectl label namespace istio-ingress istio-injection=enabled
helm install istio-ingress istio/gateway -n istio-ingress --wait

# Verify installation
kubectl get pods -n istio-system
kubectl get pods -n istio-ingress
```

Expected output:
```
NAME                      READY   STATUS    RESTARTS   AGE
istiod-xxxxxxxxx-xxxxx    1/1     Running   0          2m

NAME                            READY   STATUS    RESTARTS   AGE
istio-ingress-xxxxxxxxx-xxxxx   1/1     Running   0          1m
```

### 3. Required Tools

Install the following tools on your local machine:

For Windows (using Chocolatey in PowerShell as Administrator):**
```powershell
# Install some tools with chocolatey
choco install -y jq
choco install -y hey 

# Configure Git for proper line endings
git config --global core.autocrlf input
git config --global core.eol lf
```

**Verify tool installations:**
```bash
kubectl version --client
helm version
jq --version
hey -version
```

### 4. Verify Istio Installation

Before proceeding, verify Istio is properly installed:
```bash
# Check Istio system pods
kubectl get pods -n istio-system
# Expected: istiod pod should be Running

# Check Istio ingress gateway
kubectl get pods -n istio-ingress
# Expected: istio-ingress pod should be Running

# Check ingress gateway service
kubectl get svc -n istio-ingress
# Expected: LoadBalancer service with EXTERNAL-IP
```

---

## Deploy the Application

### Step 1: Navigate to lab Directory

```bash
# Clone this repository if you haven't already
git clone https://github.com/jruels/kube-secure-apps.git
cd $HOME/Downloads/repos/kube-secure-apps/labs/istio/getting-started
```

### Step 2: Create and Label the Workshop Namespace

Create a dedicated namespace for the workshop applications and enable Istio sidecar injection:

```bash
# Create the namespace
kubectl create namespace workshop

# Enable automatic Istio sidecar injection
kubectl label namespace workshop istio-injection=enabled
```

**What this does:**
- Creates an isolated namespace called `workshop` for our applications
- The `istio-injection=enabled` label tells Istio to automatically inject sidecar proxies into all pods deployed in this namespace

**Expected output:**
```
namespace/workshop created
namespace/workshop labeled
```

**Verify the label:**
```bash
kubectl get namespace workshop --show-labels
```

Expected output should include `istio-injection=enabled` in the labels.

### Step 3: Deploy the Microservices Using Helm

Install all microservices with a single Helm command:

```bash
helm install mesh-basic manifests -n workshop
```

**What this does:**
- Deploys frontend, productcatalog, catalogdetail (v1 and v2) services
- Creates Kubernetes Deployments, Services, and ServiceAccounts
- Creates Istio Gateway and VirtualService for ingress traffic
- All pods automatically get Istio sidecar containers injected

**Expected output:**
```
NAME: mesh-basic
LAST DEPLOYED: [timestamp]
NAMESPACE: workshop
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get the application URL by running the following command:

   ISTIO_INGRESS_URL=$(kubectl get svc istio-ingress -n istio-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')
   echo "http://$ISTIO_INGRESS_URL"

2. Access the displayed URL in a terminal using cURL or via a browser window

Note: It may take a few minutes for the istio-ingress Network LoadBalancer to associate to the instance-mode targetGroup after the application is deployed.
```

---

## Validate the Deployment

### Step 1: Check Pod Status

Wait for all pods to be in `Running` state with 2/2 containers ready:

```bash
kubectl get pods -n workshop
```

**Expected output:**
```
NAME                              READY   STATUS    RESTARTS   AGE
catalogdetail-xxxxxxxxx-xxxxx     2/2     Running   0          2m
catalogdetail2-xxxxxxxxx-xxxxx    2/2     Running   0          2m
frontend-xxxxxxxxx-xxxxx          2/2     Running   0          2m
productcatalog-xxxxxxxxx-xxxxx    2/2     Running   0          2m
```

**Key points:**
- `READY 2/2` means each pod has 2 containers: the application container + Istio sidecar proxy
- All pods should be in `Running` status
- If pods are in `Pending` or `Init` state, wait a minute and check again

**Troubleshooting:**
If pods are not starting:
```bash
# Check pod details
kubectl describe pod <pod-name> -n workshop

# Check pod logs
kubectl logs <pod-name> -n workshop -c <container-name>

# Check events
kubectl get events -n workshop --sort-by='.lastTimestamp'
```

### Step 2: Verify Istio Sidecar Injection

Confirm that Istio sidecars were automatically injected:

```bash
# Check one of the pods in detail
kubectl get pod <pod-name> -n workshop -o jsonpath='{.spec.containers[*].name}'
```

Expected output should show two containers:
```
<app-name> istio-proxy
```

### Step 3: Check Istio Resources

Verify that Istio Gateway and VirtualService were created:

```bash
kubectl get Gateway,VirtualService,DestinationRule -n workshop
```

**Expected output:**
```
NAME                                             AGE
gateway.networking.istio.io/productapp-gateway   2m

NAME                                            GATEWAYS               HOSTS   AGE
virtualservice.networking.istio.io/productapp   ["productapp-gateway"] ["*"]   2m
```

**What these resources do:**
- **Gateway**: Configures the Istio ingress gateway to accept HTTP traffic on port 80
- **VirtualService**: Routes incoming traffic from the gateway to the frontend service

### Step 4: Get the Application URL

Retrieve the external URL for accessing the application:

```bash
# Get the ingress gateway URL
ISTIO_INGRESS_URL=$(kubectl get svc istio-ingress -n istio-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')
echo "Application URL: http://$ISTIO_INGRESS_URL"
```

**Note:** It may take 2-3 minutes for the AWS Load Balancer to become fully available and start routing traffic.

### Step 5: Test Application Access

Test the application with curl:

```bash
# Test with curl
curl -I http://$ISTIO_INGRESS_URL

# Or open in browser (Mac/Linux)
open http://$ISTIO_INGRESS_URL

# Windows
start http://$ISTIO_INGRESS_URL
```

**Expected response:**
```
HTTP/1.1 200 OK
...
```

If you get a connection error, wait a minute for the Load Balancer to fully initialize, then try again.

---

## Observability with Kiali

Kiali provides a visual representation of your service mesh, showing service topology, traffic flow, and health status.

### Step 1: Install Kiali (if not already installed)

```bash
# Install Kiali and other observability tools
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/kiali.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/grafana.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/jaeger.yaml

# Wait for Kiali to be ready
kubectl rollout status deployment/kiali -n istio-system
```

### Step 2: Access Kiali Dashboard

Open a port-forward to access Kiali on your local machine:

```bash
kubectl port-forward svc/kiali 20001:20001 -n istio-system
```

**Keep this terminal window open.** Open a new terminal for the next steps.

Navigate to **http://localhost:20001** in your web browser.

**In the Kiali console:**
1. Click on **Graph** in the left navigation
2. Select namespace: **workshop**
3. Display: Check **Traffic Animation**
4. Click the refresh icon to see live traffic

![Kiali Console](../../images/01-kiali-console.png)

---

## Generate Traffic to the Application

To visualize traffic flow in Kiali, generate load against the application.

### For Windows (using hey):

```bash
# Get the ingress URL
ISTIO_INGRESS_URL=$(kubectl get svc istio-ingress -n istio-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')

# Generate traffic for 2 minutes with 5 concurrent workers at 1 req/sec each
# -z 2m: run for 2 minutes
# -c 5: 5 concurrent workers
# -q 1: 1 query per second per worker
hey -z 2m -c 5 -q 1 "http://$ISTIO_INGRESS_URL"
```

### Observe Traffic in Kiali

While traffic is being generated, switch to the Kiali dashboard at http://localhost:20001:

1. You should see traffic flowing from **istio-ingress** → **productapp-gateway** → **frontend** → **productcatalog** → **catalogdetail**
2. The **catalogdetail** service shows traffic split between **v1** and **v2** versions (approximately 50/50)

![Traffic Flow](../../images/01-kiali-traffic-flow.gif)

**What you're observing:**
1. **Ingress traffic** enters through the Istio ingress gateway
2. The **productapp-gateway** Gateway resource captures all HTTP traffic (host: *)
3. The **productapp** VirtualService routes traffic to the **frontend** service
4. **frontend** calls **productcatalog**
5. **productcatalog** calls **catalogdetail**
6. Traffic to **catalogdetail** is randomly distributed between v1 and v2 versions

---

## Understanding the Components

### Application Services

| Service | Purpose | Versions |
|---------|---------|----------|
| frontend | Web UI for the product catalog | v1 |
| productcatalog | Aggregates product data | v1 |
| catalogdetail | Provides detailed product information | v1, v2 |

### Istio Resources

| Resource Type | Name | Purpose |
|---------------|------|---------|
| Gateway | productapp-gateway | Configures ingress gateway to accept HTTP traffic on port 80 |
| VirtualService | productapp | Routes traffic from gateway to frontend service |

### Why Two Containers Per Pod?

Each pod runs two containers:
1. **Application container**: Your microservice (frontend, productcatalog, etc.)
2. **istio-proxy container**: Envoy sidecar that intercepts all network traffic

The sidecar enables Istio features like:
- Traffic routing and load balancing
- Mutual TLS encryption
- Telemetry and distributed tracing
- Circuit breaking and retries

---

## Troubleshooting

### Pods Not Starting

```bash
# Check pod status
kubectl get pods -n workshop

# Describe a problematic pod
kubectl describe pod <pod-name> -n workshop

# Check logs
kubectl logs <pod-name> -n workshop -c <container-name>

# Check if namespace has istio-injection enabled
kubectl get namespace workshop --show-labels
```

### Application Not Accessible

```bash
# Check ingress gateway is running
kubectl get pods -n istio-ingress

# Check service has external IP
kubectl get svc -n istio-ingress
# EXTERNAL-IP should show a hostname (may take 2-3 minutes)

# Check gateway configuration
kubectl get gateway productapp-gateway -n workshop -o yaml

# Verify the gateway selector matches the ingress gateway labels
kubectl get pods -n istio-ingress --show-labels
# Should see: istio=ingress
```

### Kiali Not Showing Traffic

```bash
# Verify Kiali is running
kubectl get pods -n istio-system | grep kiali

# Check Prometheus is running (Kiali depends on it)
kubectl get pods -n istio-system | grep prometheus

# Restart port-forward
kubectl port-forward svc/kiali 20001:20001 -n istio-system
```

### Connection Refused or Timeout

- Wait 2-3 minutes for AWS Load Balancer to finish provisioning
- Check security groups allow inbound traffic on port 80
- Verify your kubeconfig is pointing to the correct cluster


