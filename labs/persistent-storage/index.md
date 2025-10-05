# Elastic Filesystem Service

This lab demonstrates how to configure and use Amazon Elastic File System (EFS) with Amazon EKS using static provisioning. You will create an EFS file system, configure the necessary IAM permissions, install the EFS CSI driver, and deploy a pod that mounts and writes to the EFS volume.

## Prerequisites

Before starting this lab, ensure you have:
- An EKS cluster running with worker nodes
- `helm` installed
- AWS CLI configured with appropriate credentials
- `eksctl` installed

## Set Up Your Environment

```bash
# Save your account ID and Region for later use
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "Your Account ID: $ACCOUNT_ID"

AWS_REGION="us-west-1"
```
Verify your cluster is accessible:

```bash
kubectl get nodes
```

You should see your worker nodes in the `Ready` state.

---

## Configure Identity and Access Management

Amazon EKS requires specific permissions to mount storage volumes with the EFS CSI driver.

### Associate an OIDC Provider with the Cluster

Run the following command to associate an OIDC provider with your cluster:

```bash
eksctl utils associate-iam-oidc-provider \
  --region $AWS_REGION \
  --cluster eks-cluster \
  --approve
```

### Create IAM Policy

Navigate to the lab directory:

```bash
cd $HOME/Downloads/repos/kube-secure-apps/labs/persistent-storage/
```

Create an IAM policy that allows the Amazon EFS CSI driver to manage EFS resources:

```bash
aws iam create-policy \
  --policy-name AmazonEKS_EFS_CSI_Driver_Policy \
  --policy-document file://files/iam-policy-example.json
```

**Note:** If the policy already exists, you'll see an error message. This is safe to ignore - just use the existing policy.

### Create IAM Service Accounts

Create a Kubernetes service account for the EFS CSI controller with an IAM role attached:

```bash
eksctl create iamserviceaccount \
  --name efs-csi-controller-sa \
  --namespace kube-system \
  --cluster eks-cluster \
  --attach-policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AmazonEKS_EFS_CSI_Driver_Policy \
  --approve \
  --override-existing-serviceaccounts \
  --region $AWS_REGION
```

Create a service account for the EFS CSI node daemonset:

```bash
eksctl create iamserviceaccount \
  --cluster eks-cluster \
  --namespace kube-system \
  --name efs-csi-node-sa \
  --attach-policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AmazonEKS_EFS_CSI_Driver_Policy \
  --approve \
  --override-existing-serviceaccounts \
  --region $AWS_REGION
```

Verify the service accounts were created:

```bash
kubectl get sa -n kube-system | grep efs
```

You should see:
```
efs-csi-controller-sa
efs-csi-node-sa
```

---

## Install the EFS CSI Driver

Add the Helm repository:

```bash
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
```

Update the repository:

```bash
helm repo update
```

Install the EFS CSI driver:

```bash
helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=false \
  --set controller.serviceAccount.name=efs-csi-controller-sa \
  --set node.serviceAccount.create=false \
  --set node.serviceAccount.name=efs-csi-node-sa
```

The `controller.serviceAccount.create=false` and `node.serviceAccount.create=false` flags ensure the driver uses the IAM-enabled service accounts you created with `eksctl`.

### Verify EFS CSI Driver Installation

Check that the controller pods are running:

```bash
kubectl get pods -n kube-system -l app=efs-csi-controller
```

You should see 2 controller pods in `Running` state:
```
NAME                                  READY   STATUS    RESTARTS   AGE
efs-csi-controller-6b4d894866-xxxxx   3/3     Running   0          1m
efs-csi-controller-6b4d894866-yyyyy   3/3     Running   0          1m
```

Check that the node daemonset pods are running (one per node):

```bash
kubectl get pods -n kube-system -l app=efs-csi-node
```

You should see one pod per worker node in `Running` state:
```
NAME                 READY   STATUS    RESTARTS   AGE
efs-csi-node-xxxxx   3/3     Running   0          1m
efs-csi-node-yyyyy   3/3     Running   0          1m
```

**Important:** Do not proceed until all EFS CSI pods show `Running` status. If pods are not starting, check the service accounts were created correctly.

