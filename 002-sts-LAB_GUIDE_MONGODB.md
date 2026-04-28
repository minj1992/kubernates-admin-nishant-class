# MongoDB ReplicaSet Lab on Kubernetes
**Author: Nishant Minj**

This lab demonstrates how to set up a 3-node **MongoDB ReplicaSet** using a Kubernetes **StatefulSet** and AWS EBS persistence.

## 1. Architecture & Traffic Flow

```text
      [ USER / APPLICATION ]
               |
               | (1) WRITE Request (Always to Primary)
               | (2) READ Request  (Can be to Primary or Secondary)
               v
    [ Headless Service: mongodb-headless ]
               |
      +--------+--------------+-----------------+
      | (PRIMARY)             | (SECONDARY)     | (SECONDARY)
      v                       v                 v
[ mongodb-ss-0 ] <------ [ mongodb-ss-1 ] [ mongodb-ss-2 ]
      |        (Replication)  ^                 ^
      |           Flow        |                 |
      v                       +-----------------+
 [ EBS Vol 0 ]           [ EBS Vol 1 ]     [ EBS Vol 2 ]
(Persistent Data)       (Persistent Data) (Persistent Data)
```

### How Traffic Flows:
1.  **Write Requests**: All data modifications (Insert, Update, Delete) **MUST** go to the **Primary** node. If an application tries to write to a Secondary, MongoDB will return an error.
2.  **Read Requests**: By default, reads go to the Primary. However, applications can be configured with a **Read Preference** (e.g., `secondaryPreferred`) to distribute read traffic across the Secondary nodes, reducing load on the Primary.
3.  **Automatic Failover**: If the Primary (`mongodb-ss-0`) crashes, the Secondaries hold an election and one of them automatically becomes the new Primary.

## 2. Prerequisites: Storage Configuration
Before applying the MongoDB manifest, ensure the AWS EBS CSI driver and StorageClass are configured.

### Step 1: Create the StorageClass
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

---

## 3. Full Manifest (`mongodb.yml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-headless
  namespace: nishant-namespace
spec:
  clusterIP: None
  selector:
    app: mongodb
  ports:
    - name: mongodb
      port: 27017
      targetPort: 27017
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-ss
  namespace: nishant-namespace
spec:
  serviceName: mongodb-headless
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: mongodb
          image: mongo:7.0
          command:
            - mongod
            - "--replSet"
            - "rs0"
            - "--bind_ip_all"
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongodb-data
              mountPath: /data/db
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
  volumeClaimTemplates:
    - metadata:
        name: mongodb-data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: ebs-sc-001
        resources:
          requests:
            storage: 5Gi
```

---

## 3. Step-by-Step Setup

### Step A: Apply Manifest
```bash
# Create namespace if it doesn't exist
kubectl create namespace nishant-namespace --dry-run=client -o yaml | kubectl apply -f -

kubectl apply -f mongodb.yml
```

### Step B: Storage Validation (SC, PVC, PV)
Verify that the storage has been provisioned correctly:
```bash
# Check StorageClass
kubectl get sc ebs-sc-001

# Check PVC status (Should be 'Bound')
kubectl get pvc -n nishant-namespace -l app=mongodb

# Check PV status
kubectl get pv | grep mongodb
```

### Step C: Initiate the ReplicaSet (Manual Step)
MongoDB nodes need to be "introduced" to each other. Run this on the first pod:

```bash
kubectl exec -it mongodb-ss-0 -n nishant-namespace -- mongosh --eval '
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongodb-ss-0.mongodb-headless.nishant-namespace.svc.cluster.local:27017" },
    { _id: 1, host: "mongodb-ss-1.mongodb-headless.nishant-namespace.svc.cluster.local:27017" },
    { _id: 2, host: "mongodb-ss-2.mongodb-headless.nishant-namespace.svc.cluster.local:27017" }
  ]
})'
```

### Step D: Verify ReplicaSet Status
```bash
kubectl exec -it mongodb-ss-0 -n nishant-namespace -- mongosh --eval "rs.status()"
```
*Look for `"ok": 1` and `"myState": 1` (PRIMARY) vs `"myState": 2` (SECONDARY).*

---

## 4. Replication Validation

### Write on Primary Node
```bash
kubectl exec -it mongodb-ss-0 -n nishant-namespace -- mongosh --eval '
use lab_db;
db.test_collection.insertOne({ name: "Venky", tool: "Zencoder", date: new Date() });
'
```

### Read on Secondary Node
Log into a secondary node (e.g., `mongodb-ss-1`):
```bash
# Secondary nodes require "setSecondaryOk()" to allow reads
kubectl exec -it mongodb-ss-1 -n nishant-namespace -- mongosh --eval '
db.getMongo().setSecondaryOk();
use lab_db;
db.test_collection.find().pretty();
'
```

---

## 5. Troubleshooting & FAQ
- **Connection Refused**: Ensure the headless service name matches the `host` used in `rs.initiate`.
- **Pending Pods**: Check if `ebs-sc-001` exists and your AWS EBS CSI driver is running.
- **Election Delay**: It takes about 10-30 seconds for nodes to elect a PRIMARY after initiation.

---

## 6. Cleanup & Reset (Repeatable Lab)
To delete everything and start fresh:

```bash
# 1. Delete the StatefulSet and Service
kubectl delete -f mongodb.yml

# 2. Delete the Persistent Volume Claims (This deletes the data!)
kubectl delete pvc -l app=mongodb -n nishant-namespace

# 3. Verify cleanup
kubectl get pods,pvc -n nishant-namespace

# 4. Re-run Step A to restart the lab
```
