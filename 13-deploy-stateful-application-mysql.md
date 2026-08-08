# 13 - Deploy a Stateful Application (MySQL StatefulSet)

> Course: Cloud Native Champions: CKAD Bootcamp  
> Module: Deploy a Stateful Application in a Kubernetes Cluster  
> Lab: Inspecting the Cluster / Deploying MySQL / Working with It / Kubernetes Dashboard  
> Topics: statefulset, headless-service, configmap, storageclass, pvc, pv, ebs, init-containers, sidecar, mysql, replication, probes, drain, cordon, scale, loadbalancer, dashboard, rbac, clusterrolebinding, port-forward, taints, kube-system

## Goal

Deploy a replicated MySQL database as a StatefulSet with a headless Service, a StorageClass, and per-Pod PVCs. Then test failover, scaling, and external access, and manage the cluster from the Kubernetes Dashboard.

## Notes

### Cluster components (kube-system)

- **aws-cloud-controller:** links the cluster to AWS service APIs
- **calico:** container network between nodes; also supports network policy
- **coredns:** DNS for the cluster
- **etcd:** primary data store of all cluster state
- **kube-apiserver:** REST API for managing the cluster
- **kube-controller-manager:** runs the controllers that reconcile cluster state
- **kube-proxy:** network proxy on each node
- **kube-scheduler:** assigns Pods to nodes
- **metrics-server:** not essential; gives metrics to the dashboard
- **ebs-csi:** not essential; manages Amazon EBS volumes for persistent volumes

### Namespaces

- `default`: used when a Pod declares no namespace
- `kube-system`: Pods essential to cluster operation
- `kubernetes-dashboard`: dashboard resources

### Taints

The control plane node has the taint `node-role.kubernetes.io/control-plane:NoSchedule`. Taints repel Pods unless the Pod declares a matching toleration. With `NoSchedule`, no Pod lands on the control plane node without that toleration.

### Stateful concepts

- **ConfigMap:** decouples configuration from the image. Data is key-value pairs.
- **Headless Service:** `clusterIP: None`. No single service IP and no load balancing. DNS returns records pointing straight at the Pods. StatefulSets need one to give Pods network identity.
- **StatefulSet:** like a Deployment, but Pods are not interchangeable. Each Pod keeps a stable identity across rescheduling, and Pods are created in order. That ordering makes `mysql-0` the primary.
- **PV and PVC:** PVs are cluster storage with a lifetime independent of any Pod. Pods claim them through PVCs.
- **MySQL replication here:** one primary, asynchronous replicas. All writes go to the primary. Replicas catch up later, so they can lag slightly.

### Addressing the Pods

- Writes: `mysql-0.mysql` (the primary, first Pod in the StatefulSet)
- Reads: `mysql-read` Service (ClusterIP, load balances across all `app: mysql` Pods)
- Any Pod directly: `<pod-name>.mysql`

### When a StatefulSet makes sense

Consider Kubernetes for stateful apps if any of these apply:

- You embrace microservices
- You often create new service footprints that include stateful apps
- Your current state store cannot scale to predicted demand
- Your stateful apps meet performance needs on the same hardware as stateless ones
- You value flexible resource reallocation and automation over highly predictable performance

Helm charts already package MySQL, MongoDB, PostgreSQL, WordPress, and more.

---

## Step 1 - Inspecting the Kubernetes cluster

1. List the nodes:

```bash
kubectl get nodes
```

The control plane node's private IP appears in one of the internal DNS names. The other nodes belong to the Auto Scaling group.

2. Describe the control plane node:

```bash
kubectl describe nodes -l node-role.kubernetes.io/control-plane | more
```

The label selector picks the single control plane node. Look at the Taints section.

3. See the namespaces (type the partial command and press Tab twice for completions):

```bash
kubectl get namespace
```

4. List Pods in `kube-system`:

```bash
kubectl get pods --namespace=kube-system
```

All cluster components run here. See the component list in Notes.

### Summary (Step 1)

All cluster components are up and running.

---

## Step 2 - Deploying the stateful application

1. Declare a ConfigMap so primary and replica Pods get different MySQL config:

```bash
cat <<EOF > mysql-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  labels:
    app: mysql
data:
  master.cnf: |
   # Apply this config only on the primary.
   [mysqld]
   log-bin
  slave.cnf: |
    # Apply this config only on replicas.
    [mysqld]
    super-read-only
EOF
```

`master.cnf` turns on replication logs. `slave.cnf` enforces read-only.

2. Create the ConfigMap:

```bash
kubectl create -f mysql-configmap.yaml
```

3. Declare the two Services:

```bash
cat <<EOF > mysql-services.yaml
# Headless service for stable DNS entries of StatefulSet members.
apiVersion: v1
kind: Service
metadata:
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
# For writes, you must instead connect to the primary: mysql-0.mysql.
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
EOF
```

The headless Service (`clusterIP: None`) gives Pod DNS names such as `mysql-0.mysql`. The `mysql-read` Service uses the default ClusterIP type and load balances reads.

4. Create the Services:

