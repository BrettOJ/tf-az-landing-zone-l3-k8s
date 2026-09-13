# Kubernetes 1.35 installation on NVIDIA Jetson Orin NX 16GB

Documented: 12 September 2026

## Purpose and installation record

Install upstream Kubernetes using **kubeadm, kubelet and kubectl**, with containerd and Flannel, on two physical Jetson Orin NX 16GB devices.

The user confirmed that the reset, persistent swap-disable and initialization retry procedure worked. The earlier failure included an API connection refusal and a missing kubelet configuration. Swap had reportedly returned after reboot; it was a suspected contributor, not conclusively established as the sole cause. Worker joining, application tests and GPU scheduling are documented below as follow-on steps, not as verified completed actions.

| Setting | Value |
| --- | --- |
| Kubernetes minor version | 1.35; select the same exact patch on both nodes |
| Control-plane IP | **192.168.1.18** |
| API endpoint | **https://192.168.1.18:6443** |
| Kubernetes control-plane node name in the commands | `jetson-01` |
| Kubernetes worker node name in the commands | `jetson-02` |
| Worker IP | Record the actual reserved/static address before installation |
| Pod CIDR | `10.244.0.0/16` |
| Service CIDR | `10.96.0.0/12` |
| Container runtime socket | `unix:///run/containerd/containerd.sock` |
| CNI | Flannel, VXLAN |
| Administration kubeconfig | `$HOME/.kube/jetson-v135-config` |

`JK8002` appeared as the Linux hostname in troubleshooting logs. Its role was not explicitly confirmed. The names `jetson-01` and `jetson-02` are the Kubernetes node names selected by `--node-name`; do not infer a physical-device mapping from them. Record that mapping locally.

This is a single-control-plane cluster with one worker. Applications can optionally run on the control plane too. It does **not** provide control-plane high availability. Persistent storage and ingress controllers are separate installations.

## How to use this runbook

- Fresh JetPack installation: start at section 1.
- Existing Kubernetes 1.32 installation to discard: run section 10 first, then sections 1–8, preserving any already-correct OS/runtime configuration.
- Failed new `kubeadm init`: use section 11, then retry section 6.
- Existing cluster to preserve: do **not** reset it. Kubeadm upgrades must proceed through 1.32 → 1.33 → 1.34 → 1.35 using each release's upgrade procedure.

All command blocks are Bash commands run on the Jetsons unless otherwise stated. Do not run the complete document as one script; it includes mutually exclusive paths and destructive reset procedures.

## 1. Base OS, addressing and connectivity — both nodes

Use the same carrier-board-supported JetPack 6.x release on both devices. JetPack 6.2.3 was referenced during this installation; exact installed JetPack versions were not captured. Use NVIDIA SDK Manager or your carrier-board vendor's supported flashing process. Prefer NVMe storage for the OS and container data.

Keep JetPack's NVIDIA kernel and driver stack. Do not install generic PC/server NVIDIA GPU drivers over it.

```bash
sudo apt update
sudo apt install -y curl ca-certificates gpg openssh-server
sudo systemctl enable --now ssh

uname -m
cat /etc/nv_tegra_release
dpkg-query -W nvidia-l4t-core
hostname
ip -br -4 addr
```

Expected architecture: `aarch64`. Reserve addresses in DHCP or configure static addressing. On the control plane, `192.168.1.18` must be assigned to a local interface; it is not a Kubernetes Service ClusterIP.

Use unique Linux hostnames. If desired on fresh devices, set them with `sudo hostnamectl set-hostname jetson-01` and `sudo hostnamectl set-hostname jetson-02`, respectively. Update local name resolution accordingly. Renaming an already-joined node is outside this procedure.

Ensure Pod and Service CIDRs do not overlap with your LAN, VPN or other routed networks.

| Port | Required access |
| --- | --- |
| TCP 22 | Admin workstation → both nodes |
| TCP 6443 | Worker and admin workstation → control plane |
| TCP 10250 | Control plane → nodes; monitoring components → nodes when installed |
| UDP 8472 | Between nodes for Flannel VXLAN; keep private |
| TCP/UDP 30000–32767 | Clients → nodes, only as required for exposed NodePort services |