---

## Create EFS File System and Mount Points

Amazon EFS requires mount targets in each Availability Zone where your worker nodes run.

### Retrieve VPC Information

Get your cluster's VPC ID:

```bash
vpc_id=$(aws eks describe-cluster \
  --name eks-cluster \
  --region $AWS_REGION \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text)

echo "VPC ID: $vpc_id"
```

Retrieve the VPC CIDR range:

```bash
cidr_range=$(aws ec2 describe-vpcs \
  --vpc-ids $vpc_id \
  --region $AWS_REGION \
  --query "Vpcs[].CidrBlock" \
  --output text)

echo "VPC CIDR: $cidr_range"
```

### Create Security Group

Create a security group to allow NFS traffic (TCP port 2049):

```bash
security_group_id=$(aws ec2 create-security-group \
  --group-name MyEfsSecurityGroup \
  --description "My EFS security group" \
  --vpc-id $vpc_id \
  --region $AWS_REGION \
  --query 'GroupId' \
  --output text)

echo "Security Group ID: $security_group_id"
```

Allow inbound NFS traffic from the VPC CIDR:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id $security_group_id \
  --protocol tcp \
  --port 2049 \
  --cidr $cidr_range \
  --region $AWS_REGION
```

### Create EFS File System

Create the EFS file system:

```bash
file_system_id=$(aws efs create-file-system \
  --region $AWS_REGION \
  --performance-mode generalPurpose \
  --query 'FileSystemId' \
  --output text)

echo "EFS File System ID: $file_system_id"
```

**Important:** Save this File System ID - you'll need it later.

Wait for the EFS file system to become available:

```bash
echo "Waiting for EFS to become available..."
while [ "$(aws efs describe-file-systems \
  --file-system-id $file_system_id \
  --region $AWS_REGION \
  --query 'FileSystems[0].LifeCycleState' \
  --output text)" != "available" ]; do
  echo "Still creating..."
  sleep 10
done
echo "EFS is now available!"
```

### Create Mount Targets

Determine which Availability Zones your worker nodes are in:

```bash
echo "Your nodes are in these Availability Zones:"
kubectl get nodes -o json | \
  jq -r '.items[].metadata.labels["topology.kubernetes.io/zone"]' | \
  sort -u
```

List all subnets in your VPC to identify which subnet corresponds to each AZ:

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$vpc_id" \
  --query "Subnets[*].{SubnetId:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock}" \
  --region $AWS_REGION \
  --output table
```

Example output:
```
------------------------------------------------------------------
|                        DescribeSubnets                         |
+------------------+-------------------+---------------------------+
|        AZ        |       CIDR        |         SubnetId          |
+------------------+-------------------+---------------------------+
|  us-west-1a      |  192.168.0.0/19   |  subnet-08517d3a3f268e352 |
|  us-west-1c      |  192.168.32.0/19  |  subnet-006753ffcf5f3fe74 |
+------------------+-------------------+---------------------------+
```

**Create ONE mount target in EACH Availability Zone where you have worker nodes.**

For example, if your nodes are in `us-west-1a` and `us-west-1c`, run these commands (replace subnet IDs with your actual values):

```bash
# Create mount target in first AZ (replace subnet-id with yours)
aws efs create-mount-target \
  --file-system-id $file_system_id \
  --subnet-id subnet-08517d3a3f268e352 \
  --security-groups $security_group_id \
  --region $AWS_REGION

# Create mount target in second AZ (replace subnet-id with yours)
aws efs create-mount-target \
  --file-system-id $file_system_id \
  --subnet-id subnet-006753ffcf5f3fe74 \
  --security-groups $security_group_id \
  --region $AWS_REGION
```

**Important:** You must create mount targets in ALL Availability Zones where your nodes are running. If a node is in an AZ without a mount target, pods on that node cannot access EFS.

### Verify Mount Targets

Check that mount targets are created and becoming available:

```bash
aws efs describe-mount-targets \
  --file-system-id $file_system_id \
  --region $AWS_REGION \
  --query "MountTargets[*].{AZ:AvailabilityZoneName,Subnet:SubnetId,State:LifeCycleState,IP:IpAddress}" \
  --output table
```

