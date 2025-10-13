You’re applying a **Helm values file** with `kubectl`.
`psmdb-values.yaml` doesn’t (and shouldn’t) contain `apiVersion` or `kind`, so `kubectl apply -f psmdb-values.yaml` errors with “apiVersion not set, kind not set”.

### Fix options

**A) Use Helm (correct way for a values.yaml)**

```bash
# if needed
helm repo add percona https://percona.github.io/percona-helm-charts
helm repo update

# install/upgrade using your values file
helm install psmdb <chart-name> -f psmdb-values.yaml
# or, if already installed
helm upgrade psmdb <chart-name> -f psmdb-values.yaml
```

Replace `<chart-name>` with the Percona chart you mean to use (e.g., the operator or the DB chart). You can discover it with:

```bash
helm search repo percona | grep psmdb
```

**B) If you really want to use `kubectl`, render first then apply**

```bash
helm template psmdb <chart-name> -f psmdb-values.yaml \
  | kubectl apply -f -
```

**C) If this file is supposed to be a Kubernetes manifest (not a Helm values file)**
Then it must include `apiVersion` and `kind` at the top of **every** document. Open `psmdb-values.yaml` and confirm whether it looks like:

```yaml
apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDB
metadata: ...
spec: ...
```

If not, it’s a values file; use A or B. If yes, the file name is misleading—rename it and ensure each `---`-separated document has those two fields.

---

If you paste a few top lines of your `psmdb-values.yaml` (just the headers), I can tell you which path you’re on and give you the exact Helm command.