The control plane also uses etcd ports 2379–2380 and scheduler/controller-manager ports 10259/10257. These are not client-facing ports. Firewalls must permit Pod forwarding and the selected CNI traffic, not just the API connection.

## 2. Disable swap permanently — both nodes

**Critical: `swapoff -a` alone does not survive reboot.** Disable both swap entries in `/etc/fstab` and Jetson's zram initialization service.

```bash
sudo swapoff -a

sudo cp -a /etc/fstab "/etc/fstab.backup-$(date +%Y%m%d-%H%M%S)"

sudo sed -i \
  '/^[[:space:]]*#/! { /[[:space:]]swap[[:space:]]/ s/^/#/; }' \
  /etc/fstab

if systemctl cat nvzramconfig.service >/dev/null 2>&1; then
  sudo systemctl disable --now nvzramconfig.service
fi

sudo systemctl daemon-reload
swapon --show
free -h
```

Expected: no output from `swapon --show`; total swap is `0B` in `free -h`.

If swap returns after reboot, identify its owner before proceeding:

```bash
systemctl list-units --type=swap --all
systemctl list-unit-files | grep -Ei 'swap|zram'
```

Disable the responsible service or persistent swap definition; avoid changing unrelated services. Setting `vm.swappiness=0` is not a substitute for disabling swap.

## 3. Kernel networking and cgroup v2 — both nodes

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
sudo modprobe vxlan

cat <<'EOF' | sudo tee /etc/modules-load.d/kubernetes.conf
overlay
br_netfilter
vxlan
EOF

cat <<'EOF' | sudo tee /etc/sysctl.d/99-kubernetes.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
sudo timedatectl set-ntp true
stat -fc %T /sys/fs/cgroup
```

Expected filesystem: `cgroup2fs`. This runbook requires cgroup v2 for Kubernetes 1.35. If a required kernel module is missing, resolve it against your installed Jetson BSP/kernel before proceeding.

### If cgroup v2 is not enabled

For Jetsons using `/boot/extlinux/extlinux.conf`, back up and edit the active boot entry:

```bash
sudo cp -a /boot/extlinux/extlinux.conf \
  "/boot/extlinux/extlinux.conf.backup-$(date +%Y%m%d-%H%M%S)"
sudo nano /boot/extlinux/extlinux.conf
```

Append `systemd.unified_cgroup_hierarchy=1` to the existing active `APPEND` line, preserving all other required boot arguments. Replace an explicit conflicting `systemd.unified_cgroup_hierarchy=0` argument if present. If the vendor image uses a different boot mechanism, use its supported method instead.

Reboot both nodes after the swap and kernel preparation:

```bash
sudo reboot
```

After reconnecting:

```bash
swapon --show
stat -fc %T /sys/fs/cgroup
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
```

**Gate:** no active swap, `cgroup2fs`, and both sysctl values equal `1`.

## 4. Containerd — both nodes

First inspect existing packages:

```bash
containerd --version
dpkg-query -W containerd containerd.io 2>/dev/null
```

Use one containerd installation. If JetPack already installed `containerd.io`, retain it rather than installing the conflicting Ubuntu `containerd` package. Use a current, compatible containerd 1.7.x or 2.x package for this Kubernetes 1.35 setup.

For a fresh device without containerd:

```bash
sudo apt update
sudo apt install -y containerd
```

### Fresh runtime configuration only

The following generates a new runtime configuration. **Skip regeneration if containerd already has a correct configuration or NVIDIA runtime entries you need to preserve.** Inspect and update the existing configuration instead.

```bash
sudo mkdir -p /etc/containerd
if [ -f /etc/containerd/config.toml ]; then
  sudo cp -a /etc/containerd/config.toml \
    "/etc/containerd/config.toml.backup-$(date +%Y%m%d-%H%M%S)"
fi

containerd config default \
  | sudo tee /etc/containerd/config.toml >/dev/null

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' \
  /etc/containerd/config.toml

