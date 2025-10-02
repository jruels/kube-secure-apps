# Elastic Filesystem Service

## Configure Identity and Access Management

Amazon EKS requires specific permissions to mount storage volumes with the EFS CSI driver.

### Associate an OIDC provider with the cluster

Run the following command, replacing `CLUSTER_NAME` with your cluster’s name:

```sh
eksctl utils associate-iam-oidc-provider   --region us-west-1   --cluster eks-cluster   --approve
```

### Create IAM policies

Enter the lab directory: 
```bash
cd $HOME/Downloads/repos/secure-kube-apps/labs/persistent-storage   
```

Create an IAM policy that allows the Amazon EFS CSI driver to manage EFS resources:


```sh
aws iam create-policy   --policy-name AmazonEKS_EFS_CSI_Driver_Policy   --policy-document file://files/iam-policy-example.json
```

Create a Kubernetes service account with an IAM role attached to this policy.  
Replace `CLUSTER_NAME` and `<ACCOUNT_ID>` with your values:

To get your `<ACCOUNT_ID>`, run: `aws sts get-caller-identity`
```sh
eksctl create iamserviceaccount   --name efs-csi-controller-sa   --namespace kube-system   --cluster CLUSTER_NAME   --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AmazonEKS_EFS_CSI_Driver_Policy   --approve   --override-existing-serviceaccounts   --region us-west-1
```

Create/attach an IRSA role for the node SA  

```bash
eksctl utils associate-iam-oidc-provider --cluster CLUSTER_NAME --region us-west-1 --approve

eksctl create iamserviceaccount \
  --cluster eks-cluster \
  --namespace kube-system \
  --name efs-csi-node-sa \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AmazonEKS_EFS_CSI_Driver_Policy \
  --approve \
  --override-existing-serviceaccounts \
  --region us-west-1
```

---

## Install the EFS CSI Driver

Add the Helm repository:

```sh
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
```

Update the repo:

```sh
helm repo update
```

Install or upgrade the driver:

```bash
helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.create=false \
  --set controller.serviceAccount.name=efs-csi-controller-sa \
  --set node.serviceAccount.create=false \
  --set node.serviceAccount.name=efs-csi-node-sa
```

The `controller.serviceAccount.create=false` flag ensures the driver uses the IAM-enabled service account you created with `eksctl`.

---

## Create EFS Mount Points

Amazon EFS requires mount targets in each Availability Zone where your worker nodes run.

Retrieve your cluster’s VPC ID:

```sh
vpc_id=$(aws eks describe-cluster   --name eks-cluster   --query "cluster.resourcesVpcConfig.vpcId"   --output text)
```

Retrieve the VPC CIDR range:

```sh
cidr_range=$(aws ec2 describe-vpcs   --vpc-ids $vpc_id   --query "Vpcs[].CidrBlock"   --output text)
```

Create a security group to allow NFS (TCP 2049):

```sh
security_group_id=$(aws ec2 create-security-group   --group-name MyEfsSecurityGroup   --description "My EFS security group"   --vpc-id $vpc_id  --query 'GroupId'  --output text)
```

Allow inbound NFS traffic from the VPC CIDR:

```sh
aws ec2 authorize-security-group-ingress   --group-id $security_group_id   --protocol tcp   --port 2049   --cidr $cidr_range
```

Create an EFS file system:

```sh
file_system_id=$(aws efs create-file-system   --region us-west-1   --performance-mode generalPurpose   --query 'FileSystemId'   --output text)
```

Determine the subnets and Availability Zones of your worker nodes:

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$vpc_id" \
  --query "Subnets[*].{SubnetId:SubnetId,AZ:AvailabilityZone}" \
  --output table
```

For each AZ where your nodes are, execute (REPLACE `<subnet-id-in-that-AZ>`): 
```bash
aws efs create-mount-target \
  --file-system-id $file_system_id \
  --subnet-id <subnet-id-in-that-AZ> \
  --security-groups $security_group_id
```

Verify the file system status:

```sh
aws efs describe-file-systems --file-system-id $file_system_id
```

Wait until the `LifeCycleState` changes to `available`.

From one efs-csi-node pod (pick a name from kubectl -n kube-system get pods -l app=efs-csi-node):
```bash
kubectl -n kube-system exec -it <efs-csi-node-pod> -- getent hosts fs-$file_system_id.efs.us-west-1.amazonaws.com
```


---

## Static Provisioning: Create a Persistent Volume

Clone the EFS CSI driver examples:

```sh
git clone https://github.com/kubernetes-sigs/aws-efs-csi-driver.git
```

Navigate to the static provisioning example:

```sh
cd aws-efs-csi-driver/examples/kubernetes/static_provisioning/specs
```

Edit `pv.yaml` and set the `volumeHandle` to your file system ID:

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
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-1234567890abcdef0
```

Apply the PV, PVC, and StorageClass:

```sh
kubectl apply -f pv.yaml
kubectl apply -f claim.yaml
kubectl apply -f storageclass.yaml
```

Verify the PVC is bound:

```sh
kubectl get pv
```

Deploy the sample applications:

```sh
kubectl apply -f pod1.yaml
kubectl apply -f pod2.yaml
```

Wait for the pods to reach `Running`:

```sh
kubectl get pods
```

Verify data sharing:

```sh
kubectl exec app1 -- tail /data/out1.txt
kubectl exec app2 -- tail /data/out1.txt
```

---

## Dynamic Provisioning: Create a StorageClass and PVC

Download a StorageClass manifest for dynamic provisioning:

```sh
curl -o storageclass.yaml https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/storageclass.yaml
```

Edit the file to set your `fileSystemId` and rename it if desired:

```yaml
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-1234567890abcdef0
  directoryPerms: "700"
  gidRangeStart: "1000"
  gidRangeEnd: "2000"
  basePath: "/dynamic_provisioning"
```

Apply the StorageClass:

```sh
kubectl apply -f storageclass.yaml
```

Deploy a pod and PVC:

```sh
curl -o pod.yaml https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/pod.yaml
```

Update the manifest to use your StorageClass if needed, then apply:

```sh
kubectl apply -f pod.yaml
```

Check the PVC and PV:

```sh
kubectl get pvc
kubectl get pv
```

Confirm the pod is running and writing data:

```sh
kubectl get pods -o wide
kubectl exec efs-app -- cat /data/out
```

---

## Cleanup

```sh
kubectl delete -f .
```

Delete the EFS file system and security group if desired:

```sh
aws efs delete-file-system --file-system-id $file_system_id
aws ec2 delete-security-group --group-id $security_group_id
```
