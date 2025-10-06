# StatefulSets
StatefulSets manage the deployment and scaling of a set of Pods, and provides guarantees about the ordering and uniqueness of these Pods, suitable for applications that require one or more of the following.
* Stable, unique network identifiers
* Stable, persistent storage
* Ordered, graceful deployment and scaling
* Ordered, automated rolling updates

In this lab you are deploying a MySQL database using `StatefulSet` and AWS EBS volumes as persistent storage.

### Prerequisites
Verify your EKS cluster is running and kubectl is configured:
```sh
kubectl cluster-info
```

### Install EBS CSI Driver
EKS requires the AWS EBS CSI driver addon to provision EBS volumes dynamically. Install it using the AWS CLI:

```sh

# Install the EBS CSI driver addon
aws eks create-addon \
  --cluster-name eks-cluster \
  --addon-name aws-ebs-csi-driver \
  --region us-west-1
```

Wait for the addon to be active:
```sh
aws eks describe-addon \
  --cluster-name eks-cluster \
  --addon-name aws-ebs-csi-driver \
  --region us-west-1 \
  --query 'addon.status' \
  --output text
```

The status should show `ACTIVE`.

The node IAM role needs permissions for the EBS CSI driver. Attach the required policy:

```sh
# Get your node instance role name
aws iam list-roles --query 'Roles[?contains(RoleName, `NodeInstanceRole`)].RoleName' --output text

# Attach the EBS CSI driver policy (replace with your actual role name from above)
aws iam attach-role-policy \
  --role-name <YOUR_NODE_INSTANCE_ROLE_NAME> \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
```

After attaching the policy, restart the EBS CSI controller pods:
```sh
kubectl rollout restart deployment -n kube-system ebs-csi-controller
```

Verify the EBS CSI driver pods are running:
```sh
kubectl get pods -n kube-system -l app=ebs-csi-controller
kubectl get pods -n kube-system -l app=ebs-csi-node
```

All controller pods should show `Running` status with `6/6` containers ready. Node pods should show `3/3` containers ready.

Verify the `gp2` storage class is available:
```sh
kubectl get storageclass gp2
```

### Setup
Create a working directory for all of our manifest files.
```sh
mkdir -p ${HOME}/Downloads/labs/stateful
cd ${HOME}/Downloads/labs/stateful
```

### Create the mysql Namespace
We will create a new Namespace called `mysql` that will host all the components.

```sh
kubectl create namespace mysql
```

### Create ConfigMap
A ConfigMap allows you to decouple configuration artifacts and secrets from image content to keep containerized applications portable. Using ConfigMaps, you can independently control the MySQL configuration.

Run the following commands to create the ConfigMap.

```sh
cat << EoF > ${HOME}/Downloads/labs/stateful/mysql-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: mysql
  labels:
    app: mysql
data:
  leader.cnf: |
    # Apply this config only on the leader.
    [mysqld]
    log-bin
  follower.cnf: |
    # Apply this config only on followers.
    [mysqld]
    super-read-only
EoF
```

The ConfigMap stores `leader.cnf` and `follower.cnf` and passes them when initializing leader and follower pods defined in `StatefulSet`:
* **leader.cnf** is for the MySQL leader pod which has binary log option (log-bin) to provide a record of the data changes to be sent to follower servers.
* **follower.cnf** is for follower pods which have super-read-only option.

Create `mysql-config` ConfigMap.
```sh
kubectl apply -f ${HOME}/Downloads/labs/stateful/mysql-configmap.yaml
```

### Create Services
Services can be exposed in different ways by specifying a `type` in the `serviceSpec`. `StatefulSet` currently requires a Headless Service to control the domain of its Pods, directly reach each Pod with stable DNS entries.

By specifying **"None"** for the clusterIP, you can create a Headless Service.

Create the mysql services file:
```sh
cat << EoF > ${HOME}/Downloads/labs/stateful/mysql-services.yaml
# Headless service for stable DNS entries of StatefulSet members.
apiVersion: v1
kind: Service
metadata:
  namespace: mysql
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
---
# Client service for connecting to any MySQL instance for reads.
# For writes, you must instead connect to the leader: mysql-0.mysql.
apiVersion: v1
kind: Service
metadata:
  namespace: mysql
  name: mysql-read
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
EoF
```

