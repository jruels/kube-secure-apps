# Istio Service Mesh Labs

This folder contains Istio labs for learning service mesh concepts on Amazon EKS. Each lab is self-contained with comprehensive instructions and all necessary manifest files.

## Prerequisites

Before starting any lab, ensure you have:

1. **EKS Cluster with Istio Installed**
   - An Amazon EKS cluster running
   - Istio installed via Helm or Terraform Blueprint
   - See the [getting-started](./getting-started/index.md) lab for installation instructions

2. **Required Tools**
   - kubectl
   - helm
   - aws CLI
   - istioctl (for some labs)

3. **For Windows Users**
   - Refer to the Windows compatibility documentation in the main repository
   - Use GitBash or WSL for running commands
   - Install `hey` instead of `siege` for traffic generation

## Lab Order

Complete the labs in the following order:

### 1. [Getting Started](./getting-started/index.md)
**Time: 30-45 minutes**

Learn how to:
- Deploy microservices into an Istio service mesh
- Configure Istio Gateway and VirtualService
- Verify mesh traffic using Kiali
- Understand service-to-service communication

**Prerequisites:** EKS cluster with Istio installed

---

### 2. [Traffic Management](./traffic-management/index.md)
**Time: 45-60 minutes**

Explore advanced traffic management patterns:
- Version-based routing (100% to v1)
- Weight-based routing (canary deployments: 90/10 split)
- Path-based routing (route by URI)
- Header-based routing (route by HTTP headers)
- Traffic mirroring (shadow traffic)

**Prerequisites:** Complete Lab 1 (Getting Started)

---

### 3. [Fault Injection](./fault-injection/index.md)
**Time: 15-20 minutes**

Test application resilience with chaos engineering:
- Delay injection (simulate slow services)
- Abort injection (simulate service failures)
- Test fault tolerance of your microservices

**Prerequisites:** Complete Lab 1 and Lab 2

---

### 4. [Timeouts, Retries, Circuit Breaking](./timeouts-retries-circuitbreaking/index.md)
**Time: 30-40 minutes**

Implement network resiliency patterns:
- **Timeouts:** Prevent waiting indefinitely for slow services
- **Retries:** Automatically retry failed requests
- **Circuit Breaking:** Prevent cascading failures

**Prerequisites:** Complete Lab 1 and Lab 2

---

### 5. [Rate Limiting](./rate-limiting/index.md) *(Optional)*
**Time: 20-30 minutes**

Control request rates to protect services:
- Local rate limiting (simple, no external dependencies)
- Global rate limiting (coordinated across the mesh)

**Prerequisites:** Complete Lab 1 and Lab 2

---

## Lab Structure

Each lab folder contains:

```
lab-name/
├── index.md          # Comprehensive lab instructions
└── manifests/        # All Kubernetes/Istio YAML files
    ├── scenario-1/
    ├── scenario-2/
    └── ...
```

## How to Use These Labs

1. **Clone or navigate to this folder:**
   ```bash
   cd updated-istio-labs
   ```

2. **Choose a lab and navigate to its folder:**
   ```bash
   cd getting-started
   ```

3. **Open the `index.md` file and follow the instructions:**
   ```bash
   # View in terminal
   cat index.md
   
   # Or open in your preferred markdown viewer/editor
   ```

4. **All commands in the lab assume you're in the lab's root directory** (e.g., `updated-istio-labs/getting-started/`)

## Important Notes

### Sequential Dependencies
- **You must complete Lab 1 (Getting Started)** before attempting any other lab
- Lab 2 (Traffic Management) sets up mesh resources used by Labs 3, 4, and 5
- Labs 3, 4, and 5 can be completed in any order after Labs 1 and 2

### Cleanup Between Labs
Some labs modify VirtualServices and DestinationRules. To reset between labs:

```bash
# From within a lab folder that has setup-mesh-resources
kubectl apply -f manifests/setup-mesh-resources/
```

Or redeploy the application:
```bash
cd getting-started
helm uninstall mesh-basic -n workshop
helm install mesh-basic manifests -n workshop
```

### Windows Compatibility
- Use `hey` instead of `siege` for traffic generation
- Use `hey-pod.yaml` for circuit breaking tests (included in relevant labs)
- Some commands may require GitBash or WSL

## Troubleshooting

### Common Issues

1. **Pods not getting sidecars injected:**
   ```bash
   # Verify namespace has istio-injection label
   kubectl get namespace workshop --show-labels
   
   # If missing, add it:
   kubectl label namespace workshop istio-injection=enabled
   ```

2. **Gateway not routing traffic:**
   ```bash
   # Verify Istio ingress gateway is running
   kubectl get pods -n istio-ingress
   
   # Check Gateway selector matches ingress gateway labels
   kubectl get gateway -n workshop productapp-gateway -o yaml
   ```

3. **Services not accessible:**
   ```bash
   # Verify all pods are running with 2/2 containers (app + sidecar)
   kubectl get pods -n workshop
   
   # Check service endpoints
   kubectl get endpoints -n workshop
   ```

### Getting Help
- Each lab has a dedicated troubleshooting section in its `index.md`
- Check the main repository for Windows compatibility guides
- Review Istio documentation: https://istio.io/latest/docs/

## Lab Outcomes

After completing all labs, you will understand:

✅ How to deploy applications in an Istio service mesh
✅ Advanced traffic management patterns (routing, mirroring, canary deployments)
✅ Fault injection for chaos engineering and resilience testing
✅ Network resiliency patterns (timeouts, retries, circuit breaking)
✅ Rate limiting for protecting services from overload

---

## Additional Resources

- **Istio Documentation:** https://istio.io/latest/docs/
- **Kiali Dashboard:** http://localhost:20001 (after port-forwarding)
- **AWS EKS Blueprints:** https://aws-ia.github.io/terraform-aws-eks-blueprints/

---

**Ready to get started?** Begin with [Lab 1: Getting Started](./getting-started/index.md) 🚀