```bash
kubectl create -f mysql-services.yaml
```

5. Declare a StorageClass for dynamic gp2 EBS volumes:

```bash
cat <<EOF > mysql-storageclass.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: general
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
EOF
```

6. Create the StorageClass:

```bash
kubectl create -f mysql-storageclass.yaml
```

7. Declare the StatefulSet:

```bash
cat <<'EOF' > mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:5.7.35
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Generate mysql server-id from pod ordinal index.
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # Add an offset to avoid reserved server-id=0 value.
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # Copy appropriate conf.d files from config-map to emptyDir.
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/slave.cnf /mnt/conf.d/
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
          # Skip the clone on primary (ordinal index 0).
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
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
            cpu: 100m
            memory: 200Mi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          timeoutSeconds: 5
        readinessProbe:
          exec:
            # Check we can execute queries over TCP (skip-networking is off).
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
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
            # because we're cloning from an existing replica.
            mv xtrabackup_slave_info change_master_to.sql.in
            # Ignore xtrabackup_binlog_info in this case (it's useless).
            rm -f xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # We're cloning directly from primary. Parse binlog position.
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
            memory: 50Mi
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 2Gi
      storageClassName: general
EOF
```

Do not focus on the MySQL-specific bash. The parts that matter:

- **initContainers** run to completion before the app containers start.
  - `init-mysql`: sets a unique server ID starting at 100, and copies the right config file from the ConfigMap onto the `conf` volume.
  - `clone-mysql`: for Pods after the primary, clones the database files from the previous Pod with `xtrabackup` onto the `data` volume.
- **spec.containers** holds two containers.
  - `mysql`: runs the MySQL daemon, mounts `conf` and `data`.
  - `xtrabackup`: sidecar that serves data for cloning and starts replication on replicas.
- **spec.volumes:** `conf` and `config-map` live on the node's local disk. They are easy to regenerate, so they need no PV.
- **volumeClaimTemplates:** creates one PVC per Pod. `ReadWriteOnce` lets one node mount it read/write at a time. `storageClassName: general` points at the gp2 StorageClass.

8. Create the StatefulSet and watch the Pods:

```bash
kubectl create -f mysql-statefulset.yaml
kubectl get pods -l app=mysql --watch
```

All three replicas take a few minutes to initialize.

9. Open the lab's AWS Management Console from the lab UI.

10. Sign in with the IAM user created for your lab session:

- Account ID or alias: keep the pre-populated value
- IAM user name: `student`
- Password: shown in the lab UI for that session

11. Go to Elastic Block Store > Volumes in the EC2 Console. The 2GiB PVs appear as each Pod is created. The Tags show the PV and its PVC.

12. Back in the SSH shell, press Ctrl+C to stop the watch once `mysql-2` shows 2/2 containers running.

13. Describe the PVs and PVCs:

```bash
kubectl describe pv
kubectl describe pvc
```

PV output shows AWS VolumeIDs, FSType, and the bound Claim. PVC output shows whether it is Bound.

14. Confirm replicas match the desired count:

```bash
kubectl get statefulset
```

### Summary (Step 2)

You created a ConfigMap, a headless Service plus a read Service, a StorageClass, and a StatefulSet with two init containers, two containers, and one PVC template. Pods initialize in order and each gets its own EBS-backed PV.

---

## Step 3 - Working with the stateful application

1. Write to the primary from a temporary client Pod:

```bash
kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  /usr/bin/mysql -h mysql-0.mysql -e "CREATE DATABASE mydb; CREATE TABLE mydb.notes (note VARCHAR(250)); INSERT INTO mydb.notes VALUES ('k8s Cloud Academy Lab');"
```

2. Read through the `mysql-read` Service:

```bash
kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  /usr/bin/mysql -h mysql-read -e "SELECT * FROM mydb.notes"
```

3. Confirm reads spread across Pods by printing the server ID in a loop:

```bash
kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
  bash -ic "while sleep 1; do /usr/bin/mysql -h mysql-read -e 'SELECT @@server_id'; done"
```

Every server answers eventually. The primary has ID 100.

4. Press Ctrl+C to stop the loop.

5. Show which node runs each Pod:

```bash
kubectl get pod -o wide
```

Kubernetes spreads the Pods across worker nodes as evenly as it can.

6. Simulate maintenance by draining the node that runs `mysql-2`:

```bash
node=$(kubectl get pods --field-selector metadata.name=mysql-2 -o=jsonpath='{.items[0].spec.nodeName}')
kubectl drain $node --force --delete-emptydir-data --ignore-daemonsets
```

The node name looks like `ip-10-0-#-#.us-west-2.compute-internal`. `drain` stops new scheduling on the node and evicts the Pods already there.

7. Watch `mysql-2` move to another node:

```bash
kubectl get pod -o wide --watch
```

8. Allow scheduling on the node again:

```bash
kubectl uncordon $node
```

9. Delete `mysql-2` to simulate a node failure and watch it come back:

```bash
kubectl delete pod mysql-2
kubectl get pod mysql-2 -o wide --watch
```