Wait until all mount targets show `State: available` (takes 1-2 minutes):

```bash
echo "Waiting for mount targets to become available..."
while [ $(aws efs describe-mount-targets \
  --file-system-id $file_system_id \
  --region $AWS_REGION \
  --query "MountTargets[?LifeCycleState!='available'] | length(@)" \
  --output text) -gt 0 ]; do
  echo "Still waiting..."
  sleep 10
done
echo "All mount targets are available!"
```

### Test EFS Connectivity

Verify that the EFS CSI node pods can reach the EFS mount targets:

```bash
# Get one of the EFS CSI node pods
POD_NAME=$(kubectl get pods -n kube-system -l app=efs-csi-node -o jsonpath='{.items[0].metadata.name}')

# Test DNS resolution
kubectl -n kube-system exec -it $POD_NAME -- \
  getent hosts ${file_system_id}.efs.${AWS_REGION}.amazonaws.com
```

You should see the IP addresses of your mount targets. If this fails, check that:
- Mount targets are in the `available` state
- Security group allows port 2049 from VPC CIDR
- DNS resolution is working in your VPC

---

## Static Provisioning: Create a Persistent Volume

Now we'll create a persistent volume that uses your EFS file system.

### Clone the EFS CSI Driver Examples

Clone the repository containing example manifests:

```bash
git clone https://github.com/kubernetes-sigs/aws-efs-csi-driver.git
```

Navigate to the static provisioning example directory:

```bash
cd aws-efs-csi-driver/examples/kubernetes/static_provisioning/specs
```

### Edit the Persistent Volume

Edit the `pv.yaml` file to use your EFS file system ID:

```bash
# Display your file system ID
echo "Your EFS File System ID: $file_system_id"

# Edit pv.yaml and replace the volumeHandle with your file system ID
```

Open `pv.yaml` in your editor and change the `volumeHandle` value:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  storageClassName: efs-sc
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-06ffb4102d3d309d7  # Replace with your actual EFS ID
```

**Or use sed to replace it automatically:**

```bash
sed -i.bak "s/volumeHandle:.*/volumeHandle: $file_system_id/" pv.yaml
```

### Deploy the Storage Resources

Apply the StorageClass:

```bash
kubectl apply -f storageclass.yaml
```

Apply the Persistent Volume:

```bash
kubectl apply -f pv.yaml
```

Apply the Persistent Volume Claim:

```bash
kubectl apply -f claim.yaml
```

### Verify the PVC is Bound

Check that the Persistent Volume Claim is bound to the Persistent Volume:

```bash
kubectl get pv,pvc
```

You should see output like:

```
NAME                      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS
persistentvolume/efs-pv   5Gi        RWO            Retain           Bound    default/efs-claim   efs-sc

NAME                              STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
persistentvolumeclaim/efs-claim   Bound    efs-pv   5Gi        RWO            efs-sc
```

**Important:** Do not proceed until the STATUS shows `Bound`. If it shows `Pending`, check:
- The EFS file system ID in pv.yaml is correct
- Mount targets are in the `available` state
- EFS CSI driver pods are running

### Deploy the Sample Application

Deploy the sample pod that will mount the EFS volume:

```bash
kubectl apply -f pod.yaml
```

Watch the pod status:

```bash
kubectl get pods -w
```

Wait until the pod shows `Running` status. Press `Ctrl+C` to exit the watch.

### Verify the Pod is Running

Check the pod status:

```bash
kubectl get pods
```

Expected output:
```
NAME      READY   STATUS    RESTARTS   AGE
efs-app   1/1     Running   0          1m
```

If the pod is stuck in `ContainerCreating` or shows errors, describe it to see what's wrong:

```bash
kubectl describe pod efs-app
```

Common issues:
- **"Failed to resolve" error**: Mount targets aren't available or DNS isn't working
- **"Permission denied" error**: Security group doesn't allow port 2049
- **"serviceaccount not found"**: EFS CSI driver service accounts weren't created

### Verify Data is Being Written to EFS

The sample application writes the current date/time to `/data/out.txt` every 5 seconds.

Check that data is being written:

```bash
kubectl exec efs-app -- tail -10 /data/out.txt
```

You should see timestamps like:

```
Sun Oct 05 20:39:44 UTC 2025
Sun Oct 05 20:39:49 UTC 2025
Sun Oct 05 20:39:54 UTC 2025
```

Wait a few seconds and run the command again - you should see new timestamps, proving the pod is actively writing to the EFS volume.

### Test Persistence Across Pod Restarts

To verify that data persists even when the pod is deleted and recreated:

```bash
# Note the last few lines
kubectl exec efs-app -- tail -3 /data/out.txt

