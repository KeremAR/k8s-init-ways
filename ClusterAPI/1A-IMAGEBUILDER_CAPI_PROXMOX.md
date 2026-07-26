# CAPI + Proxmox + Image Builder — Setup Notes

Based on [this video](https://www.youtube.com/watch?v=G72ylsRmspY). Goal: build a CAPI-compatible Proxmox VM template, then use Cluster API to provision K8s clusters on a two-node homelab Proxmox cluster.

## Architecture decision: where things run

| Task | Machine | Why |
|---|---|---|
| Image Builder (Packer + Ansible template build) | Native Ubuntu box | WSL2 NAT breaks inbound connections from the build VM back to the HTTP server (see Issue 3) |
| clusterctl / kind / kubectl (CAPI provisioning) | Windows + WSL2 | This direction is outbound-only (WSL → Proxmox API), no NAT issue applies |

Management cluster (kind) is kept temporary/disposable — it's only needed to bootstrap CAPI, then `clusterctl move` migrates CAPI itself into the real cluster, and kind is deleted. No need to keep it running 24/7 unless using GitOps (which we're explicitly not doing — overkill for 2 small clusters).

---

## Step 1: Proxmox API user + token

> In a Proxmox cluster, users and API tokens are Datacenter-wide resources. Create the user/token once from any node's web interface; Proxmox automatically makes it available across all cluster nodes.

`Datacenter → Permissions → Users → Add`
- Username: `capi`, Realm: `pve`

`Datacenter → Permissions → Add → User Permission`
- User: `capi@pve`, Role: `PVEAdmin`, Path: `/`

`Datacenter → Permissions → API Tokens → Add`
- User: `capi@pve`, Token ID: `capitoken`
- **Uncheck "Privilege Separation"**
- Secret is shown only once — save it immediately.

Result:
```
Username (for env var): capi@pve!capitoken
Token secret:           9d4f1434-3d92-40bd-9284-746ea7ade180
```

This is used both by Image Builder (to upload ISO / create the template VM) and by clusterctl's Proxmox provider (to create workload cluster VMs).

---

## Step 2: Install tooling (Ubuntu / WSL Ubuntu)

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.32.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# Packer
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y packer

# clusterctl
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.13.2/clusterctl-linux-amd64 -o clusterctl
chmod +x clusterctl && sudo mv clusterctl /usr/local/bin/

# misc deps used throughout
sudo apt install -y make git unzip ansible
```

---

## Step 3: Image Builder — build the Proxmox template

Official docs: https://image-builder.sigs.k8s.io/capi/providers/proxmox

```bash
git clone https://github.com/kubernetes-sigs/image-builder.git
cd image-builder/images/capi
```

The build prerequisites for using image-builder for building Proxmox VM templates are managed by running the following command:

```bash
make deps-proxmox
```

From the `images/capi` directory, run `make build-proxmox-<OS>` where `<OS>` is the desired operating system. The available choices are listed via:

```bash
make help | grep proxmox
```

The full list of available environment vars can be found in the variables section of `images/capi/packer/proxmox/packer.json`.

Each variable in this section can also be overridden via the `PACKER_FLAGS` environment var.

<details>
<summary><strong>Issue 1 — Ansible version too old via apt</strong></summary>

`make deps-proxmox` requires Ansible ≥ 2.18 core. Ubuntu's apt package is too old (2.16.x), and Ubuntu 24.04's PEP 668 blocks system-wide `pip install` (`externally-managed-environment` error).

**Fix:** use a venv.
```bash
sudo apt install python3.12-venv
python3 -m venv ~/image-builder-venv
source ~/image-builder-venv/bin/activate
pip install --upgrade pip
pip install "ansible>=13.0.0"
```
> Note: "Ansible" package version ≠ "ansible-core" version — check the [release table](https://docs.ansible.com/projects/ansible/latest/reference_appendices/release_and_maintenance.html#ansible-community-changelogs) if unsure which Ansible version maps to which core version.

**Remember:** this venv must be re-activated (`source ~/image-builder-venv/bin/activate`) every new terminal session — it's not a system-wide install.

</details>

<details>
<summary><strong>Issue 2 — incompatible Packer version</strong></summary>

Packer's own version-pinning logic downloads a compatible Packer binary into `.local/bin` (the apt-installed Packer is usually too new/incompatible and gets ignored). This step needs `unzip`. No need to do anything, it's just ignored — not a real issue actually, but might become one in the future.

</details>

<details>
<summary><strong>Issue 3 — Build VM stuck on "Network Stage" / language-selection screen (the big one)</strong></summary>

Symptom: VM boots, autoinstall is supposed to kick in automatically, but instead it lands on the **interactive Ubuntu installer language-selection screen**.

**Root cause chain:**
1. Ubuntu's autoinstall uses `nocloud-net` datasource — the boot VM fetches its install config over HTTP from `http://{{ .HTTPIP }}:{{ .HTTPPort }}/...`, where `HTTPIP` is auto-detected by Packer as **the IP of the machine running Packer** (because Packer's own temporary HTTP server hosts the config).
2. On WSL2 (default NAT mode), `ip addr show` reports multiple interfaces (loopback alias, `eth0`, link-local `eth2`, etc). Packer picked a useless one (`10.255.255.254`, a loopback alias) — completely unreachable from the Proxmox VM.
3. Switched WSL2 to **mirrored networking mode** (`%USERPROFILE%\.wslconfig`: `[wsl2]` `networkingMode=mirrored`, then `wsl --shutdown`) so WSL shares the host's real LAN IP (`192.168.0.26`).
   - Gotcha: `wsl --shutdown` doesn't actually stop WSL if Docker Desktop or any terminal/VS Code window still holds a session open. Must close everything (incl. Docker Desktop) first, **then** shutdown.