sudo systemctl enable --now containerd
sudo systemctl restart containerd
```

Verify the effective configuration:

```bash
sudo systemctl is-active containerd
sudo containerd config dump | grep -n SystemdCgroup
```

The default `runc` runtime must use `SystemdCgroup = true`. CRI must be enabled. Containerd 1.x and 2.x use different plugin configuration sections, so generate defaults with the installed binary rather than pasting a version-specific full configuration.

## 5. Install Kubernetes 1.35 packages — both nodes

For an existing cluster, complete the reset/removal procedure in section 10 before replacing packages. Do not use these commands as an in-place multi-minor upgrade.

```bash
sudo install -d -m 0755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \
  | sudo gpg --dearmor --yes \
      -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
apt-cache madison kubeadm
```

Check for duplicate/old repositories and disable obsolete Kubernetes entries if found:

```bash
grep -RnsE 'pkgs\.k8s\.io|apt\.kubernetes\.io' \
  /etc/apt/sources.list /etc/apt/sources.list.d 2>/dev/null
```

On the control plane, select and record the newest available 1.35 package version:

```bash
JETSON_K8S_PACKAGE_VERSION=$(
  apt-cache madison kubeadm \
    | awk '$3 ~ /^1\.35\./ {print $3; exit}'
)
printf 'Selected package version: %s\n' "$JETSON_K8S_PACKAGE_VERSION"
```

Stop if that variable is empty. On the worker, set it to the exact full package version selected on the control plane:

```bash
read -rp "Enter the full package version selected on the control plane: " \
  JETSON_K8S_PACKAGE_VERSION
```

On each node, with the variable set:

```bash
sudo apt install -y \
  kubeadm="$JETSON_K8S_PACKAGE_VERSION" \
  kubelet="$JETSON_K8S_PACKAGE_VERSION" \
  kubectl="$JETSON_K8S_PACKAGE_VERSION"

sudo apt-mark hold kubeadm kubelet kubectl
sudo systemctl enable kubelet

kubeadm version -o short
kubelet --version
kubectl version --client
```

Record the exact installed versions. The kubelet can restart with a missing `/var/lib/kubelet/config.yaml` before initialization/join; kubeadm creates that file. Do not create an empty file or copy another node's configuration.

## 6. Initialize the control plane — 192.168.1.18 only

Run only on the device whose local interface has `192.168.1.18`:

```bash
ip -br -4 addr
swapon --show
stat -fc %T /sys/fs/cgroup
sudo systemctl is-active containerd
kubeadm version -o short
```

Once checks pass:

```bash
sudo systemctl enable --now kubelet

sudo kubeadm init \
  --kubernetes-version="$(kubeadm version -o short)" \
  --apiserver-advertise-address=192.168.1.18 \
  --control-plane-endpoint=192.168.1.18:6443 \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --cri-socket=unix:///run/containerd/containerd.sock \
  --node-name=jetson-01 \
  --v=5
```

Wait for the successful initialization message and retain the join command securely. Do not put join tokens or kubeconfig credentials into this document or source control.

Configure local administrative access:

```bash
mkdir -p "$HOME/.kube"

sudo install -m 600 -o "$(id -u)" -g "$(id -g)" \
  /etc/kubernetes/admin.conf "$HOME/.kube/jetson-v135-config"

export KUBECONFIG="$HOME/.kube/jetson-v135-config"
kubectl get nodes
```

Run the `export` again in new administration shells, or pass `--kubeconfig` explicitly. Keep this file separate from AKS/other cluster credentials. For workstation access, securely copy this kubeconfig and verify the API endpoint is `https://192.168.1.18:6443`. It grants cluster-admin access.

The node can show `NotReady` before the CNI is installed.

## 7. Install Flannel — control plane administration shell

The upstream standard manifest uses the matching Pod CIDR `10.244.0.0/16`:

```bash
curl -fL \
  https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml \
  -o "$HOME/kube-flannel.yml"

kubectl apply -f "$HOME/kube-flannel.yml"
kubectl get pods -n kube-flannel -o wide
kubectl get pods -n kube-system
```

For future repeatable installations, retain this manifest and record its image versions; `latest` can change. Review compatibility before adopting a newer Flannel release. Confirm `/opt/cni/bin` contains the required CNI binaries if Flannel reports missing executables.