# Delete the pod
kubectl delete pod efs-app

# Recreate it
kubectl apply -f pod.yaml

# Wait for it to be running
kubectl wait --for=condition=ready pod/efs-app --timeout=60s

# Check the data again - you should see the old data plus new entries
kubectl exec efs-app -- tail -10 /data/out.txt
```

The old timestamps should still be there, followed by new ones. This proves your data is persisted in EFS.

---

## Dynamic Provisioning (Optional)

Dynamic provisioning allows Kubernetes to automatically create EFS Access Points for each PersistentVolumeClaim. This section is optional but demonstrates a more advanced use case.

### Download the Dynamic Provisioning StorageClass

```bash
cd ~/
curl -o storageclass-dynamic.yaml https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/storageclass.yaml
```

### Edit the StorageClass

Edit the file to use your EFS file system ID and give it a unique name:

```bash
cat > storageclass-dynamic.yaml <<EOF
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc-dynamic
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: ${file_system_id}
  directoryPerms: "700"
  gidRangeStart: "1000"
  gidRangeEnd: "2000"
  basePath: "/dynamic_provisioning"
EOF
```

Apply the StorageClass:

```bash
kubectl apply -f storageclass-dynamic.yaml
```

### Deploy a Pod with Dynamic Provisioning

Download the pod manifest:

```bash
curl -o pod-dynamic.yaml https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/pod.yaml
```

The downloaded file includes both a PVC and a Pod. Update it to use the dynamic storage class:

```bash
sed -i.bak 's/storageClassName: efs-sc/storageClassName: efs-sc-dynamic/' pod-dynamic.yaml
```

Apply the manifest:

```bash
kubectl apply -f pod-dynamic.yaml
```

### Verify Dynamic Provisioning

Watch the PVC get dynamically bound:

```bash
kubectl get pvc -w
```

You should see a PVC get created and bound to a dynamically created PV.

Check the controller logs to see the dynamic provisioning in action:

```bash
kubectl logs -n kube-system -l app=efs-csi-controller \
  -c csi-provisioner \
  --tail 20
```

Look for a log line like:
```
successfully created PV pvc-xxxxx for PVC efs-claim
```

Verify the pod is running and writing data:

```bash
kubectl get pods

# Once running, check the data
kubectl exec efs-app -- tail /data/out
```

---

## Cleanup

When you're finished with the lab, clean up all resources to avoid charges.

### Delete Kubernetes Resources

Delete the pods, PVCs, and PVs:

```bash
# Delete the static provisioning resources
cd ~/aws-efs-csi-driver/examples/kubernetes/static_provisioning/specs
kubectl delete -f pod.yaml
kubectl delete -f claim.yaml
kubectl delete -f pv.yaml
kubectl delete -f storageclass.yaml

# If you did dynamic provisioning, delete those too
cd ~/
kubectl delete -f pod-dynamic.yaml --ignore-not-found
kubectl delete -f storageclass-dynamic.yaml --ignore-not-found
```

### Delete EFS Mount Targets

**Important:** You must delete mount targets BEFORE deleting the file system, or the deletion will fail.

```bash
echo "Deleting mount targets..."
for mt in $(aws efs describe-mount-targets \
  --file-system-id $file_system_id \
  --region $AWS_REGION \
  --query "MountTargets[*].MountTargetId" \
  --output text); do
  aws efs delete-mount-target \
    --mount-target-id $mt \
    --region $AWS_REGION
  echo "Deleted mount target: $mt"
