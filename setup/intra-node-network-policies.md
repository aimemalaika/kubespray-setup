```YAML
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: calico-network-offload
  namespace: kube-system
spec:
  selector:
    matchLabels: {app: calico-network-offload}
  template:
    metadata:
      labels: {app: calico-network-offload}
    spec:
      hostNetwork: true
      hostPID: true
      tolerations:
      - operator: Exists
      priorityClassName: system-cluster-critical
      containers:
      - name: fixer
        image: nicolaka/netshoot:latest
        securityContext:
          privileged: true
        command: ["/bin/sh","-lc"]
        args:
        - |
          set -eu
          # 1) pick the underlay NIC (the one with the default route)
          IFACE="$(ip route show default | awk '/default/ {print $5; exit}')"
          echo "Underlay IFACE=$IFACE"

          # 2) disable problematic offloads on underlay + vxlan.calico
          for DEV in "$IFACE" vxlan.calico; do
            if ip link show "$DEV" >/dev/null 2>&1; then
              echo "Disabling offloads on $DEV"
              ethtool -K "$DEV" rx off tx off tso off gso off gro off || true
            fi
          done

          # 3) normalize rp_filter so new ifaces inherit 0; also flip existing ones
          echo 0 > /proc/sys/net/ipv4/conf/all/rp_filter || true
          echo 0 > /proc/sys/net/ipv4/conf/default/rp_filter || true
          for f in /proc/sys/net/ipv4/conf/*/rp_filter; do echo 0 > "$f" || true; done

          # stay alive so it persists across reboots/restarts
          sleep infinity
```
OR

```
cat <<'YAML' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: calico-network-offload
  namespace: kube-system
spec:
  selector:
    matchLabels: {app: calico-network-offload}
  template:
    metadata:
      labels: {app: calico-network-offload}
    spec:
      hostNetwork: true
      hostPID: true
      tolerations:
      - operator: Exists
      priorityClassName: system-cluster-critical
      containers:
      - name: fixer
        image: nicolaka/netshoot:latest
        securityContext:
          privileged: true
        command: ["/bin/sh","-lc"]
        args:
        - |
          set -eu
          # 1) pick the underlay NIC (the one with the default route)
          IFACE="$(ip route show default | awk '/default/ {print $5; exit}')"
          echo "Underlay IFACE=$IFACE"

          # 2) disable problematic offloads on underlay + vxlan.calico
          for DEV in "$IFACE" vxlan.calico; do
            if ip link show "$DEV" >/dev/null 2>&1; then
              echo "Disabling offloads on $DEV"
              ethtool -K "$DEV" rx off tx off tso off gso off gro off || true
            fi
          done

YAML
```
