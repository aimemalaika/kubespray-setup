Yep — we can ditch Bitnami and use a different, well-maintained option. The two solid choices are:

## Option 1 (recommended): **Percona Server for MongoDB (PSMDB) Operator** via Helm

This is a first-class open-source operator with Helm charts from Percona. It deploys a robust replica set (no arbiters needed; use 3 data members). ([Artifact Hub][1])

### Install (namespace = `tugane-sit`)

```bash
# Repo
helm repo add percona https://percona.github.io/percona-helm-charts
helm repo update

# 1) Install the operator
helm install psmdb-operator percona/psmdb-operator -n tugane-sit --create-namespace

# 2) Install a 3-member replica set managed by the operator
helm install psmdb percona/psmdb-db -n tugane-sit \
  --set replsets.rs0.size=3 \
  --set image.tag=7.0.15-20  \
  --set replsets.rs0.storage.size=20Gi
```

> Check status:

```bash
kubectl -n tugane-sit get psmdb
kubectl -n tugane-sit get pods -l app.kubernetes.io/instance=psmdb
```

**Connect**: the operator exposes a headless service per replset; you’ll get a standard MongoDB replica-set URI. (Docs cover CR options, exposure modes, etc.) ([docs.percona.com][2])

**Why Percona?** Mature operator, active docs, Helm-native install, and no Bitnami catalog issues. ([docs.percona.com][3])

### Using your private registry + amd64

If you mirror images to `registry.app.brd-hq-cluster.brd.rw` (amd64 only), override the image repo/tag in Helm:

```bash
helm upgrade psmdb percona/psmdb-db -n tugane-sit --reuse-values \
  --set image.repository=registry.app.brd-hq-cluster.brd.rw/mirrors/percona/percona-server-mongodb \
  --set image.tag=7.0.15-20
```

(When mirroring from macOS, use `--platform=linux/amd64` or `skopeo --override-arch amd64` so your cluster pulls amd64 layers.)

---

## Option 2: **MongoDB Community Kubernetes Operator** (official)

MongoDB’s own operator (community edition). You install the operator (Helm or kubectl), then create a `MongoDBCommunity` CR for a 3-member replica set. Note: their operator doesn’t use arbiters; plan for 3 data members. ([MongoDB][4])

### Quick start (operator + CR)

```bash
# (a) Install operator CRDs & controller (Helm chart exists on Artifact Hub)
helm repo add mongodb https://mongodb.github.io/helm-charts
helm repo update
helm install community-operator mongodb/community-operator -n tugane-sit --create-namespace
# If CRDs already installed: add --set community-operator-crds.enabled=false
```

Create a minimal replica-set CR:

```yaml
# mongo-community.yaml
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: my-rs
  namespace: tugane-sit
spec:
  members: 3
  type: ReplicaSet
  version: "7.0.15"
  security:
    authentication:
      modes: ["SCRAM"]
  statefulSet:
    spec:
      volumeClaimTemplates:
      - metadata:
          name: data-volume
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 20Gi
```

Apply it:

```bash
kubectl apply -f mongo-community.yaml
kubectl -n tugane-sit get mongodbcommunity my-rs
```

(See the operator docs for install/config details.) ([MongoDB][4])

---

## Which should we use?

* **Percona Operator (PSMDB)**: best balance of simplicity + production features, great docs and Helm UX. My pick for your case. ([docs.percona.com][3])
* **MongoDB Community Operator**: official, lean; good if you want “vanilla” community MongoDB and are comfortable with operator CRs. ([GitHub][5])

If you tell me your storage class name and whether you want to **pull from your private registry** only, I’ll drop in exact Helm values (including amd64-only images and resource limits) for the option you choose.

[1]: https://artifacthub.io/packages/helm/percona/psmdb-operator?utm_source=chatgpt.com "psmdb-operator 1.20.1 · percona/percona - artifacthub.io"
[2]: https://docs.percona.com/percona-operator-for-mongodb/operator.html?utm_source=chatgpt.com "Custom Resource options - Percona Operator for MongoDB"
[3]: https://docs.percona.com/percona-operator-for-mongodb/helm.html?utm_source=chatgpt.com "With Helm - Percona Operator for MongoDB"
[4]: https://www.mongodb.com/docs/kubernetes/current/tutorial/install-k8s-operator/?utm_source=chatgpt.com "Install the MongoDB Controllers for Kubernetes Operator - MongoDB ..."
[5]: https://github.com/mongodb/mongodb-kubernetes-operator?utm_source=chatgpt.com "MongoDB Community Kubernetes Operator - GitHub"