You can see the **mysql** service is for DNS resolution so that when pods are placed by StatefulSet controller, pods can be resolved using ``pod-name.mysql``. **mysql-read** is a client service that does load balancing for all followers.

Create service `mysql` and `mysql-read` by executing the following command
```sh
kubectl apply -f ${HOME}/Downloads/labs/stateful/mysql-services.yaml
```

### Create StatefulSet
StatefulSet consists of serviceName, replicas, template and volumeClaimTemplates:

* **serviceName** is "mysql", headless service we created in previous section
* **replicas** is 2, the desired number of pods
* **template** is the configuration of pod
* **volumeClaimTemplates** is to claim volume for pod

Create the StatefulSet manifest:
```sh
cat << 'EoF' > ${HOME}/Downloads/labs/stateful/mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  namespace: mysql
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 2
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:5.7
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Generate mysql server-id from pod ordinal index.
          [[ `uname -n` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # Add an offset to avoid reserved server-id=0 value.
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # Copy appropriate conf.d files from config-map to emptyDir.
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/leader.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/follower.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      - name: clone-mysql
        image: gcr.io/google-samples/xtrabackup:1.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Skip the clone if data already exists.
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          # Skip the clone on leader (ordinal index 0).
          [[ `uname -n` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] && exit 0
          # Clone data from previous peer.
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # Prepare the backup.
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: "1"
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - "[ ! -f /tmp/mysql-not-ready ] && mysql -h 127.0.0.1 -e 'SELECT 1'"
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
      - name: xtrabackup
        image: gcr.io/google-samples/xtrabackup:1.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql

          # Determine binlog position of cloned data, if any.
          if [[ -f xtrabackup_slave_info ]]; then
            # XtraBackup already generated a partial "CHANGE MASTER TO" query
            # because we're cloning from an existing follower.
            mv xtrabackup_slave_info change_master_to.sql.in
            # Ignore xtrabackup_binlog_info in this case (it's useless).
            rm -f xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # We're cloning directly from leader. Parse binlog position.
            [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm xtrabackup_binlog_info
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi

          # Check if we need to complete a clone by starting replication.
          if [[ -f change_master_to.sql.in ]]; then
            echo "Waiting for mysqld to be ready (accepting connections)"
            until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done

            echo "Initializing replication from clone position"
            # In case of container restart, attempt this at-most-once.
            mv change_master_to.sql.in change_master_to.sql.orig
            mysql -h 127.0.0.1 <<EOF
          $(<change_master_to.sql.orig),
            MASTER_HOST='mysql-0.mysql',
            MASTER_USER='root',
            MASTER_PASSWORD='',
            MASTER_CONNECT_RETRY=10;
          START SLAVE;
          EOF
          fi

          # Start a server to send backups when requested by peers.
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp2
      resources:
        requests:
          storage: 10Gi
EoF
```

The StatefulSet uses the `gp2` storage class, which provisions AWS EBS volumes for persistent storage.

Apply the StatefulSet:
```sh
kubectl apply -f ${HOME}/Downloads/labs/stateful/mysql-statefulset.yaml
```

Watch StatefulSet deployment status
```sh
kubectl -n mysql rollout status statefulset mysql
```

It will take a few minutes for pods to initialize and the `StatefulSet` to be created.

Output:
```
Waiting for 2 pods to be ready...
Waiting for 1 pods to be ready...
partitioned roll out complete: 2 new pods have been updated...
```

Open another terminal and watch the progress of pods creation using the following command.
```sh
kubectl -n mysql get pods -l app=mysql --watch
```

You can see ordered, graceful deployment with a stable, unique name for each pod.

```
NAME      READY   STATUS           RESTARTS    AGE
mysql-0   0/2     Init:0/2          0          16s
mysql-0   0/2     Init:1/2          0          17s
mysql-0   0/2     PodInitializing   0          18s
mysql-0   1/2     Running           0          19s
mysql-0   2/2     Running           0          25s
mysql-1   0/2     Pending           0          0s
mysql-1   0/2     Pending           0          0s
mysql-1   0/2     Init:0/2          0          0s
mysql-1   0/2     Init:1/2          0          10s
mysql-1   0/2     PodInitializing   0          11s
mysql-1   1/2     Running           0          12s
mysql-1   2/2     Running           0          16s
```

Press `Ctrl+C` to stop watching.