10. Press Ctrl+C to stop watching.

11. Scale to 5 replicas:

```bash
kubectl scale --replicas=5 statefulset mysql
```

12. Watch the new Pods:

```bash
kubectl get pods -l app=mysql --watch
```

13. Press Ctrl+C to stop watching.

14. Check the new server IDs:

```bash
kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
  bash -ic "while sleep 1; do /usr/bin/mysql -h mysql-read -e 'SELECT @@server_id'; done"
```

15. Confirm the data reached `mysql-4`:

```bash
kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  /usr/bin/mysql -h mysql-4.mysql -e "SELECT * FROM mydb.notes"
```

16. Show the internal IP of `mysql-read`:

```bash
kubectl get services mysql-read
```

It is ClusterIP, so it is reachable only inside the cluster.

17. Append a LoadBalancer type to the `mysql-read` declaration:

```bash
echo "  type: LoadBalancer" >> mysql-services.yaml
```

18. Apply the change:

```bash
kubectl apply -f mysql-services.yaml
```

`apply` updates existing resources and creates missing ones. You created the Services with `create`, so a warning appears. Ignore it here.

19. Check the Service again:

```bash
kubectl get services mysql-read
```

Kubernetes starts provisioning an elastic load balancer (ELB).

20. After about a minute, find the external DNS name:

```bash
kubectl describe services mysql-read | grep "LoadBalancer Ingress"
```

21. Send reads through the load balancer:

```bash
load_balancer=$(kubectl get services mysql-read -o=jsonpath='{.status.loadBalancer.ingress[0].hostname}')
kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
  bash -ic "while sleep 1; do /usr/bin/mysql -h $load_balancer -e 'SELECT @@server_id'; done"
```

The ELB DNS name looks like `a830e12d78dcd11e79aba028416f4825-905974806.us-west-2.elb.amazonaws.com`.

Note: nodes take about a minute to join the load balancer. You see unknown MySQL host messages until then. Let the loop run until server IDs appear.

22. Press Ctrl+C to stop the temporary container.

### Summary (Step 3)

You addressed Pods directly, read through the `mysql-read` Service, drained a node, recovered from a simulated node failure, scaled the StatefulSet, and exposed reads through an ELB.

---

## Step 4 - Monitoring with the Kubernetes Dashboard

1. Declare a ServiceAccount and bind `cluster-admin` to it:

```bash
cat << EOF > dashboard-admin.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: dashboard-admin
  namespace: kubernetes-dashboard
EOF
```

`roleRef` points at the `cluster-admin` ClusterRole. `subjects` binds it to the dashboard ServiceAccount.

2. Create the binding:

```bash
kubectl create -f dashboard-admin.yaml
```

3. Create a sign-in token:

```bash
kubectl -n kubernetes-dashboard create token dashboard-admin
```

4. Port-forward the dashboard from the bastion:

```bash
kubectl port-forward -n kubernetes-dashboard --address 0.0.0.0 service/kubernetes-dashboard 8001:443
```

`--address 0.0.0.0` accepts connections from anywhere. Acceptable only for a demo.

5. Open the dashboard in a browser:

```text
https://<bastion-public-ip>:8001
```

The lab shows the actual IP, for example `54.202.126.109`. The browser flags the TLS certificate because the cluster uses a self-signed one.

6. Continue past the warning. In recent Chrome versions, type `thisisunsafe` on the warning page.

7. Paste the token from instruction 3 into the Enter token field and click Sign in.

8. Go to Workloads > Stateful Sets > mysql. The CPU and Memory graphs work because metrics-server runs in the cluster.

9. Click Scale in the top-right of the mysql view.

10. Set Desired replicas to `3` and click Scale.

11. Scroll to Pods. `mysql-4` is removed first, then `mysql-3`. StatefulSets remove Pods in reverse creation order.

12. Go to Cluster > Persistent Volumes. Five PVs remain. Scaling down does not delete PVs. You remove them yourself.

13. In the row for Claim `default/data-mysql-4`, open the three-dot menu and click Delete.

14. Confirm Delete. The PV is not deleted because a PVC still binds it.

15. Go to Config and Storage > Persistent Volume Claims and open `data-mysql-4`.

16. Click Delete in the upper-right corner.

17. Confirm Delete.

18. Return to the PV view. Only 4 PVs remain.

### Summary (Step 4)

You bound `cluster-admin` to the dashboard ServiceAccount, reached the dashboard through `kubectl port-forward`, and used it to scale the StatefulSet and delete a PVC and its PV.

---

## Validation

- **Service Exposing StatefulSet Via Load Balancer:** the StatefulSet is reachable through a LoadBalancer Service (Step 3).

## Summary

A stateful app on Kubernetes needs four pieces working together: a ConfigMap for per-role config, a headless Service for stable Pod DNS, a StorageClass plus `volumeClaimTemplates` for per-Pod storage, and a StatefulSet for ordered, identity-keeping Pods. Pods keep their name and volume across rescheduling, and PVs survive scale-down until you delete them.