4. Even with the correct IP, the JSON config (`packer/proxmox/ubuntu-2404.json`) had no exposed `http_bind_address`/`http_ip` variable (confirmed via `grep -r "http_bind_address\|http_ip\|HTTPIP" packer/`) — unlike the OVA/vSphere provider, the Proxmox builder doesn't expose this as an overridable var. Manually edited `boot_command_prefix` in that file, replacing `{{ .HTTPIP }}` with the literal LAN IP.
5. **Still failed** even after the IP fix. Verified with `curl` that the HTTP server (`meta-data`, `user-data`) was reachable *from WSL itself* — fine. But `curl` from the **Proxmox host** to the WSL IP timed out completely (`Trying... ^C`).
   - **Conclusion: WSL2 mirrored mode allows outbound connections from WSL to the LAN, but does not reliably accept inbound connections from the LAN back into WSL** (Windows Firewall / mirrored-mode limitation). This only breaks scenarios where something *external* needs to reach WSL (like the build VM fetching cloud-init data) — not the other way around.

**Final fix:** abandoned WSL for this step entirely. Ran Image Builder on a **separate native Ubuntu machine** instead. Build completed cleanly, no IP edits needed — Packer auto-detected the correct IP because there's no NAT/firewall layer to fight.

**Takeaway for future provisioning steps (clusterctl/kind):** this WSL limitation should *not* resurface for clusterctl/kind/CAPI usage, because those flows are WSL-initiated (outbound: WSL → Proxmox API, WSL → VM SSH). The inbound-connection problem is specific to Image Builder's "VM calls back into Packer's HTTP server" pattern. Mirrored mode is not known to be required for CAPI itself — it just happened to be tried as a (partial, ultimately insufficient) fix for the Image Builder issue.

</details>

### Final working build command (run on native Ubuntu)

```bash
source ~/image-builder-venv/bin/activate
cd ~/image-builder/images/capi

export PROXMOX_URL="https://192.168.0.100:8006/api2/json"
export PROXMOX_USERNAME='capi@pve!capitoken'
export PROXMOX_TOKEN="9d4f1434-3d92-40bd-9284-746ea7ade180"
export PROXMOX_NODE="pve"
export PROXMOX_ISO_POOL="local"
export PROXMOX_BRIDGE="vmbr0"
export PROXMOX_STORAGE_POOL="local-lvm"
export PACKER_FLAGS="--var 'disk_format=raw'"

make build-proxmox-ubuntu-2404
```
This process may take 15-60 minutes — Packer downloads the ISO, uploads it to Proxmox, creates the VM, installs the necessary packages (containerd, kubeadm, kubelet, kubectl) into it, and then converts the VM to a template.

> **Required before provisioning — CPU type:** after the build completes, open the template in Proxmox and set `Hardware → Processors → Edit → CPU type` to `host`. Do this before CAPI clones any VMs. Otherwise, workloads such as recent Calico versions may fail because the default `kvm64` CPU exposes only x86-64-v1 instructions. See Issue 7 for the symptoms and recovery details.