## 8. Join the worker and verify

Prepare the worker using sections 1–5 first, including reboot verification of swap. On the control plane, generate a join command when needed:

```bash
sudo kubeadm token create --print-join-command
```

Run the returned command on the worker, adding the socket and node name:

```bash
sudo kubeadm join 192.168.1.18:6443 \
  --token <NEW-TOKEN> \
  --discovery-token-ca-cert-hash sha256:<NEW-HASH> \
  --cri-socket=unix:///run/containerd/containerd.sock \
  --node-name=jetson-02
```

Replace the placeholders with the generated values. If the worker belonged to the discarded cluster, reset it before joining this new cluster. Old join credentials do not apply.

On the control plane:

```bash
kubectl wait --for=condition=Ready \
  node/jetson-01 node/jetson-02 --timeout=300s

kubectl get nodes -o wide
kubectl get pods -A -o wide
```

Both nodes should show `Ready` and `v1.35.x`. A worker role displayed as `<none>` is normal.

### Optional: run application workloads on both devices

```bash
kubectl taint nodes jetson-01 \
  node-role.kubernetes.io/control-plane:NoSchedule-
```

Leave the taint in place to dedicate the first device to the control plane. Budget CPU and memory for the control plane when running applications there.

### Application and cross-node placement test

After permitting applications on both nodes:

```bash
kubectl create deployment web-test --image=nginx:stable-alpine
kubectl scale deployment web-test --replicas=2

kubectl patch deployment web-test --type=merge -p '{
  "spec": {
    "template": {
      "spec": {
        "topologySpreadConstraints": [{
          "maxSkew": 1,
          "topologyKey": "kubernetes.io/hostname",
          "whenUnsatisfiable": "DoNotSchedule",
          "labelSelector": {"matchLabels": {"app": "web-test"}}
        }]
      }
    }
  }
}'

kubectl rollout status deployment/web-test
kubectl get pods -l app=web-test -o wide
kubectl expose deployment web-test --type=NodePort --port=80 --target-port=80
kubectl get service web-test
```

Expect one replica per eligible node. Open `http://192.168.1.18:<allocated-nodeport>` and the equivalent worker address. This checks HTTP exposure; use targeted Pod-to-Pod tests when diagnosing cross-node networking specifically.

Remove the test afterwards:

```bash
kubectl delete service web-test
kubectl delete deployment web-test
```

## 9. Optional NVIDIA GPU scheduling

This section documents the proposed follow-on configuration; successful GPU validation was not reported in the conversation. Run ordinary Kubernetes workloads successfully before adding GPU scheduling.

### Host runtime — both GPU nodes

Use JetPack's GPU drivers. Install the toolkit from the configured JetPack-compatible repositories:

```bash
sudo apt update
sudo apt install -y nvidia-container-toolkit
nvidia-container-runtime --version
nvidia-ctk --version
```

If the package is unavailable, follow NVIDIA's official toolkit repository instructions listed in the references, checking compatibility with the installed JetPack release.

Back up the containerd configuration before changing it:

```bash
sudo cp -a /etc/containerd/config.toml \
  "/etc/containerd/config.toml.before-nvidia-$(date +%Y%m%d-%H%M%S)"

sudo nvidia-ctk runtime configure --runtime=containerd
sudo systemctl restart containerd
sudo containerd config dump | grep -n -E 'nvidia|SystemdCgroup'
```

Keep the default general-purpose runtime configured correctly; select NVIDIA explicitly using RuntimeClass. Inspect effective runtime configuration if GPU containers report cgroup/runtime errors.

### RuntimeClass and device plugin — administration shell

```bash
kubectl apply -f - <<'EOF'
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: nvidia
handler: nvidia
EOF

kubectl label nodes jetson-01 jetson-02 \
  nvidia.com/gpu.present=true --overwrite
```

Install Helm on your ARM64 administration node or use an existing Helm-equipped workstation with this kubeconfig. See the official Helm installation reference.

