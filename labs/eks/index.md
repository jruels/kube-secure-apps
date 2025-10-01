# Create an Elastic Kubernetes Cluster (EKS)
In this lab, you will create a Kubernetes cluster on AWS (EKS)

## Prerequisites
### Launch VS Code in Administrator Mode to install software in a Terminal window

### Step 1: Run VS Code as Administrator

1. Close any open instances of Visual Studio Code.
2. Search for "Visual Studio Code" in the Start menu.
3. Right-click on it and select **Run as Administrator**.
   - If prompted by User Account Control (UAC), click **Yes** to allow it.

---

### **Step 2: Open the Integrated Terminal**

1. In VS Code, click on the **Terminal** menu at the top and select **New Terminal**.
   - Alternatively, use the shortcut: `Ctrl + ~`.
2. Ensure the terminal is using **PowerShell**, as Chocolatey requires it.

---

### **Step 3: Use Chocolatey to install needed packages**

1. Install some required packages:

   ```powershell
   choco install eksctl kubernetes-helm -y
   ```

---

### **Important Note:**

Always run VS Code in **Administrator mode** whenever you need to use Chocolatey to install or manage software that requires system-level changes.

After completing the installation with Chocolatey, close VS Code, and relaunch it as a normal user.

## Deploy EKS cluster 
Now that all the utilities are installed, let's start creating a cluster! 

```sh
eksctl create cluster \
--name eks-cluster \
--nodegroup-name standard-workers \
--node-type t3.medium \
--nodes 3 \
--nodes-min 1 \
--nodes-max 3
```

After the command completes, list the nodes: 
```sh
kubectl get nodes
```

Now list the running pods:
```sh
kubectl get pods -A
```

## Lab Complete
