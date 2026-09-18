# Talos Linux — 1 Control Plane + 2 Workers

Setup and lifecycle for a small Talos Linux + Kubernetes cluster:
**1 control plane node + 2 worker nodes** (home lab / dev / small prod).

> Notes:
> - Replace `<k8s-version>`, `<talos-version>` and `<schematic-id>` with your actual versions —
>   read them from `talosctl version --nodes <cp-ip>` and `kubectl version`.
> - No env vars are exported. Configs live in the current directory (`./configs`): every
>   `talosctl` call that talks to a running node carries `--talosconfig ./configs/talosconfig`.
>   `apply-config --insecure` works without it, because the node is still in maintenance mode.
> - Compatibility: each Talos minor supports **6 adjacent Kubernetes minors**. Check yours:
>   https://docs.siderolabs.com/talos/latest/getting-started/support-matrix
>   **Upgrade Talos before Kubernetes**, and both **one minor at a time** — never skip.
> - Flannel CNI (default), bare metal.

---

## 1. Install talosctl

```bash
# Installer script (macOS / Linux / WSL):
curl -sL https://talos.dev/install | sh

talosctl version --client
```

Your `talosctl` client should be **≥ the node version**.

## 2. Boot and configure the control plane

Boot the CP server from the ISO, then:

```bash
talosctl gen config my-cluster https://<lb-or-dns>:6443 \
  --with-secrets secrets.yaml \
  --kubernetes-version <k8s-version> \
  --output-dir ./configs
# (omit --kubernetes-version to use your client's Talos default)
# generates ./configs/{controlplane.yaml,worker.yaml,talosconfig}

# optional static IP for the CP (choose one; otherwise DHCP)
talosctl machineconfig patch ./configs/controlplane.yaml \
  --patch @cp-ip.yaml --output ./configs/cp.yaml

talosctl apply-config --insecure --nodes <cp-ip> --file ./configs/cp.yaml
talosctl bootstrap --talosconfig ./configs/talosconfig --nodes <cp-ip>   # ONCE
talosctl health --talosconfig ./configs/talosconfig --wait-timeout 10m

talosctl config endpoint <cp-ip> --talosconfig ./configs/talosconfig
talosctl kubeconfig ./kubeconfig --talosconfig ./configs/talosconfig --nodes <cp-ip>

# 'kubeconfig' writes ./kubeconfig in this dir — move it wherever you want,
# e.g. to the default location, so plain 'kubectl' works:
#   mkdir -p ~/.kube && mv ./kubeconfig ~/.kube/config
```

## 3. Join the 2 workers

```bash
# boot each worker from the ISO, then:
talosctl apply-config --insecure --nodes <worker1-ip> --file ./configs/worker.yaml
talosctl apply-config --insecure --nodes <worker2-ip> --file ./configs/worker.yaml

talosctl health --talosconfig ./configs/talosconfig -n <worker1-ip> --wait-timeout 10m
talosctl health --talosconfig ./configs/talosconfig -n <worker2-ip> --wait-timeout 10m

kubectl get nodes -o wide      # expect: 1 control-plane + 2 workers, all Ready
```

## 4. Day-2 basics

```bash
talosctl health --talosconfig ./configs/talosconfig --wait-timeout 10m
talosctl reboot --talosconfig ./configs/talosconfig -n <ip>          # reboot a node
talosctl logs kubelet --talosconfig ./configs/talosconfig -n <ip>
talosctl etcd snapshot --talosconfig ./configs/talosconfig \
  ./etcd-$(date +%F).snapshot --nodes <cp-ip>     # backup, run regularly
```

## 5. Scaling

### Add another worker

```bash
talosctl apply-config --insecure --nodes <new-ip> --file ./configs/worker.yaml
talosctl health --talosconfig ./configs/talosconfig -n <new-ip> --wait-timeout 10m
kubectl get nodes
```

### Remove a worker

```bash
kubectl cordon <worker>
kubectl drain <worker> --ignore-daemonsets --delete-emptydir-data --grace-period=120 --timeout=600s
kubectl delete node <worker>
talosctl reset --graceful --talosconfig ./configs/talosconfig -n <worker-ip>
```

### Grow the control plane (1 → 3)

Keep the count **odd** (1, 3, 5). Generate configs for CP nodes 2 and 3 with static IPs, apply,
then verify etcd joins:

```bash
talosctl etcd members --talosconfig ./configs/talosconfig -n <cp-ip>  # after adding: 3 members
kubectl get nodes -o wide
talosctl config endpoint --talosconfig ./configs/talosconfig <cp1-ip> <cp2-ip> <cp3-ip>
```

## 6. Upgrades

### Kubernetes (zero-ish downtime)

```bash
talosctl upgrade-k8s --talosconfig ./configs/talosconfig --nodes <cp-ip> --to <k8s-version> --dry-run
talosctl upgrade-k8s --talosconfig ./configs/talosconfig --nodes <cp-ip> \
  --to <k8s-version> --upgrade-kubelet=false
```

> `--upgrade-kubelet=false` requires a modern talosctl. It upgrades the control-plane components
> cluster-wide but **skips the kubelet**, which you then roll node-by-node below. (The default
> behavior upgrades the kubelet on all nodes at once and can restart workloads.)

Then roll the kubelet one node at a time (avoids workload restarts). First, check your config
format — it decides the patch syntax:

```bash
talosctl get machineconfig -o yaml --talosconfig ./configs/talosconfig --nodes <ip>
```

```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

Patch the kubelet image (use the form that matches your config):

```bash
# Multi-document config (configs generated on Talos >= v1.13) — strategic-merge patch:
talosctl patch mc --mode=no-reboot --talosconfig ./configs/talosconfig -n <ip> -p "apiVersion: v1alpha1
kind: KubeletConfig
image: ghcr.io/siderolabs/kubelet:v<k8s-version>"

# Single-document config (Talos < v1.13):
talosctl patch mc --mode=no-reboot --talosconfig ./configs/talosconfig -n <ip> -p "machine:
  kubelet:
    image: ghcr.io/siderolabs/kubelet:v<k8s-version>"
```

Then wait, verify, and allow scheduling:

```bash
kubectl wait --for=jsonpath='{.status.nodeInfo.kubeletVersion}'=v<k8s-version> node/<node> --timeout=15m
talosctl health --talosconfig ./configs/talosconfig -n <ip> --wait-timeout 10m
kubectl uncordon <node>
```

Run for each worker (one at a time), then the control plane last. Verify:
`kubectl get nodes -o wide` (all on `<k8s-version>`) and `kubectl version`.

### Talos OS

Workers first, control plane last:

```bash
talosctl upgrade --talosconfig ./configs/talosconfig --nodes <ip> \
  --image factory.talos.dev/installer/<schematic-id>:<talos-version> --preserve
talosctl health --talosconfig ./configs/talosconfig -n <ip> --wait-timeout 10m
```

Optional: `--stage` prepares the upgrade now and reboots later; `--no-reboot` (v1.13.7+)
downloads without applying.

Rollback to the previous version:
`talosctl rollback --nodes <ip> --talosconfig ./configs/talosconfig` (A/B slots — previous
version only).