```bash
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update

helm upgrade --install nvidia-device-plugin nvdp/nvidia-device-plugin \
  --version 0.17.1 \
  --namespace nvidia-device-plugin \
  --create-namespace \
  --set runtimeClassName=nvidia \
  --set deviceListStrategy=envvar \
  --set failOnInitError=true \
  --set-string 'nodeSelector.nvidia\.com/gpu\.present=true'

kubectl get pods -n nvidia-device-plugin -o wide
kubectl get nodes \
  -o custom-columns='NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu'
```

Version `0.17.1` is the pinned example discussed in this installation, not a claim of the latest release or certification of this exact hardware/software combination. Validate Tegra detection with the actual JetPack/toolkit versions before relying on GPU scheduling. Each Jetson should advertise one GPU. Inspect the device-plugin Pod's logs if it does not.

GPU workload Pod-spec fragment (replace the image before use):

```yaml
spec:
  runtimeClassName: nvidia
  containers:
    - name: application
      image: <jetpack-compatible-arm64-image>
      resources:
        limits:
          nvidia.com/gpu: 1
```

For a Deployment, place this under `spec.template.spec`. Use images compatible with ARM64 **and** the installed JetPack/L4T release. Run a real CUDA/inference workload on each node for final validation; GPU resource registration alone is insufficient. If pulling from ACR, preserve the ARM64 image variant and configure appropriate imagePullSecrets.

## 10. Optional: discard Kubernetes 1.32 and reinstall 1.35

**Destructive: use only when deliberately discarding the old cluster.** Back up required application state, manifests, secrets and persistent data separately. Kubeadm reset is not a backup or an application-data migration.

Reset the old worker first, then the control plane, using the currently installed kubeadm before removing packages:

```bash
sudo kubeadm reset --cri-socket=unix:///run/containerd/containerd.sock
sudo systemctl stop kubelet
```

Confirm the reset prompt. Stop if reset reports container cleanup or volume-unmount failures.

On each reset node:

```bash
if [ -d /etc/cni/net.d ]; then
  sudo mv /etc/cni/net.d \
    "/etc/cni/net.d.backup-$(date +%Y%m%d-%H%M%S)"
fi
sudo mkdir -p /etc/cni/net.d

sudo apt-mark unhold kubeadm kubelet kubectl
sudo apt purge -y kubeadm kubelet kubectl
sudo systemctl daemon-reload
```

If only the packages were installed and the node never ran init/join, skip the reset. Retain containerd and the NVIDIA stack. Do not run indiscriminate package autoremove or delete containerd's data directory.

Continue with sections 1–8. Replace the old repository with 1.35 in section 5 and use new join credentials. Reset removes local stacked-etcd membership/state; it does not wipe an external etcd cluster, all CNI state or all host firewall rules. Do not flush the host firewall indiscriminately.

## 11. Recovery: failed init, swap returned after reboot

Use this sequence only on the failed new control-plane installation at `192.168.1.18`:

1. Run `sudo kubeadm reset --cri-socket=unix:///run/containerd/containerd.sock` and confirm.
2. Run `sudo systemctl stop kubelet`.
3. Move any old `/etc/cni/net.d` aside as shown in section 10.
4. Apply section 2 to disable both normal swap and `nvzramconfig`.
5. Reboot and confirm `swapon --show` is empty; check cgroup v2 and containerd too.
6. Retry the full initialization command in section 6.
7. Refresh the administration kubeconfig, install Flannel, and join the worker using the new token/hash.

There is no need to reinstall Kubernetes packages for this failed-init recovery. A missing kubelet config between reset and init/join is expected.

## 12. Troubleshooting

### API connection refused on 192.168.1.18:6443

Example error from this installation:

```text
could not bootstrap the admin user in file admin.conf
unable to create ClusterRoleBinding
dial tcp 192.168.1.18:6443: connect: connection refused
```

The RBAC creation failed because the API connection could not be established. Check the local address, API listener, kubelet and control-plane containers before changing RBAC or reinstalling networking.

```bash
ip -br -4 addr
sudo ss -lntp 'sport = :6443'
sudo systemctl status kubelet containerd --no-pager -l
sudo journalctl -u kubelet -b -n 100 --no-pager
sudo journalctl -u containerd -b -n 100 --no-pager

sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps -a
```

