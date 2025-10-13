Got it, barafinda — that’s a **Helm values file for Percona’s `psmdb-db` chart**. Use Helm (not `kubectl apply`) and you’ll be fine.

### 1) Two small fixes in your values

* `sharding.mongos.expose.exposeType` → **`sharding.mongos.expose.type`** (the chart expects `type`). ([Percona Documentation][1])
* Put the pull policy **inside** the `image` block (the chart uses `image.pullPolicy`):

```diff
 image:
   repository: registry.kube-system.svc:5000/percona-server-mongodb
   tag: "7.0.24-13"
- imagePullPolicy: IfNotPresent
+  pullPolicy: IfNotPresent
```

### 2) Install the Operator (once per cluster/namespace)

```bash
helm repo add percona https://percona.github.io/percona-helm-charts
helm repo update

# choose a namespace; example: percona
kubectl create ns percona

helm install psmdb-operator percona/psmdb-operator -n percona
```

(That chart deploys the **Percona Operator for MongoDB** and its CRDs.) ([Artifact Hub][2])

### 3) Install your sharded DB using this values file

```bash
# after the two tweaks above
helm install mongo-rs percona/psmdb-db -n percona -f psmdb-values.yaml
# later updates:
helm upgrade mongo-rs percona/psmdb-db -n percona -f psmdb-values.yaml
```

(Official docs show `helm install <release> percona/psmdb-db ...`.) ([Percona Documentation][3])

### 4) Verify it’s healthy

```bash
kubectl -n percona get psmdb
kubectl -n percona get pods -l app.kubernetes.io/instance=mongo-rs
kubectl -n percona get svc -l app.kubernetes.io/component=mongos
```

### 5) If you *must* use `kubectl apply`

Render the chart (including CRDs) then pipe to kubectl:

```bash
helm template mongo-rs percona/psmdb-db -n percona -f psmdb-values.yaml --include-crds \
  | kubectl apply -n percona -f -
```

(Helm templating the DB chart is supported; Operator manages the CR.) ([Percona Community Forum][4])

---

#### Notes on your settings

* `sharding.enabled: true` with `mongos` and `configsvrReplSet` blocks matches the Operator’s sharding model. Keep `expose.type: ClusterIP` if you only need in-cluster access. ([Percona Documentation][5])
* Your `configuration:` stanzas placing `authenticationMechanisms` under each replset/configsvr are correct for modern Operator versions (options moved under `replsets[].configuration`). ([Percona Community Forum][6])

If you want, paste your **namespace** and whether the Operator is already running, and I’ll give you the exact commands tailored to your setup.

[1]: https://docs.percona.com/percona-operator-for-mongodb/expose.html?utm_source=chatgpt.com "Percona Operator for MongoDB - Exposing the cluster"
[2]: https://artifacthub.io/packages/helm/percona/psmdb-operator?utm_source=chatgpt.com "psmdb-operator 1.20.1 · percona/percona - artifacthub.io"
[3]: https://docs.percona.com/percona-operator-for-mongodb/helm.html?utm_source=chatgpt.com "With Helm - Percona Operator for MongoDB"
[4]: https://forums.percona.com/t/helm-chart-percona-psmdb-db-does-include-crds/17518?utm_source=chatgpt.com "Helm chart percona/psmdb-db does include CRDs"
[5]: https://docs.percona.com/percona-operator-for-mongodb/sharding.html?utm_source=chatgpt.com "MongoDB sharding - Percona Operator for MongoDB"
[6]: https://forums.percona.com/t/helm-upgrade-from-1-11-to-1-12-confusing-crds/17027/1?utm_source=chatgpt.com "Helm Upgrade from 1.11 to 1.12 confusing crds - Percona Operator for ..."



Percona Server for MongoDB cluster is deployed now. Get the username and password:

  ADMIN_USER=$(kubectl -n tugane-sit get secrets mongo-rs-psmdb-db-secrets -o jsonpath="{.data.MONGODB_USER_ADMIN_USER}" | base64 --decode)
  ADMIN_PASSWORD=$(kubectl -n tugane-sit get secrets mongo-rs-psmdb-db-secrets -o jsonpath="{.data.MONGODB_USER_ADMIN_PASSWORD}" | base64 --decode)

Connect to the cluster:

  kubectl run -i --rm --tty percona-client --image=percona/percona-server-mongodb:7.0 --restart=Never \
  -- mongosh "mongodb://${ADMIN_USER}:${ADMIN_PASSWORD}@mongo-rs-psmdb-db-mongos.tugane-sit.svc.cluster.local/admin?ssl=false"
aimemalaika@MacBook-Pro-3 mongo-db %   ADMIN_USER=$(kubectl -n tugane-sit get secrets mongo-rs-psmdb-db-secrets -o jsonpath="{.data.MONGODB_USER_ADMIN_USER}" | base64 --decode)
  ADMIN_PASSWORD=$(kubectl -n tugane-sit get secrets mongo-rs-psmdb-db-secrets -o jsonpath="{.data.MONGODB_USER_ADMIN_PASSWORD}" | base64 --decode)
aimemalaika@MacBook-Pro-3 mongo-db %
