# Deploy multi-tier application

This lab shows you how to build, deploy, and manage a simple, multi-tier web application using Kubernetes.

We will deploy the guestbook demo application, which comprises a Redis leader, Redis follower, and guestbook frontend. After successfully deploying, we will update the application and then roll back to the previous version.

---

## Start up Redis Leader

The guestbook application stores its data in Redis. It writes data to a Redis leader instance and reads data from multiple Redis follower instances.

```bash
cd $HOME/kube-secure-apps/labs/multi-tier
kubectl apply -f manifests/redis-leader-deployment.yaml
kubectl get pods
```

Now let’s check the logs:

```bash
kubectl logs -f <POD NAME>
```

You may see some warning about Transparent Huge Pages (THP). They are safe to ignore.

---

### Create the Redis Leader Service

```bash
kubectl apply -f manifests/redis-leader-service.yaml
kubectl get svc
```

---

## Start up the Redis Followers

Deploy the Redis follower Deployment:

```bash
kubectl apply -f manifests/redis-follower-deployment.yaml
kubectl get pods
```

---

### Create the Redis Follower Service

```bash
kubectl apply -f manifests/redis-follower-service.yaml
kubectl get services
```

---

## Setup and Expose the Guestbook Frontend

### Create the Guestbook Frontend Deployment

> ⚠️ **Deprecation Note:** The `--record` flag shown below is **deprecated** in modern Kubernetes.  
> Instead, you should use the `kubernetes.io/change-cause` annotation to track the reason for each rollout.  
> Both approaches are shown here for clarity — prefer the annotation method on new clusters.

#### Option 1 — Legacy way (deprecated but still works on older clusters)

```bash
kubectl apply --record -f manifests/frontend-deployment.yaml
```

#### Option 2 — Recommended way (modern Kubernetes)

```bash
kubectl apply -f manifests/frontend-deployment.yaml
kubectl annotate deployment frontend kubernetes.io/change-cause="Initial deployment of guestbook frontend"
```

Verify the pods:

```bash
kubectl get pods -l app=guestbook -l tier=frontend
```

---

### Create the Frontend Service

```bash
kubectl apply -f manifests/frontend-service.yaml
kubectl get services
```

---

## Scale the Web Frontend

```bash
kubectl scale deployment frontend --replicas=5
kubectl get pods -l app=guestbook -l tier=frontend

kubectl scale deployment frontend --replicas=2
kubectl get pods -l app=guestbook -l tier=frontend
```

---

## Update Frontend Image

Edit the manifest to change the image tag (e.g., v4 → v5):

```bash
vim manifests/frontend-deployment.yaml
```

Before applying, choose one of these methods:

#### Option 1 — Legacy (deprecated)

```bash
kubectl apply --record -f manifests/frontend-deployment.yaml
```

#### Option 2 — Recommended

```bash
kubectl annotate deployment frontend kubernetes.io/change-cause="Upgrade frontend to v5" --overwrite
kubectl apply -f manifests/frontend-deployment.yaml
```

Verify rollout:

```bash
kubectl get pods -l tier=frontend
kubectl rollout history deployment frontend
```

Repeat to update to `v6`:

```bash
kubectl annotate deployment frontend kubernetes.io/change-cause="Upgrade frontend to v6" --overwrite
kubectl apply -f manifests/frontend-deployment.yaml
```

---

## Rollback Deployment

Check history:

```bash
kubectl rollout history deployment frontend
```

Rollback to previous version:

```bash
kubectl rollout undo deployment frontend
```

Rollback to a specific revision:

```bash
kubectl rollout undo deployment frontend --to-revision=1
```

View details of a revision:

```bash
kubectl rollout history deployment/frontend --revision=<number>
```

---

## Cleanup

```bash
kubectl delete --all -f manifests/
kubectl get pods
```

---

### Summary
- `--record` worked for older clusters but is now **deprecated**.
- Use `kubectl annotate deployment frontend kubernetes.io/change-cause="..."` to record why a change was made.
- `kubectl rollout history` and `kubectl rollout undo` continue to work, now showing your custom change causes from annotations.