Check the dynamically created PVC
```sh
kubectl -n mysql get pvc -l app=mysql
```

We can see `data-mysql-0`, and `data-mysql-1` have been created.

Output:
```
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-mysql-0   Bound    pvc-62eb26d0-7d0e-4e37-af8c-96f451d070dc   10Gi       RWO            gp2            22m
data-mysql-1   Bound    pvc-8b355427-7487-4515-897e-685efeddc290   10Gi       RWO            gp2            21m
```

### Test MySQL
You can use **mysql-client** to send some data to the leader, **mysql-0.mysql** by running the following command.

```sh
kubectl -n mysql run mysql-client --image=mysql:5.7 -i --rm --restart=Never --\
  mysql -h mysql-0.mysql <<EOF
CREATE DATABASE test;
CREATE TABLE test.messages (message VARCHAR(250));
INSERT INTO test.messages VALUES ('hello, from mysql-client');
EOF
```

Run the following to test follower `mysql-read` received the data.

```sh
kubectl -n mysql run mysql-client --image=mysql:5.7 -it --rm --restart=Never --\
  mysql -h mysql-read -e "SELECT * FROM test.messages"
```

Output:
```
+--------------------------+
| message                  |
+--------------------------+
| hello, from mysql-client |
+--------------------------+
```

To test load balancing across followers, run the following command.
```sh
kubectl -n mysql run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
   bash -ic "while sleep 1; do mysql -h mysql-read -e 'SELECT @@server_id,NOW()'; done"
```

Each MySQL instance is assigned a unique identifier, and it can be retrieved using `@@server_id`. It will print the server id serving the request and the timestamp.

```
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         101 | 2021-02-21 19:17:52 |
+-------------+---------------------+
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         101 | 2021-02-21 19:17:53 |
+-------------+---------------------+
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         100 | 2021-02-21 19:17:54 |
+-------------+---------------------+
```

Leave this open in a separate window while you test failure in the next section.

### Test Readiness Probe Failure
MySQL container uses readiness probe by running `mysql -h 127.0.0.1 -e 'SELECT 1'` on the server to make sure MySQL server is still active. Open a new terminal and simulate MySQL as being unresponsive.

```sh
kubectl -n mysql exec mysql-1 -c mysql -- touch /tmp/mysql-not-ready
```

This command creates a file `/tmp/mysql-not-ready` which will cause the readiness probe to fail. The readiness probe is configured to check for the absence of this file. During the next health check, the pod should report that its MySQL container is not ready.

```sh
kubectl -n mysql get pod mysql-1
```

Output:
```
NAME      READY     STATUS    RESTARTS   AGE
mysql-1   1/2       Running   0          12m
```

Notice only one container is in a `READY` state.

**mysql-read** load balancer detects failures and takes action by not sending traffic to the failed container, `@@server_id 101`. You can check this by viewing the loop running in the separate window from previous section. The loop shows the following output.

```
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         100 | 2020-01-25 17:32:19 |
+-------------+---------------------+
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         100 | 2020-01-25 17:32:20 |
+-------------+---------------------+
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         100 | 2020-01-25 17:32:21 |
+-------------+---------------------+
```

Notice it does not read `@@server_id 101`

Fix the mysql server
```sh
kubectl -n mysql exec mysql-1 -c mysql -- rm /tmp/mysql-not-ready
```

Check the status again to see that both containers are running and healthy

```sh
kubectl -n mysql get pod mysql-1
```

Output:
```
NAME      READY     STATUS    RESTARTS   AGE
mysql-1   2/2       Running   0          5h
```

The loop in another terminal is now showing` @@server_id 101` is back and all servers are running. Press `Ctrl+C` to stop watching.

### Test Pod Failure and Recovery
To simulate a failed pod, delete mysql-1
```sh
kubectl -n mysql delete pod mysql-1
```

```
pod "mysql-1" deleted
```

StatefulSet controller recognizes failed pod and creates a new one to maintain the number of replicas with the same name and link to the same `PersistentVolumeClaim`.

```sh
kubectl -n mysql get pod mysql-1 -w
```

