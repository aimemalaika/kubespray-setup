here’s a clean recap of what we did, why it worked, and what to keep:

# what was broken

* **Symptom:** pods on **worker-node-2** could ping pods on **worker-node-1** (ICMP OK), but **TCP/UDP timed out** (e.g., HTTP :8080, DNS to CoreDNS on w1).
* **Scope:** same behavior from host on w2 → pod on w1, so not just pod-to-pod; **cross-node overlay traffic** was the issue.

# how we proved it

1. **Test pods**

   * `web-a` (on w1) and `web-b` (on w2) with `hashicorp/http-echo`.
   * `client-w2` (on w2) & `client-w1` (on w1) for curl/wget/ping.
   * Result:

     * `client-w1 → web-a` ✅ (local node OK)
     * `client-w2 → web-a` ❌ (timeout)
     * `client-w2 → CoreDNS(w1) TCP/53` ❌ (timeout)
     * **ICMP** both ways ✅
   * ⇒ Pod config and service are fine; **overlay dataplane / host networking path** is suspect.

2. **Quick Calico/Kubespray checks**

   * Calico IPs matched node InternalIPs.
   * IPPool: `vxlanMode: Always`, `ipipMode: Never` ✅
   * No NetworkPolicy / GlobalNetworkPolicy blocking ✅
   * Pod MTU 1450 ✅

3. **Reverse-path filtering (rp\_filter)**

   * Found inconsistencies: `vxlan.calico` and some `cali*` ifaces had `rp_filter=2`/`1`.
   * Set **rp\_filter=0** on `vxlan.calico`, all `cali*`, and `all/default` on both nodes.
   * Helped ensure asymmetric overlay traffic isn’t dropped — **but timeouts persisted** (so rp\_filter wasn’t the only cause).

4. **Sniffing / rules sanity**

   * Looked at nft/iptables Calico chains; nothing obviously dropping data.
   * Started tcpdump attempts (pcaps/log) to confirm VXLAN traffic path.

5. **Breakthrough: NIC offloads**

   * Disabled offloads on **w1** (`vxlan.calico` + underlay NIC) → still ❌
   * Disabled offloads on **w2** as well → **HTTP from w2 → w1 started working** ✅
   * ⇒ Root cause: **GRO/TSO/GSO/RX/TX offload quirks** interacting with VXLAN/conntrack/nftables caused TCP/UDP timeouts (ICMP unaffected).

# what we changed (temporary → permanent)

* **Temporary fixes we tried**

  * `echo 0 > /proc/sys/net/ipv4/conf/{all,default}/rp_filter`
  * `echo 0 > /proc/sys/net/ipv4/conf/vxlan.calico/rp_filter`
  * `echo 0 > /proc/sys/net/ipv4/conf/cali*/rp_filter`
  * `ethtool -K vxlan.calico rx off tx off tso off gso off gro off`
  * `ethtool -K <underlay-iface> rx off tx off tso off gso off gro off`

* **Made it stick (cluster-wide)**

  1. **DaemonSet “offload fixer”** (your `calico-network-offload`):

     * On each node, at start:

       * Detects underlay interface (default route).
       * **Disables offloads** on underlay NIC **and** `vxlan.calico`.
       * Normalizes **`rp_filter=0`** on `all`, `default`, and existing ifaces.
     * This runs on every node and reapplies after reboot or node replacement.

  2. **Kubespray inventory hardening (so Ansible won’t undo things):**

     * In `inventory/<cluster>/group_vars/all/all.yml`:

       ```yaml
       sysctl_overwrite: true
       sysctl_file: 99-kubespray-calico.conf   # or set sysctl_file_path: "/etc/sysctl.d/99-kubespray-calico.conf"
       sysctl_vars:
         net.ipv4.conf.all.rp_filter: 0
         net.ipv4.conf.default.rp_filter: 0
       ```

       Apply with:

       ```bash
       ansible-playbook -i inventory/<cluster>/hosts.yaml cluster.yml --tags sysctl -b
       ```
     * For Calico autodetection sanity:

       ```bash
       kubectl -n kube-system set env ds/calico-node IP_AUTODETECTION_METHOD=kubernetes-internal-ip
       ```

       (Prevents weirdness from “can-reach=\$(NODEIP)” across reboots/changes.)

> Note: Kubespray doesn’t natively manage NIC offloads. Keep the **DaemonSet** (or create a systemd unit via Ansible) to enforce offload settings persistently.

# why this fixes it

* VXLAN encapsulates packets in UDP/4789. Certain NIC/driver combinations with GRO/TSO/GSO (and sometimes RX/TX checksum offloads) can confuse Linux conntrack/iptables-nft on decap/forward, especially for small TCP SYN/ACK packets. ICMP often slips through. Turning **offloads off** ensures the kernel sees sane packets on both sides of the tunnel.

# quick runbook (if it ever resurfaces)

1. Reproduce with a tiny echo pod on each node + a client pod.
2. If **ICMP OK** but **TCP/UDP time out**, check:

   * `rp_filter` on `vxlan.calico` / `cali*` / `all`/`default` → set to `0`.
   * IPPool modes (`vxlan: Always`, `ipip: Never`) and Calico peers (`ipset list cali40all-vxlan-net`).
3. Temporarily disable offloads on **both ends**:

   ```
   ethtool -K vxlan.calico rx off tx off tso off gso off gro off
   ethtool -K <underlay>     rx off tx off tso off gso off gro off
   ```

   If it fixes it → keep the DaemonSet/systemd approach.

# verify (anytime)

* `client-w2 → web-a`: `wget -qO- -T 2 http://<web-a-ip>:8080` should print the echo text.
* DNS from w2 to CoreDNS on w1:
  `nslookup kubernetes.default.svc.cluster.local 10.233.84.193` should succeed.
* Spot-check:

  ```
  cat /proc/sys/net/ipv4/conf/all/rp_filter        # 0
  cat /proc/sys/net/ipv4/conf/vxlan.calico/rp_filter # 0
  ip -d link show vxlan.calico                      # up, mtu 1450
  ```

that’s it. you fixed a tricky **Calico VXLAN + NIC offload** edge case, and you hardened the cluster so it stays fixed.


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

```bash
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