done
```

Wait for mount targets to be fully deleted (takes 30-60 seconds):

```bash
echo "Waiting for mount targets to be deleted..."
while [ $(aws efs describe-mount-targets \
  --file-system-id $file_system_id \
  --region $AWS_REGION \
  --query "length(MountTargets)" \
  --output text) -gt 0 ]; do
  echo "Still deleting..."
  sleep 10
done
echo "All mount targets deleted!"
```

### Delete EFS File System

Now you can safely delete the EFS file system:

```bash
aws efs delete-file-system \
  --file-system-id $file_system_id \
  --region $AWS_REGION

echo "Deleted EFS file system: $file_system_id"
```

### Delete Security Group

Delete the security group:

```bash
aws ec2 delete-security-group \
  --group-id $security_group_id \
  --region $AWS_REGION

echo "Deleted security group: $security_group_id"
```

### Uninstall EFS CSI Driver (Optional)

If you want to completely remove the EFS CSI driver:

```bash
helm uninstall aws-efs-csi-driver -n kube-system
```

Delete the IAM service accounts:

```bash
eksctl delete iamserviceaccount \
  --cluster eks-cluster \
  --namespace kube-system \
  --name efs-csi-controller-sa \
  --region $AWS_REGION

eksctl delete iamserviceaccount \
  --cluster eks-cluster \
  --namespace kube-system \
  --name efs-csi-node-sa \
  --region $AWS_REGION
```

Delete the IAM policy (optional - only if you're sure no other clusters are using it):

```bash
aws iam delete-policy \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AmazonEKS_EFS_CSI_Driver_Policy
```

---

## Troubleshooting

### Pod Stuck in ContainerCreating

If your pod is stuck in `ContainerCreating` state, describe the pod to see the error:

```bash
kubectl describe pod efs-app
```

**Common errors:**

1. **"Failed to resolve fs-xxxxx.efs.us-west-1.amazonaws.com"**
   - Cause: Mount targets aren't available yet or DNS resolution is failing
   - Fix: Wait for mount targets to reach `available` state, verify VPC DNS settings

2. **"Permission denied"**
   - Cause: Security group doesn't allow NFS traffic (port 2049)
   - Fix: Verify security group ingress rule allows TCP 2049 from VPC CIDR

3. **"serviceaccount 'efs-csi-node-sa' not found"**
   - Cause: Node service account wasn't created
   - Fix: Re-run the `eksctl create iamserviceaccount` command for efs-csi-node-sa

4. **"Mount failed: no mount targets found"**
   - Cause: No mount targets in the AZ where the pod's node is running
   - Fix: Create mount targets in ALL AZs where you have worker nodes

### PVC Stuck in Pending

If your PVC shows `Pending` status:

```bash
kubectl describe pvc efs-claim
```

Check:
- EFS file system ID in pv.yaml matches your actual EFS ID
- PV was created successfully: `kubectl get pv`
- StorageClass exists: `kubectl get sc`

### EFS CSI Driver Pods Not Running

Check pod status:

```bash
kubectl get pods -n kube-system -l app=efs-csi-controller
kubectl get pods -n kube-system -l app=efs-csi-node
```

If pods are failing, check their logs:

```bash
kubectl logs -n kube-system -l app=efs-csi-controller --tail 50
kubectl logs -n kube-system -l app=efs-csi-node --tail 50
```

Common issues:
- Service accounts not created or incorrect IAM permissions
- Image pull errors (check network connectivity)

---

## Summary

In this lab, you:

1. Configured IAM permissions for the EFS CSI driver
2. Created IAM service accounts for both controller and node components
3. Installed the EFS CSI driver using Helm
4. Created an EFS file system and mount targets
5. Used static provisioning to create a Persistent Volume backed by EFS
6. Deployed a pod that successfully mounts and writes to the EFS volume
7. Verified data persistence across pod restarts
8. (Optional) Explored dynamic provisioning with EFS Access Points
9. Cleaned up all resources properly

**Key Takeaways:**

- EFS provides shared, persistent storage for Kubernetes pods
- Mount targets must exist in every AZ where worker nodes run
- Static provisioning gives you full control over the EFS file system
- Dynamic provisioning automatically creates isolated access points per PVC
- Proper cleanup requires deleting mount targets before the file system