> **Multiple Proxmox nodes with local storage:** a template stored on `local`/`local-lvm` is available only on the node where it was built. If CAPI may place VMs on multiple nodes and you do not use shared storage, run the Image Builder command once per node, changing `PROXMOX_NODE` each time (for example, `pve1`, then `pve2`). Record the template VMID created on each node; the generated CAPI YAML must reference the matching `sourceNode` and `templateID`. Creating the template only on one node causes the clone failure documented after Step 8.

### raw vs qcow2 — why the format matters

The end result is a VM template, not an image file you download (no `.qcow2`/`.raw` file lands in your hands — it just lives in Proxmox's storage). But the format still matters because it determines how the template's underlying disk is stored in Proxmox:

**Format must be compatible with the storage type.** Each Proxmox storage type (LVM-thin, ZFS, Directory, Ceph RBD, etc.) supports specific disk formats:

| Storage type | Supported format |
|---|---|
| `local-lvm` (LVM-thin) | `raw` only (no `qcow2`) |
| `local` (directory-based) | `qcow2`, `raw`, `vmdk` |
| ZFS (`local-zfs`) | `raw` only (no `qcow2`) |
| Ceph RBD | `raw` only |

If your storage is `local-lvm` (Proxmox's default in most setups), choosing `qcow2` will make the build fail — LVM-thin doesn't support the file-based disk structure `qcow2` needs; it only supports block-level `raw` disks.


## Step 5: Create the bootstrap cluster (kind)
 
Official doc: https://cluster-api.sigs.k8s.io/user/quick-start
 
CAPI needs an existing K8s cluster to run on (the "management cluster"). Common practice is to spin up a temporary local `kind` cluster, bootstrap CAPI on it, then `clusterctl move` CAPI into the real cluster once it exists — kind gets deleted afterward.
 
```bash
kind create cluster --name capi-bootstrap
kubectl cluster-info
```
 
<details>
<summary><strong>Issue 4 — kubelet refuses to start: cgroup v1 not supported</strong></summary>
Symptom: `kind create cluster` fails during `kubeadm init` with `connection refused` / `context deadline exceeded` on `172.18.0.2:6443`. Tried this both in WSL2 and in native Windows (downloaded `kind.exe`, `kubectl.exe`, `clusterctl.exe` separately) — **same error in both**, which ruled out WSL/NAT as the cause (Docker Desktop uses a WSL2 backend either way, confirmed via `docker logs <container>` showing `Detected virtualization wsl`).
 
Root cause, found via:
```bash
docker exec capi-bootstrap-control-plane systemctl status kubelet
docker exec capi-bootstrap-control-plane journalctl -u kubelet -n 100 --no-pager
```
which showed:
```
kubelet is configured to not run on a host using cgroup v1. cgroup v1 support is unsupported and will be removed in a future release
```
The `kindest/node:v1.36.1` image flat out refuses to boot on cgroup v1. The WSL2 distro's kernel was still using cgroup v1.
 
**Fix:** force the WSL2 kernel to use cgroup v2.
```powershell
notepad $env:USERPROFILE\.wslconfig
```
```ini
[wsl2]
kernelCommandLine = cgroup_no_v1=all
```
Close Docker Desktop and all terminal/WSL windows first, then:
```powershell
wsl --shutdown
```
Reopen Docker Desktop, verify with `cat /sys/fs/cgroup/cgroup.controllers` (should print a list, not error) — then `kind create cluster` worked immediately, all pods `Running`.
 
 
</details>
Verify:
```bash
kubectl get pods -A
```
All pods (`etcd`, `kube-apiserver`, `coredns`, etc.) should be `Running`.
 
---
 
## Step 6: Initialize the management cluster with clusterctl
 
Proxmox provider doc: https://github.com/ionos-cloud/cluster-api-provider-proxmox/blob/main/docs/Usage.md
 
```bash
export PROXMOX_URL="https://192.168.0.100:8006"
export PROXMOX_TOKEN='capi@pve!capitoken'
export PROXMOX_SECRET="9d4f1434-3d92-40bd-9284-746ea7ade180"
 
clusterctl init --infrastructure proxmox --ipam in-cluster
```
 
> ⚠️ **Naming gotcha:** Image Builder and CAPMOX use *different* env var names for the same credentials. Image Builder: `PROXMOX_USERNAME` (token ID) + `PROXMOX_TOKEN` (secret). CAPMOX: `PROXMOX_TOKEN` (token ID) + `PROXMOX_SECRET` (secret). Same values, different variable names — mixing them up across terminal sessions causes auth failures. Use a fresh terminal for this step.
 
The `--ipam in-cluster` flag installs a controller that statically assigns IPs to new VMs from a defined pool (`NODE_IP_RANGES`) — workload cluster nodes need stable IPs, can't rely on DHCP for this.
 
Verify:
```bash
kubectl get pods -A
```
Should see `capi-system`, `capi-kubeadm-bootstrap-system`, `capi-kubeadm-control-plane-system`, `capmox-system`, `capi-ipam-in-cluster-system`, all `1/1 Running` (IPAM pod may take a minute longer to become ready).
 
---
 
## Step 7: Set workload cluster variables
 
```bash
# SSH key (used to access the VMs)
ssh-keygen -t ed25519 -C "capi-cluster"
```
 
```bash
# --- Proxmox auth (CAPMOX naming, see gotcha above) ---
export PROXMOX_URL="https://192.168.0.100:8006"
export PROXMOX_TOKEN='capi@pve!capitoken'
export PROXMOX_SECRET="9d4f1434-3d92-40bd-9284-746ea7ade180"
 
# --- Template info ---
export PROXMOX_SOURCENODE="pve"
export TEMPLATE_VMID="100"
 
# --- Which Proxmox nodes VMs can land on ---
export ALLOWED_NODES="[pve1,pve2]"
 
# --- SSH ---
export VM_SSH_KEYS="$(cat ~/.ssh/id_ed25519.pub)"
 
# --- Networking ---
export CONTROL_PLANE_ENDPOINT_IP="192.168.0.150"      # see kube-vip note below
export NODE_IP_RANGES="[192.168.0.151-192.168.0.160]" # static IP pool for nodes
export GATEWAY="192.168.0.1"                          # LAN router IP, not the Proxmox host IP
export IP_PREFIX="24"
export DNS_SERVERS="[8.8.8.8,8.8.4.4]"
export BRIDGE="vmbr0"
 
# --- VM sizing (small for first test) ---
export BOOT_VOLUME_DEVICE="scsi0"   # matches the disk controller Image Builder used (virtio-scsi-pci)
export BOOT_VOLUME_SIZE="50"
export NUM_SOCKETS="1"
export NUM_CORES="2"
export MEMORY_MIB="4096"
 
# --- Required feature flags ---
export EXP_CLUSTER_RESOURCE_SET="true"
export CLUSTER_TOPOLOGY="true"
```
 
**kube-vip / `CONTROL_PLANE_ENDPOINT_IP` note:** this is *not* a MetalLB-style pool. It's a separate, fixed IP (outside `NODE_IP_RANGES`, same subnet) that `kube-vip` floats across whichever control-plane node is currently healthy. `kubectl`/`kubeconfig` always points here — if a control-plane node dies, kube-vip moves this IP to a surviving one, transparently. MetalLB (planned separately, later) is unrelated — that's for exposing in-cluster `Service: LoadBalancer` apps, not for reaching the K8s API itself.
 
---
 
## Step 8: Generate and apply the workload cluster
 
```bash
clusterctl generate cluster homelab \
  --infrastructure proxmox \
  --kubernetes-version v1.36.1 \
  --control-plane-machine-count 1 \
  --worker-machine-count 1 > cluster.yaml
 
```
CAPMOX's default cluster template also hardcodes pod subnet `192.168.0.0/16` in the generated `cluster.yaml`. This place need the fix:
 
**In the generated `cluster.yaml`** (right after `clusterctl generate cluster`, before `kubectl apply`):
```bash
sed -i 's/192.168.0.0\/16/10.244.0.0\/16/g' cluster.yaml
```

### Give control-plane and worker VMs different RAM sizes

Before `kubectl apply`, edit the generated YAML. It contains separate `ProxmoxMachineTemplate` resources for the control plane and workers, so each template can use a different `memoryMiB` value:

```yaml
# ProxmoxMachineTemplate: homelab-control-plane
spec:
  template:
    spec:
      memoryMiB: 4096
---
# ProxmoxMachineTemplate: homelab-worker-pve1
spec:
  template:
    spec:
      memoryMiB: 10240 # 10 GiB
---
# When using a node-specific worker template on pve2
# ProxmoxMachineTemplate: homelab-worker-pve2
spec:
  template:
    spec:
      memoryMiB: 7168 # 7 GiB
```

The node-specific worker template pattern above is taken from [`cluster3.yaml`](../cluster3.yaml). When templates live on local storage, also ensure each machine template has the correct `sourceNode` and that node's `templateID`.

### Enable signed kubelet serving certificates before applying the YAML

Before applying the generated CAPI YAML, configure both kubelet serving-certificate requests and API server verification. The serving CSRs remain `Pending` until `kubelet-csr-approver` is installed after Calico. During this bootstrap window, normal API operations such as `kubectl get` and `apply` work, but kubelet-proxied operations such as `logs` and `exec` may fail.

 CAPI supplies the `KubeletConfiguration` through kubeadm's patch directory, as described in the Cluster API [Kubelet Configuration](https://cluster-api.sigs.k8s.io/tasks/bootstrap/kubeadm-bootstrap/kubelet-config) documentation.

<details>
<summary><strong>How kubelet serving TLS, CSR approval and API server verification work</strong></summary>

Kubelet uses two different certificate paths:

- `kubernetes.io/kube-apiserver-client-kubelet` is the kubelet's **client** certificate for `kubelet → API server`. Kubernetes approves this automatically so the node can join the cluster.
- `kubernetes.io/kubelet-serving` is the kubelet's **serving** certificate for clients connecting to `kubelet:10250`, including Metrics Server and the API server proxy used by `kubectl logs`/`exec`. Kubernetes does not approve this signer automatically.

By default, kubeadm does not leave kubelet without a serving certificate: kubelet uses a **self-signed** serving certificate. The problem is that this certificate is not signed by the Kubernetes CA and may not contain the node IP/DNS SANs required by clients such as Metrics Server.

The complete setup has three independent parts:

1. `serverTLSBootstrap: true` makes kubelet request a Kubernetes-CA-signed serving certificate with its node identity and SANs.
2. `kubelet-csr-approver` validates and approves those serving CSRs; Kubernetes does not approve them automatically.
3. `--kubelet-certificate-authority=/etc/kubernetes/pki/ca.crt` makes the API server verify the certificate presented by kubelet when proxying `logs`, `exec`, `attach` and `port-forward` requests to `kubelet:10250`.

`serverTLSBootstrap: true` does not sign or approve a certificate by itself. It enables kubelet's serving-certificate manager:

1. Kubelet creates a private key locally.
2. Kubelet submits a `kubernetes.io/kubelet-serving` CSR containing its node identity and requested IP/DNS SANs.
3. The CSR stays `Pending` until an authorized approver approves it.
4. The Kubernetes signer issues the certificate.
5. Kubelet stores the issued certificate and key under its certificate directory (normally `/var/lib/kubelet/pki`) and starts presenting that certificate on HTTPS port `10250`.
6. Kubelet requests a replacement before the certificate expires, so the new serving CSR also needs automatic approval.

This serving flow is independent from node join:

| CSR signer | Purpose | Default approval |
|---|---|---|
| `kubernetes.io/kube-apiserver-client-kubelet` | Authenticates `kubelet → API server`; required for the node to join and operate | Automatically approved by the standard kubeadm-installed bootstrap/rotation RBAC and Kubernetes controllers |
| `kubernetes.io/kubelet-serving` | Authenticates the kubelet HTTPS server to clients connecting to `:10250` | Not automatically approved by Kubernetes; handled here by `kubelet-csr-approver` |

Therefore, `kubelet-csr-approver` does **not** approve a node joining the cluster. The node's client CSR can already be `Approved,Issued` and the node can join while its separate serving CSR remains `Pending`.

CSR approval and TLS verification are different operations. The approver determines whether the CA may issue the certificate; the API server flag determines whether that issued certificate must be trusted and valid during each connection. Without the flag, the API server uses HTTPS but does not authenticate the kubelet endpoint, so the connection is vulnerable to a man-in-the-middle attack. A forged certificate alone is not enough to perform the attack: the attacker must also redirect or intercept API server-to-kubelet traffic, for example through ARP, DNS or routing manipulation. Metrics Server performs its own kubelet certificate verification, which is why Metrics Server can reject the old self-signed certificate even while `kubectl logs` still works.

This behavior is documented by Kubernetes under [Enabling signed kubelet serving certificates](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#kubelet-serving-certs) and [API server to kubelet communication](https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/#api-server-to-kubelet).

</details>

Add the following entries to the existing `KubeadmControlPlane.spec.kubeadmConfigSpec`. `clusterConfiguration.apiServer.extraArgs` configures the API server directly; the file patch configures kubelet. Append the new file to the existing `files` list, do not remove the kube-vip files, and keep the existing `nodeRegistration` fields:

```yaml
apiVersion: controlplane.cluster.x-k8s.io/v1beta2
kind: KubeadmControlPlane
metadata:
  name: homelab-control-plane
  namespace: default
spec:
  kubeadmConfigSpec:
    clusterConfiguration:
      apiServer:
        extraArgs:
        - name: kubelet-certificate-authority
          value: /etc/kubernetes/pki/ca.crt
    files:
    # ...keep the existing kube-vip files
    - content: |
        {
          "apiVersion": "kubelet.config.k8s.io/v1beta1",
          "kind": "KubeletConfiguration",
          "serverTLSBootstrap": true
        }
      owner: root:root
      path: /etc/kubernetes/patches/kubeletconfiguration0+strategic.json
      permissions: "0644"
    initConfiguration:
      patches:
        directory: /etc/kubernetes/patches
      nodeRegistration:
        # ...keep the existing control-plane nodeRegistration
    joinConfiguration:
      patches:
        directory: /etc/kubernetes/patches
      nodeRegistration:
        # ...keep the existing control-plane nodeRegistration
```

Add the same file and patch directory to the existing worker `KubeadmConfigTemplate.spec.template.spec`:

```yaml
apiVersion: bootstrap.cluster.x-k8s.io/v1beta2
kind: KubeadmConfigTemplate
metadata:
  name: homelab-worker
  namespace: default
spec:
  template:
    spec:
      files:
      - content: |
          {
            "apiVersion": "kubelet.config.k8s.io/v1beta1",
            "kind": "KubeletConfiguration",
            "serverTLSBootstrap": true
          }
        owner: root:root
        path: /etc/kubernetes/patches/kubeletconfiguration0+strategic.json
        permissions: "0644"
      joinConfiguration:
        patches:
          directory: /etc/kubernetes/patches
        nodeRegistration:
          # ...keep the existing worker nodeRegistration
```

Both resources are required for `serverTLSBootstrap`: `KubeadmControlPlane` configures control-plane kubelets and `KubeadmConfigTemplate` configures worker kubelets. The API server CA flag belongs only in `KubeadmControlPlane` because workers do not run an API server.

```bash
kubectl apply -f cluster.yaml
```
Check status:
```bash
clusterctl describe cluster homelab
kubectl get machines
```

<details>
<summary><strong>Issue 5 — unable to create new VM: 500 can't clone VM to node (VM uses local storage)</strong></summary>

Example errors:

```text
unable to create new vm: 500 can't clone VM to node 'pve2' (VM uses local storage)
node 'pve2' not allowed for this action
```

**Cause:** the Image Builder template was created only on `pve1`, while CAPI tried to create a VM on `pve2`. A template on `local`/`local-lvm` storage cannot be cloned by another Proxmox node.

**Fix:** build the template on every target node as described in Step 3 (or move it to shared storage). For local templates, use separate node-specific `ProxmoxMachineTemplate` resources and set each resource's `sourceNode` and `templateID` to the template that exists on that node. Then regenerate/reapply the cluster resources.

</details>

<details>
<summary><strong>Issue 6 — Machine stays Unknown while waiting for its providerID Node</strong></summary>

Sometimes Proxmox successfully clones a VM, but the VM does not finish booting or join Kubernetes. `clusterctl describe cluster` may show:

```text
Machine/homelab-workers-pve2-4jwgz-qw5th 1 0 0 1 Unknown ReadyUnknown
* NodeHealthy: Waiting for a Node with spec.providerID
  proxmox://c880f573-13d9-4e7a-84c1-1d5ccedd2874 to exist
```

Delete the stuck `Machine` object from the management cluster:

```bash
kubectl delete machine homelab-workers-pve2-4jwgz-qw5th -n default
```

The owning `MachineDeployment` notices that a replica is missing and CAPI automatically creates a replacement Machine/VM. Wait for the replacement to boot and join the cluster, then check it again with `clusterctl describe cluster` and `kubectl get machines`.

</details>

 
## Step 9: Get kubeconfig, install Calico and finish kubelet TLS
 
```bash
clusterctl get kubeconfig homelab > homelab.kubeconfig
```
 
Nodes will show `NotReady` until a CNI is installed — this is expected, not an error:
```bash
KUBECONFIG=homelab.kubeconfig kubectl get nodes
```

### Install Calico

The homelab LAN uses `192.168.0.0/24`, which overlaps with Calico's default `192.168.0.0/16` pod CIDR. Download the manifest and change its pod CIDR to the same non-overlapping range configured in `cluster.yaml` before applying it:

```bash
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/calico.yaml

# Uncomment CALICO_IPV4POOL_CIDR
sed -i 's|.*- name: CALICO_IPV4POOL_CIDR.*|            - name: CALICO_IPV4POOL_CIDR|' calico.yaml

# Use the same pod CIDR configured in cluster.yaml
sed -i 's|.*value: "192.168.0.0/16".*|              value: "10.244.0.0/16"|' calico.yaml

kubectl --kubeconfig=homelab.kubeconfig apply -f calico.yaml
```

> Always use `--kubeconfig=homelab.kubeconfig` (or `KUBECONFIG=homelab.kubeconfig kubectl ...`) for anything targeting the workload cluster. A bare `kubectl` command still points at the kind management cluster.

Wait, then verify:

```bash
KUBECONFIG=homelab.kubeconfig kubectl get pods -n kube-system
KUBECONFIG=homelab.kubeconfig kubectl get nodes
```

Nodes should flip to `Ready` once Calico pods are `Running` on all of them.

### Install `kubelet-csr-approver`

The bootstrap patch makes every kubelet request a CA-signed serving certificate, but Kubernetes deliberately leaves `kubernetes.io/kubelet-serving` CSRs `Pending`. Manual approval is not a permanent solution because replacement nodes and certificate rotation create new CSRs. Install [`kubelet-csr-approver`](https://github.com/postfinance/kubelet-csr-approver) after Calico so these requests are validated and approved automatically:

> Without `kubelet-csr-approver`, every serving CSR would have to be reviewed and approved manually with `kubectl certificate approve <csr-name>`, including CSRs created by new/replacement nodes and certificate rotation.

```bash
helm repo add kubelet-csr-approver https://postfinance.github.io/kubelet-csr-approver
helm repo update

helm install kubelet-csr-approver kubelet-csr-approver/kubelet-csr-approver \
  --namespace kube-system \
  --set providerRegex='^homelab.*$' \
  --set providerIpPrefixes='192.168.0.0/24' \
  --set bypassDnsResolution='true'
```

`providerRegex` restricts allowed node hostnames, while `providerIpPrefixes` restricts the IP SANs that may be signed. `bypassDnsResolution=true` is required here because the homelab does not provide DNS records for individual node names. The approver still performs its other requester, Common Name, hostname, signer and SAN checks.

Wait for the controller and verify that the serving CSRs change to `Approved,Issued`:

```bash
kubectl rollout status deployment/kubelet-csr-approver -n kube-system
kubectl get csr
```

The expected distinction is:

```text
kubernetes.io/kube-apiserver-client-kubelet   Approved,Issued  # automatic node join
kubernetes.io/kubelet-serving                 Approved,Issued  # kubelet-csr-approver
```

Old duplicate serving CSRs may remain visible because CSRs are historical API objects. What matters is that each current node has an approved serving CSR and starts presenting the signed certificate.

The API server verification flag was already installed from `cluster.yaml`; the approver does not add or modify it. Once the serving CSRs are approved and kubelet starts presenting the CA-signed certificates, verify the API server-to-kubelet path:

```bash
kubectl get --raw "/api/v1/nodes/$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')/proxy/healthz"
kubectl logs -n kube-system deployment/kubelet-csr-approver --tail=20
```

<details>
<summary><strong>Issue 7 — nodes stuck NotReady: Calico init containers crash with "v2 microarchitecture" error</strong></summary>

After CNI install (see Step 9), `calico-node` pods stuck in `Init:Error`:
```bash
kubectl logs calico-node-xxxxx -n kube-system -c upgrade-ipam
# This program can only be run on AMD64 processors with v2 microarchitecture support.
```
 
Root cause: the template VM (and any VM cloned from it) had **CPU type `kvm64`** in Proxmox — Proxmox's old default virtual CPU model, which only exposes **x86-64-v1** instruction set to the guest, regardless of what the underlying physical CPU (i5-6600T, which supports v3) actually supports. The Calico `v3.32.0` image requires v2 minimum, so it refuses to run.
 
Quick test fix (per-VM, not persistent): stop the VM → `Hardware → Processors → Edit → CPU type: host` → start. Calico pods went `Running` immediately.
 
**Permanent fix:** fix the *template* itself, not each cloned VM (since every new node is cloned from it and would otherwise hit this every time):

**Hardware → Processors → Edit → CPU type: `host`**

No `cpuType` field exists in CAPMOX's `ProxmoxMachineTemplate` schema (checked) — this has to be fixed at the template level in Proxmox, not via CAPI YAML.
 
After fixing the template: `kubectl delete cluster homelab`, regenerate + reapply → new VMs come up with `host` CPU type from the start, Calico goes `Running` without manual intervention.
 
</details>
---

<details>
<summary><strong>Issue 8 — pods unreachable from the LAN: Calico's default pod CIDR collides with the home network</strong></summary>

**Root cause:** Calico's official manifest (`calico.yaml`) defaults to pod CIDR `192.168.0.0/16`. The homelab LAN is `192.168.0.0/24` (Proxmox host: `192.168.0.100`), which falls *inside* that `/16` range.
 
What goes wrong: a pod gets an IP like `192.168.204.135`. When that pod tries to reach `192.168.0.100` (the Proxmox host), Calico checks its pool — `192.168.0.100` falls inside `192.168.0.0/16`, so Calico assumes the destination is *another pod in the cluster* and skips IP masquerade (outgoing NAT). The packet leaves the VM with its raw pod-internal source IP (`192.168.204.135`). The Proxmox host receives it, tries to reply, but has no route back to `192.168.204.x` (that's the cluster's internal network, not the LAN) — reply gets dropped by the home router. Result: connections from pods to anything on the LAN hang and time out indefinitely, with no obvious error.
 
**Fix:** use a pod CIDR that cannot overlap with the LAN, such as `10.244.0.0/16`. Both `cluster.yaml` and the Calico manifest must use the same value. Step 8 patches `cluster.yaml`; the Calico installation commands above patch `calico.yaml` before it is applied.
 
**Takeaway:** always check the LAN subnet against the CNI's default pod CIDR before installing — this only surfaces as a silent timeout, not a clear error, so it's easy to lose time on.
 
</details>

## Step 10: Pivot — move CAPI from kind into the real cluster
 
Once the workload cluster (`homelab`) is healthy (CNI installed, nodes `Ready`), CAPI itself can be migrated off the temporary kind cluster and into the real one, so kind can be deleted and the cluster becomes self-managing.
 
### Step 10.1: Re-set env vars (fresh terminal)
 
```bash
export PROXMOX_URL="https://192.168.0.100:8006"
export PROXMOX_TOKEN='capi@pve!capitoken'
export PROXMOX_SECRET="9d4f1434-3d92-40bd-9284-746ea7ade180"
 
export EXP_CLUSTER_RESOURCE_SET="true"
export CLUSTER_TOPOLOGY="true"
```
 
### Step 10.2: Install CAPI provider components into the target (workload) cluster
 
`clusterctl move` only *transfers* CAPI objects — it doesn't install CAPI itself on the target. The target needs its own CAPI install first:
 
```bash
clusterctl init --kubeconfig=homelab.kubeconfig --infrastructure proxmox --ipam in-cluster
 
# wait until capi + capmox pods are 1/1
KUBECONFIG=homelab.kubeconfig kubectl get pods -A -w
```
 
### Step 10.3: Move
 
Run from the context still pointing at the **source** (kind):
```bash
clusterctl move --to-kubeconfig=homelab.kubeconfig
```
 
### Step 10.4: Verify the pivot
 
**1. Source (kind) should now be empty:**
```bash
kubectl get clusters,machines,proxmoxclusters -A
```
 
**2. Target (`homelab`) should now hold the objects:**
```bash
KUBECONFIG=homelab.kubeconfig kubectl get clusters,machines,proxmoxclusters -A
```
 
**3. Ultimate test — delete kind, confirm nothing breaks:**
```bash
kind delete cluster --name capi-bootstrap
KUBECONFIG=homelab.kubeconfig kubectl get clusters,machines,proxmoxclusters -A
```
If this still reports the cluster/machines correctly after kind is gone, the pivot succeeded — `homelab` is now self-hosting its own CAPI and can provision/manage further clusters without needing kind again.