Inspect the latest API-server and etcd container logs, including exited containers:

```bash
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock \
  logs --tail=100 <KUBE-APISERVER-CONTAINER-ID>

sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock \
  logs --tail=100 <ETCD-CONTAINER-ID>
```

If `crictl` is absent, install the `cri-tools` package from the configured Kubernetes package source. Static control-plane Pods use host networking and do not need Flannel to be installed first.

### Missing /var/lib/kubelet/config.yaml

```bash
sudo ls -l /var/lib/kubelet/config.yaml \
  /var/lib/kubelet/kubeadm-flags.env \
  /etc/kubernetes/kubelet.conf
sudo ls -l /etc/kubernetes/manifests/
sudo journalctl -u kubelet --since "5 minutes ago" --no-pager
```

| State | Action |
| --- | --- |
| Fresh/reset node before init/join | Expected; complete init on the control plane or join on the worker |
| Worker not joined | Join once the control plane is healthy |
| Failed new control plane being discarded | Use section 11 |
| Previously working node, unexpectedly missing file | Investigate deletion/configuration changes; do not fabricate a config |

### Other useful checks

```bash
swapon --show
stat -fc %T /sys/fs/cgroup
sudo containerd config dump | grep -n SystemdCgroup
sudo modprobe br_netfilter
sysctl net.bridge.bridge-nf-call-iptables

kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl get pods -A -o wide
kubectl describe node jetson-02
```

| Symptom | Investigate |
| --- | --- |
| Swap returns after reboot | fstab and swap/zram services |
| Kubelet cgroup error | cgroup v2 and matching systemd cgroup drivers |
| Flannel missing bridge sysctl | `br_netfilter` availability and persistent module loading |
| Cross-node Pod traffic fails | UDP 8472, forwarding, CIDR overlap, interface selection |
| Container image pull fails | Registry connectivity, credentials, ARM64 manifest availability |
| `exec format error` | Wrong CPU architecture in image/binary |
| GPU plugin starts but advertises no GPU | NVIDIA runtime and JetPack/Tegra discovery logs |

## 13. Post-installation record and boundaries

Capture these non-secret details for future rebuilds:

```bash
date -Is
hostname
ip -br -4 addr
cat /etc/nv_tegra_release
dpkg-query -W kubeadm kubelet kubectl nvidia-l4t-core
containerd --version
kubectl get nodes -o wide
kubectl -n kube-flannel get daemonsets -o wide
```

Record the worker IP, physical-node mapping, exact JetPack/package versions and retained Flannel manifest. Never commit kubeconfig credentials, tokens or private keys.

- Kubeadm installs no default StorageClass. Select a storage provisioner separately; local disks are not automatically replicated between nodes.
- Install an ingress controller/Gateway API implementation separately when needed.
- Configure backups for application data and the control-plane datastore.
- Follow supported sequential kubeadm upgrade procedures rather than replacing all held packages with a newer minor version.
- A two-node installation does not pool the two GPUs or their memory into one GPU automatically.

## Official references

These are the primary references used during the installation discussion. Versioned Kubernetes documentation is preferable when rebuilding 1.35; unversioned pages and `latest` downloads can change.

- [Kubernetes 1.35: installing kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Kubernetes 1.35: container runtimes](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/)
- [Kubeadm reset](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/)
- [Kubeadm upgrades](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [Kubelet integration with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/)
- [Troubleshooting kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/)
- [Debugging with crictl](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/)
- [Kubernetes ports and protocols](https://kubernetes.io/docs/reference/networking/ports-and-protocols/)
- [Flannel project and installation](https://github.com/flannel-io/flannel)
- [NVIDIA JetPack 6.2.3](https://developer.nvidia.com/embedded/jetpack-sdk-623)
- [NVIDIA SDK Manager](https://developer.nvidia.com/sdk-manager)
- [NVIDIA discussion: disable Orin zram swap](https://forums.developer.nvidia.com/t/remove-agx-orin-swap/240222)
- [NVIDIA Container Toolkit installation](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
- [NVIDIA Kubernetes device plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [Installing Helm](https://helm.sh/docs/intro/install/)
