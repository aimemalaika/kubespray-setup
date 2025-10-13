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

> Check status:

```bash
kubectl -n tugane-sit get psmdb
kubectl -n tugane-sit get pods -l app.kubernetes.io/instance=psmdb
```

## Apply values

```bash
```bash
kubectl apply -f filename.yam
```