Output
```
NAME      READY   STATUS        RESTARTS   AGE
mysql-1   2/2     Terminating   0          15m
mysql-1   0/2     Terminating   0          16m
mysql-1   0/2     Terminating   0          16m
mysql-1   0/2     Terminating   0          16m
mysql-1   0/2     Pending       0          0s
mysql-1   0/2     Pending       0          0s
mysql-1   0/2     Init:0/2      0          0s
mysql-1   0/2     Init:1/2      0          11s
mysql-1   0/2     PodInitializing   0          12s
mysql-1   1/2     Running           0          13s
mysql-1   2/2     Running           0          18s
```

### Test Scaling
More followers can be added to the MySQL Cluster to increase read capacity.
```sh
kubectl -n mysql scale statefulset mysql --replicas=3
```

Watch the progress of ordered and graceful scaling.

```sh
kubectl -n mysql rollout status statefulset mysql
```

Output:
```
Waiting for 1 pods to be ready...
partitioned roll out complete: 3 new pods have been updated...
```

In another terminal watch the new pod come online
```sh
kubectl -n mysql get pods -l app=mysql --watch
```

To exit type `Ctrl+C`

If you stopped the loop start it again.
```sh
kubectl -n mysql run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
   bash -ic "while sleep 1; do mysql -h mysql-read -e 'SELECT @@server_id,NOW()'; done"
```

You will now see 3 servers running.

Output:
```
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         100 | 2020-01-25 02:32:43 |
+-------------+---------------------+
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         102 | 2020-01-25 02:32:44 |
+-------------+---------------------+
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         101 | 2020-01-25 02:32:45 |
+-------------+---------------------+
```

Verify if the newly deployed follower `mysql-2` has the same data set.
```sh
kubectl -n mysql run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
 mysql -h mysql-2.mysql -e "SELECT * FROM test.messages"
```

It will show the same data that the leader has.

Output:
```
+--------------------------+
| message                  |
+--------------------------+
| hello, from mysql-client |
+--------------------------+
```

Scale the replicas to 2

```sh
kubectl -n mysql scale statefulset mysql --replicas=2
```

You can see that it removed the last added replica.
```
kubectl -n mysql get pods -l app=mysql
```

Output:
```
NAME      READY     STATUS    RESTARTS   AGE
mysql-0   2/2       Running   0          1d
mysql-1   2/2       Running   0          1d
```
Confirm the pvcs still exist
```sh
kubectl -n mysql  get pvc -l app=mysql
```

Output:
```
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-mysql-0   Bound    pvc-62eb26d0-7d0e-4e37-af8c-96f451d070dc   10Gi       RWO            gp2            18m
data-mysql-1   Bound    pvc-8b355427-7487-4515-897e-685efeddc290   10Gi       RWO            gp2            17m
data-mysql-2   Bound    pvc-b71c4122-809f-42ad-82ce-d1c1060365f4   10Gi       RWO            gp2            9m35s
```

### Bonus (Change reclaim policy)
By default, deleting a `PersistentVolumeClaim` will delete its associated persistent volume. What if you wanted to keep the volume?

Change the reclaim policy:
Find the `PersistentVolume` attached to the `PersistentVolumeClaim` `data-mysql-2`

```sh
kubectl -n mysql get pvc data-mysql-2 -o jsonpath='{.spec.volumeName}'
```

Copy the PV name from the output and use it in the following command:

```sh
# Replace <PV_NAME> with the actual PV name from above
kubectl patch pv <PV_NAME> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

Verify the `ReclaimPolicy` was updated

```sh
kubectl get pv <PV_NAME> -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'
```

You should see `Retain` as the output.

Now, if you delete the `PersistentVolumeClaim` `data-mysql-2`, the EBS volume will be retained and can be viewed in the AWS EC2 console under Elastic Block Store > Volumes.

Let's change the reclaim policy back to "Delete" to avoid orphaned volumes:

```sh
kubectl patch pv <PV_NAME> -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
```

Delete `data-mysql-2`

```sh
kubectl -n mysql delete pvc data-mysql-2
```

Output:
```
persistentvolumeclaim "data-mysql-2" deleted
```

## Cleanup
```sh
kubectl delete \
  -f ${HOME}/Downloads/labs/stateful/mysql-statefulset.yaml \
  -f ${HOME}/Downloads/labs/stateful/mysql-services.yaml \
  -f ${HOME}/Downloads/labs/stateful/mysql-configmap.yaml

# Delete the mysql namespace
kubectl delete namespace mysql
```

## Congrats!
