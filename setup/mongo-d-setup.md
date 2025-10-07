Got it! The safest way to have “one replica in case the master goes down” is a MongoDB **replica set**. Quick terminology note: MongoDB calls them **primary** and **secondary** (not master/slave). If you run only **2 data nodes** (primary + one replica), you still need a **third voting member** (usually an **arbiter**) so elections can succeed automatically when one node fails. Below are two solid setups:

---

# Option A (recommended): 2 data nodes + 1 arbiter (saves storage)

## Quick install (Helm)

```bash
# Namespace
kubectl create ns mongo

# Add chart repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install a replicaset with 2 data nodes + 1 arbiter
helm install mongo bitnami/mongodb -n mongo \
  --set architecture=replicaset \
  --set replicaCount=2 \
  --set arbiter.enabled=true \
  --set auth.rootUser=admin \
  --set auth.rootPassword='S3curePassw0rd' \
  --set auth.replicaSetKey='replica-set-shared-key' \
  --set persistence.size=20Gi
```

What you get:

* Pods: `mongo-mongodb-0`, `mongo-mongodb-1` (data-bearing) and `mongo-mongodb-arbiter-0` (no data).
* Headless service for stable DNS.
* Automatic replica-set init.

Connect (from a pod in the cluster):

```bash
kubectl -n mongo exec -it deploy/mongo-mongodb -- \
  mongosh "mongodb://admin:S3curePassw0rd@mongo-mongodb-0.mongo-mongodb-headless.mongo.svc.cluster.local,mongo-mongodb-1.mongo-mongodb-headless.mongo.svc.cluster.local/?replicaSet=rs0&authSource=admin"
```

## Why this works

* **Elections need a majority.** With 2 data nodes + 1 arbiter, you have **3 votes**. If the primary dies, the secondary + arbiter (2/3) elect a new primary automatically.
* **Arbiter saves storage/CPU** (no data), but it does not hold data—just votes.

---

# Option B: 3 data nodes (no arbiter, more resilient, uses more storage)

## Minimal DIY manifests (StatefulSet)

This is a compact example that works on most clusters. Adjust storage class, resources, and passwords as needed.

### 1) Secrets (root user + replSet key)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongo-auth
  namespace: mongo
type: Opaque
stringData:
  MONGODB_ROOT_USER: admin
  MONGODB_ROOT_PASSWORD: S3curePassw0rd
  MONGODB_REPLICA_SET_KEY: replica-set-shared-key
```

### 2) Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb
  namespace: mongo
spec:
  clusterIP: None
  selector:
    app: mongodb
  ports:
  - name: mongo
    port: 27017
```

### 3) StatefulSet (3 replicas)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: mongo
spec:
  serviceName: mongodb
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: mongodb
              topologyKey: kubernetes.io/hostname
      containers:
      - name: mongod
        image: mongo:6.0
        args: ["--replSet", "rs0", "--bind_ip_all"]
        ports:
        - containerPort: 27017
          name: mongo
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom: { secretKeyRef: { name: mongo-auth, key: MONGODB_ROOT_USER } }
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom: { secretKeyRef: { name: mongo-auth, key: MONGODB_ROOT_PASSWORD } }
        - name: MONGODB_REPLICA_SET_KEY
          valueFrom: { secretKeyRef: { name: mongo-auth, key: MONGODB_REPLICA_SET_KEY } }
        volumeMounts:
        - name: data
          mountPath: /data/db
        - name: keyfile
          mountPath: /opt/mongo
          readOnly: true
      initContainers:
      - name: init-keyfile
        image: mongo:6.0
        command: ["bash","-lc","cat <<EOF >/opt/mongo/keyfile && chmod 400 /opt/mongo/keyfile && chown 999:999 /opt/mongo/keyfile
$(MONGODB_REPLICA_SET_KEY)
EOF"]
        env:
        - name: MONGODB_REPLICA_SET_KEY
          valueFrom: { secretKeyRef: { name: mongo-auth, key: MONGODB_REPLICA_SET_KEY } }
        volumeMounts:
        - name: keyfile
          mountPath: /opt/mongo
      securityContext:
        fsGroup: 999
      volumes:
      - name: keyfile
        emptyDir: {}
      terminationGracePeriodSeconds: 30
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 20Gi
```

Apply:

```bash
kubectl create ns mongo
kubectl apply -f mongo-secrets.yaml
kubectl apply -f mongo-svc.yaml
kubectl apply -f mongo-statefulset.yaml
```

### 4) Initialize the replica set once pods are Ready

```bash
# Get a shell in the first pod
kubectl -n mongo exec -it mongodb-0 -- mongosh -u admin -p 'S3curePassw0rd' --authenticationDatabase admin

# In mongosh:
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongodb-0.mongodb.mongo.svc.cluster.local:27017" },
    { _id: 1, host: "mongodb-1.mongodb.mongo.svc.cluster.local:27017" },
    { _id: 2, host: "mongodb-2.mongodb.mongo.svc.cluster.local:27017" }
  ]
})
rs.status()
```

---

## How the replica/failover works (both options)

* **Replication:** The **primary** accepts writes; secondaries replicate via the oplog (a rolling log of operations). Replication is near-real-time.
* **Elections & quorum:** Members vote to choose a primary. You need **majority votes** to elect/keep a primary.

  * With **2 data + 1 arbiter** (3 votes): if the primary dies, the secondary + arbiter (2/3) elect a new primary automatically.
  * With **3 data nodes** (3 votes): if one node dies, the other two elect a new primary.
  * With **only 2 nodes and no arbiter** (2 votes): if one dies, the survivor **cannot** form a majority alone → **no automatic primary → writes stop** (reads from the remaining secondary are still possible with `readPreference=secondary`).
* **Write safety:** Use `writeConcern: "majority"` so writes are acknowledged only after being replicated to a majority, ensuring durability across failover.
* **Read routing:** Apps typically connect using a replica set URI (multiple hosts + `replicaSet=rs0`). Drivers automatically discover the current primary and reroute writes after failover. You can set `readPreference` to control reads (e.g., `primary`, `primaryPreferred`, `secondary`).

---

## Production hardening checklist

* **PodDisruptionBudget** (e.g., `minAvailable: 2`) to avoid voluntary disruptions taking down majority.
* **Pod anti-affinity** (already shown) so members land on different nodes/zones.
* **Storage class & IOPS** sized for your workload; enable volume expansion.
* **Resource requests/limits** on pods.
* **TLS + SCRAM** auth; rotate the replica-set key.
* **Backup** (e.g., Velero + volume snapshots, or `mongodump`/Oplog backup).
* **Liveness/Readiness** probes tuned for elections and startup timeouts.

---

### TL;DR for your ask

If by “one replica” you mean **one extra copy** of the data, use **Option A** (2 data nodes + 1 arbiter). That gives you automatic failover when the primary goes down without paying for a third data disk. If you prefer no arbiter, run **3 data nodes**.

If you’d like, tell me your preferred path (Helm vs. manifests) and your storage class name, and I’ll tailor the exact YAML/values for your cluster.
