# MySQL Replication Lab on Kubernetes (AWS EKS/EC2)
**Author: Nishant Minj**

This lab demonstrates how to set up a **Master-Slave MySQL Cluster** using a Kubernetes **StatefulSet**, unique server identities, and AWS EBS persistence.

## 1. Architecture Visualization

```text
       [ Headless Service: mysql-headless ]
                     |
      +--------------+--------------+
      |              |              |
[ mysql-ss-0 ] [ mysql-ss-1 ] [ mysql-ss-2 ]
( MASTER ID:100)( SLAVE ID:101 )( SLAVE ID:102 )
      |              |              |
      | <----------  | <----------  |  (Replication)
      |              |              |
 [ EBS Vol 0 ]  [ EBS Vol 1 ]  [ EBS Vol 2 ]
```

---

## 2. Prerequisites: AWS EBS CSI Driver & StorageClass
Before the pods can start, the nodes must be able to provision EBS volumes.

### Step 1: Install AWS EBS CSI Driver
```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.25"
```

### Step 2: Create the StorageClass
Save this as `sc.yml` and apply it:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc-001
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```
```bash
kubectl apply -f sc.yml
```

### Step 3: Verify
```bash
kubectl get sc ebs-sc-001
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

---

## 3. Step-by-Step Setup

### Step A: Create Namespace and Secret
```bash
# Create namespace if it doesn't exist
kubectl create namespace nishant-namespace --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic mysql-secret -n nishant-namespace \
  --from-literal=root-password='Login%12345'
```

### Step B: Apply Manifests
Create a file named `test.yml` with the following content. This manifest automates the configuration of Master and Slave nodes.

#### The Full Manifest (`test.yml`)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: nishant-namespace
data:
  master.cnf: |
    [mysqld]
    log-bin
    server-id=1
    binlog-ignore-db=mysql
    bind-address=0.0.0.0
    skip-name-resolve
  slave.cnf: |
    [mysqld]
    server-id=2
    relay-log=mysql-relay-bin
    binlog-ignore-db=mysql
    bind-address=0.0.0.0
    skip-name-resolve
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-ss
  namespace: nishant-namespace
spec:
  serviceName: mysql-headless
  replicas: 2
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:8.0.36
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Use $HOSTNAME to get pod index (e.g., mysql-ss-0)
          [[ $HOSTNAME =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # Generate unique server-id (100, 101, 102...)
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # Assign master.cnf to pod 0, slave.cnf to others
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/configmap/master.cnf /mnt/conf.d/
          else
            cp /mnt/configmap/slave.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/configmap
      containers:
      - name: mysql
        image: mysql:8.0.36
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql-config
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: ebs-sc-001
      resources:
        requests:
          storage: 5Gi
```

#### What this Manifest is Doing:
1.  **ConfigMap**: Stores two templates: `master.cnf` (enables binary logging) and `slave.cnf` (enables relay logging).
2.  **StatefulSet**: Ensures pods have a stable hostname (`mysql-ss-0`, `mysql-ss-1`) and their own dedicated EBS storage.
3.  **InitContainer (`init-mysql`)**:
    -   Runs before the main MySQL container.
    -   Extracts the pod index from the hostname.
    -   **Server-ID**: Generates a unique `server-id` (100 + index) so MySQL doesn't conflict.
    -   **Dynamic Config**: Copies the correct configuration (Master vs Slave) into an `emptyDir` volume which is then mounted by the main container.
4.  **VolumeClaimTemplates**: Automatically creates an AWS EBS volume for each pod to ensure data survives pod restarts.

```bash
kubectl apply -f test.yml
```

### Step C: Identify Master Log Position
```bash
export ROOT_PASSWORD=$(kubectl get secret mysql-secret -n nishant-namespace -o jsonpath='{.data.root-password}' | base64 -d)

# Run on mysql-ss-0
kubectl exec -it mysql-ss-0 -n nishant-namespace -- mysql -u root -p"$ROOT_PASSWORD"
run the belwo query:
===================
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'Login%12345';
FLUSH PRIVILEGES;

kubectl exec mysql-ss-0 -n nishant-namespace -- mysql -u root -p"$ROOT_PASSWORD" -e "SHOW MASTER STATUS;"

```
*Note the **File** (e.g. `mysql-ss-0-bin.000003`) and **Position**.*

### Step D: Configure Slaves (Manual Step)
You must manually link each slave to the master. Repeat this for `mysql-ss-1` and `mysql-ss-2`:

```bash
# Log into Slave
kubectl exec -it mysql-ss-1 -n nishant-namespace -- mysql -u root -p"$ROOT_PASSWORD"

# Inside MySQL Shell:
STOP SLAVE;
CHANGE MASTER TO
  MASTER_HOST='mysql-ss-0.mysql-headless.nishant-namespace.svc.cluster.local',
  MASTER_USER='root',
  MASTER_PASSWORD='Login%12345',
  MASTER_LOG_FILE='mysql-ss-0-bin.000003', -- USE OUTPUT FROM STEP C
  MASTER_LOG_POS=4;                         -- Use 4 to catch all data
START SLAVE;
SHOW SLAVE STATUS\G


# Inside MySQL Shell:
STOP SLAVE;
START SLAVE;
SHOW SLAVE STATUS\G

```

---

## 4. Validation Steps

### Write on Master
```bash
kubectl exec mysql-ss-0 -n nishant-namespace -- mysql -u root -p"$ROOT_PASSWORD" \
  -e "CREATE DATABASE lab_test; USE lab_test; CREATE TABLE checks (id INT, status VARCHAR(20)); INSERT INTO checks VALUES (1, 'Success');"
```

### Read on Slave
```bash
kubectl exec mysql-ss-1 -n nishant-namespace -- mysql -u root -p"$ROOT_PASSWORD" \
  -e "SELECT * FROM lab_test.checks;"
```

---

## 5. Troubleshooting & FAQ

### Why do we do manual steps?
1.  **Replication Identity**: MySQL requires the Slave to know the *exact* binary log file and position of the Master at the moment it starts. Since the Master is already running and generating logs, this "handshake" is usually done manually or by an automation script (Sidecar).
2.  **Server IDs**: We use an `initContainer` to automate the `server-id` so you don't have to manage different ConfigMaps for each pod.

### Common Errors:
- **Multi-Attach Error**: Happens if you delete a pod and K8s tries to start it on a different node before AWS releases the volume. **Fix**: Force delete the pod or wait 2-5 minutes.
- **Unknown Database**: Happens if the Slave starts replication *after* the database was created. **Fix**: Set `MASTER_LOG_POS=4` to replay the whole log.
- **Init:Error**: Usually a syntax error in the `initContainer` script (e.g. `hostname` command missing). **Fix**: Use `$HOSTNAME` env variable.

---

## 6. Cleanup and Recreate

### Full Delete
```bash
kubectl delete statefulset mysql-ss -n nishant-namespace
kubectl delete pvc --all -n nishant-namespace
kubectl delete configmap mysql-config -n nishant-namespace
```

### Recreate
```bash
kubectl apply -f test.yml
# Then repeat Step C and D to re-link replication.
```
