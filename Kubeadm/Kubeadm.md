# Building a Kubernetes Cluster with kubeadm on Proxmox

This guide builds Kubernetes manually on two Ubuntu VMs with `kubeadm`. The goal is to expose the host preparation, container runtime, PKI, static Pods, CNI, and node-join steps that higher-level provisioning tools normally automate.

> This is a learning cluster. A single control-plane node means that the control plane and stacked etcd have no high availability. Do not treat this topology as a production design.

## Topology

| VM | Proxmox node | Role | vCPU | RAM | Example IP |
|---|---|---|---:|---:|---|
| `k8s-master-1` | `pve2` | control plane | 2 | 4 GB | `192.168.0.161` |
| `k8s-worker-1` | `pve2` | worker | 2 | 4 GB | `192.168.0.162` |

Kubernetes networks:

| Network | CIDR |
|---|---|
| Node/LAN network | `192.168.0.0/24` |
| Pod network | `10.244.0.0/16` |
| Service network | `10.96.0.0/12` |

Before starting:

- Verify that `192.168.0.161` and `192.168.0.162` are unused.
- Keep these addresses outside the DHCP pool or reserve them by MAC address on the router.
- Ensure that the Pod and Service CIDRs do not overlap each other or the node/LAN network.
- Independent clusters may use the same Pod and Service CIDRs as long as their overlay networks are not routed together. Node IP addresses must still be unique on the LAN.

## What kubeadm does—and does not do

`kubeadm` is not a fully automated infrastructure or operating-system installer. A more accurate description is:

> kubeadm bootstraps the Kubernetes control plane and nodes on already prepared Linux machines, using secure defaults.

Kubernetes documents kubeadm as its official cluster bootstrap tool for administrators who manage their own infrastructure.

### Tasks completed before kubeadm

The administrator still prepares the two VMs:

- Install the operating system.
- Configure hostname and stable IP addressing.
- Ensure that nodes can reach each other.
- Configure the required firewall ports.
- Decide how swap will be handled.
- Load the required kernel modules and configure sysctl.
- Install a CRI-compatible runtime such as containerd or CRI-O.
- Configure the container runtime's cgroup driver.
- Install kubelet, kubeadm, and kubectl.
- Pin or otherwise manage Kubernetes package versions.
- Select and install a CNI.

Kubeadm does not install or manage the container runtime, kubelet, or kubectl packages.

### Tasks completed by `kubeadm init`

At a high level, `kubeadm init`:

- Creates the Kubernetes certificate authorities.
- Creates the API server certificate.
- Creates the API server-to-etcd client certificate.
- Creates etcd server, peer, and health-check client certificates.
- Creates the front-proxy CA and client certificate.
- Creates ServiceAccount signing keys.
- Creates kubeconfig files such as `admin.conf`, `scheduler.conf`, and `controller-manager.conf`.
- Writes the local etcd static Pod manifest.
- Writes the API server, scheduler, and controller-manager static Pod manifests.
- Creates the bootstrap token and kubelet TLS bootstrap configuration.
- Installs CoreDNS and kube-proxy.
- Prints the `kubeadm join` command used by workers.

These operations correspond to individual `kubeadm init phase` steps, which are examined later in this guide.

---

## 1. Upload the Ubuntu Server 24.04 ISO to Proxmox

Download **Ubuntu Server 24.04 LTS** from the official page:

<https://ubuntu.com/download/server>

In the Proxmox web interface:

1. Open `pve2 → local (pve2) → ISO Images`.
2. Select **Upload** for a local ISO or **Download from URL** for a direct link.
3. Wait until the ISO appears in the storage content list.

> ISO files must be stored on storage that supports the `ISO image` content type. In a default installation this is normally `local`, not `local-lvm`.

---

## 2. Create the virtual machines

Create both VMs independently on pve2.

Recommended settings:

- OS: the uploaded Ubuntu Server 24.04 ISO
- Machine type: default or `q35`
- BIOS: SeaBIOS is sufficient; UEFI also works
- Disk controller: VirtIO SCSI
- Disk: 32–40 GB
- CPU: 1 socket, 2 cores, CPU type `host`
- RAM: 4096 MiB
- Network model: VirtIO
- Bridge: `vmbr0`
- QEMU Guest Agent: enabled

During Ubuntu installation:

- Enable **OpenSSH server**.
- Create the initial local user and password.
- Leaving IPv4 on DHCP is acceptable during installation. The permanent addresses are configured after the first boot.

### Finish the installation and remove the ISO

At the end of the installation:

1. Wait until the installer logs have finished.
2. Select **Reboot Now** and press `Enter`.
3. When the console displays the prompt asking you to remove the installation medium, press `Enter` again. This allows the VM to continue rebooting from its installed virtual disk instead of the Ubuntu ISO.

If the VM opens the Ubuntu installer again, stop the VM and detach the ISO in Proxmox under `VM → Hardware → CD/DVD Drive`, or set the drive to **Do not use any media**. Then start the VM again.

### Find the temporary DHCP address and connect over SSH

Open each VM's Proxmox console and find the address assigned by DHCP:

```bash
hostname -I
```

Use the temporary `192.168.0.x` LAN address from that output to connect from the administration machine. Ignore loopback, container, or IPv6 addresses if more than one address is shown.

```bash
ssh <UBUNTU_USER>@<DHCP_IP>
```

The password created during the Ubuntu installation is used for this first connection.

### Configure the hostname and a static IPv4 address

Set the hostname before joining Kubernetes.

On the control plane:

```bash
sudo hostnamectl set-hostname k8s-master-1
```

On the worker:

```bash
sudo hostnamectl set-hostname k8s-worker-1
```

Find the network interface name:

```bash
ip -br link
```

Inspect the installer-generated Netplan files:

```bash
sudo ls -l /etc/netplan
sudo cat /etc/netplan/*.yaml
```

The interface on these VMs is `enp6s18`. Create `/etc/netplan/99-k8s-static.yaml` on the control-plane VM. The `99-` prefix makes this override load after typical installer-generated files:

```bash
sudo nano /etc/netplan/99-k8s-static.yaml
```

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp6s18:
      dhcp4: false
      addresses:
        - 192.168.0.161/24
      routes:
        - to: default
          via: 192.168.0.1
      nameservers:
        addresses:
          - 192.168.0.1
          - 1.1.1.1
```

Use `192.168.0.162/24` in `addresses` on the worker. Adjust the interface, gateway, and DNS values for the actual network.

```bash
sudo chmod 600 /etc/netplan/99-k8s-static.yaml
sudo netplan generate
sudo netplan try
sudo netplan apply
```

> Applying Netplan over SSH may interrupt the session. Keep the Proxmox console open. `netplan try` automatically rolls back the change unless it is confirmed in time.

The old SSH connection may close when the address changes. Reconnect using the new static address:

```bash
ssh <UBUNTU_USER>@192.168.0.161
ssh <UBUNTU_USER>@192.168.0.162
```

Verify each VM:

```bash
hostnamectl
ip -br address
ip route
ping -c 3 192.168.0.1
```

### Install the SSH public key

Create a dedicated key on the administration machine. The `-f` option gives the private key an explicit path; the matching public key is created as `~/.ssh/kubeadm_lab_ed25519.pub`:

```bash
ssh-keygen \
  -t ed25519 \
  -f "$HOME/.ssh/kubeadm_lab_ed25519" \
  -C "kubeadm-lab"
```

If that file already exists, do not overwrite it unless the old key is no longer needed.

Copy that exact public key to both VMs after their permanent addresses are configured:

```bash
ssh-copy-id \
  -i "$HOME/.ssh/kubeadm_lab_ed25519.pub" \
  <UBUNTU_USER>@192.168.0.161

ssh-copy-id \
  -i "$HOME/.ssh/kubeadm_lab_ed25519.pub" \
  <UBUNTU_USER>@192.168.0.162
```

Verify key-based access:

```bash
ssh -i "$HOME/.ssh/kubeadm_lab_ed25519" \
  <UBUNTU_USER>@192.168.0.161

ssh -i "$HOME/.ssh/kubeadm_lab_ed25519" \
  <UBUNTU_USER>@192.168.0.162
```

Do not disable password authentication until key-based login has been tested successfully from a second terminal.

### Expand the LVM root filesystem to use the virtual disk

First compare the virtual disk, root logical volume, and mounted root filesystem sizes on both VMs:

```bash
lsblk
df -h /
```

If `lsblk` shows a 32 GB disk but the LVM device mounted at `/` and the `df -h /` filesystem are only about 15 GB, the root logical volume is not using all available disk space. If the root LV already occupies nearly the full disk, skip the expansion.

When the unused-space pattern is present, run:

```bash
sudo lvextend \
  --extents +100%FREE \
  --resizefs \
  /dev/ubuntu-vg/ubuntu-lv
```

This assigns all unallocated space in `ubuntu-vg` to the root logical volume. `--resizefs` also grows the ext4 filesystem, so a separate `resize2fs` command is not required.

<details>
<summary><strong>Why the 32 GB virtual disk initially appeared as a 15 GB root filesystem</strong></summary>

Ubuntu's guided LVM installation may create a physical volume covering almost the entire virtual disk while allocating only part of the volume group to the root logical volume:

```text
sda                         32G disk
└─sda3                      30G part
  └─ubuntu--vg-ubuntu--lv   15G lvm  /
```

The remaining space is not lost:

- `/dev/sda3` already occupies approximately 30 GB.
- `/dev/sda3` is the LVM physical volume.
- `ubuntu-vg` is the volume group containing that space.
- `ubuntu-lv` is the 15 GB logical volume mounted at `/`.
- The rest remains unallocated inside `ubuntu-vg`.

The layout can be checked before expansion with:

```bash
lsblk
df -h /
findmnt -no SOURCE /
sudo pvs
sudo vgs
sudo lvs -o lv_path,lv_size,vg_name
```

The command in this guide is applicable when `/` is mounted from `/dev/mapper/ubuntu--vg-ubuntu--lv` or `/dev/ubuntu-vg/ubuntu-lv`, and `sudo vgs` reports free space in `VFree`.

Verify the result afterwards:

```bash
sudo vgs
sudo lvs -o lv_path,lv_size,vg_name
lsblk
df -h /
```

The root filesystem should use nearly all available space after accounting for the boot partition and filesystem/LVM overhead.

Do not use the command on a non-LVM installation or when the root logical volume has a different path. Derive the actual path from `findmnt` and `sudo lvs`. Important data should be backed up before changing storage.

Ubuntu reference: <https://documentation.ubuntu.com/server/how-to/storage/manage-logical-volumes/>

</details>

---

## 3. Prepare both VMs for kubeadm

Run every subsection below on **both VMs**.

### 3.1 Update the OS and install basic tools

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt install -y \
  apt-transport-https \
  ca-certificates \
  conntrack \
  curl \
  gpg \
  socat
```

<details>
<summary><strong>Install the optional QEMU Guest Agent package</strong></summary>

QEMU Guest Agent is not required by Kubernetes. It is a Proxmox integration used to retrieve guest information, request clean shutdowns, and coordinate operations such as filesystem freeze/thaw for snapshots and backups.

Because **QEMU Guest Agent** was selected while creating the VM, install the matching package inside Ubuntu:

```bash
sudo apt install -y qemu-guest-agent
sudo systemctl start qemu-guest-agent
systemctl is-active qemu-guest-agent
```

The last command should return `active`.

Do not run `systemctl enable qemu-guest-agent`. On Ubuntu this is a static unit without an `[Install]` section; it starts when the Proxmox-provided virtio device becomes available.

If the Proxmox option is enabled after the VM is already running, a normal guest `reboot` is not sufficient. Shut down the VM until Proxmox shows it as **Stopped**, then use **Start** so the virtio-serial device is added on a fresh VM process.

The default virtio-serial path is documented in the Ubuntu [`qemu-ga` manpage](https://manpages.ubuntu.com/manpages/noble/man8/qemu-ga.8.html).

</details>

Reboot both VMs after the OS and package updates:

```bash
sudo reboot
```

### 3.2 Disable swap

Kubelet's default behavior is to refuse to start when swap is detected. Keeping swap disabled makes resource requests, limits, eviction decisions, and node memory accounting predictable.

Recent Kubernetes releases can use swap, but it must be enabled deliberately in the kubelet configuration. Disabling it is the simpler choice for this first learning cluster.

```bash
sudo swapoff -a
swapon --show
```

To make the change persistent, comment out the swap entry in `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

Verify the file and the active swap devices:

```bash
sudo mount -a
swapon --show
```

The second command should produce no output.

### 3.3 Load kernel modules and configure sysctl

Pod traffic is routed through Kubernetes nodes. `net.ipv4.ip_forward = 1` allows Linux to forward packets between interfaces, which is necessary for Pod-to-Pod and Pod-to-external-network communication.

`br_netfilter` allows traffic crossing a Linux bridge to pass through netfilter/iptables processing. Whether every bridge setting is required can depend on the selected CNI; the settings below provide the conventional baseline used for this Calico installation.

The `overlay` module supports OverlayFS, which containerd commonly uses for container image and writable layers.

```bash
sudo tee /etc/modules-load.d/k8s.conf >/dev/null <<'EOF'
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

```bash
sudo tee /etc/sysctl.d/k8s.conf >/dev/null <<'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

Verify the result:

```bash
lsmod | grep -E 'overlay|br_netfilter'
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv4.ip_forward
```

### 3.4 Install containerd and use the systemd cgroup driver

Kubelet does not start containers directly. It communicates with a container runtime through the Container Runtime Interface (CRI):

```text
kubelet
   │
   │ CRI over a Unix socket
   ▼
containerd
   │
   ▼
runc
   │
   ▼
Linux namespaces + cgroups
```

Containerd manages container images and lifecycle, while `runc` creates the Linux processes, namespaces, and cgroups.

Cgroups enforce resource controls such as CPU, memory, process count, and I/O. Kubelet and containerd manage the same cgroup hierarchy, so their drivers must agree:

```text
kubelet    → systemd
containerd → systemd
```

Kubeadm configures kubelet to use `systemd` by default. Containerd is configured explicitly below; mixing `systemd` and `cgroupfs` can make the node unstable.

```bash
sudo apt update
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
```

Enable `SystemdCgroup`:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml

sudo systemctl enable --now containerd
sudo systemctl restart containerd
```

Verify containerd:

```bash
grep -n 'SystemdCgroup' /etc/containerd/config.toml
grep -n 'disabled_plugins' /etc/containerd/config.toml
sudo systemctl --no-pager --full status containerd
sudo ctr version
```

Also ensure that `cri` is not present in containerd's `disabled_plugins` list.

Install `crictl` on both nodes. It talks directly to the CRI runtime and is used
later to inspect Pod sandboxes, containers, images, and runtime state without
going through the Kubernetes API:

```bash
CRICTL_VERSION="v1.36.0"
curl -LO \
  "https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-amd64.tar.gz"
sudo tar -xzf "crictl-${CRICTL_VERSION}-linux-amd64.tar.gz" \
  -C /usr/local/bin
rm "crictl-${CRICTL_VERSION}-linux-amd64.tar.gz"
```

The prepared [`crictl.yaml`](./crictl.yaml) in the local repository configures
the containerd CRI socket. From the repository root on the workstation, copy it
to both nodes:

```bash
scp Kubeadm/crictl.yaml \
  ubuntu@192.168.0.161:/home/ubuntu/crictl.yaml
scp Kubeadm/crictl.yaml \
  ubuntu@192.168.0.162:/home/ubuntu/crictl.yaml
```

On each node, install the copied file at the system-wide path and remove the
temporary user-owned copy:

```bash
sudo install -o root -g root -m 0644 \
  /home/ubuntu/crictl.yaml /etc/crictl.yaml
rm /home/ubuntu/crictl.yaml
```

Verify the installation and runtime connection:

```bash
crictl --version
sudo crictl version
sudo crictl info
```

This installation follows the official
[`cri-tools` crictl documentation](https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md#install-crictl).
Kubernetes also documents the containerd endpoint in
[Debugging Kubernetes nodes with crictl](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/).

### 3.5 Install kubelet and kubeadm on every node

Both nodes need:

- `kubelet` to register the node and manage Pods through the container runtime.
- `kubeadm` because the control plane runs `kubeadm init` and the worker runs `kubeadm join`.

Neither node needs `kubectl` in the main workflow. Cluster administration is
performed from the workstation after copying and importing the kubeconfig.

The following repository and keyring setup comes from the official Kubernetes installation documentation:

<https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/>

Run the repository setup on both nodes:

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm
sudo apt-mark hold kubelet kubeadm
sudo systemctl enable --now kubelet
```

The downloaded `Release.key` is the public signing key for the Kubernetes APT repository; it is not a credential or private key. `gpg --dearmor` stores it in APT's binary keyring format, and the `signed-by` option restricts this repository to packages whose metadata is signed by that key.

Verify both nodes:

```bash
kubeadm version -o short
kubelet --version
```

Verify that a compatible `kubectl` is already installed on the workstation:

```bash
kubectl version --client
```

> Kubelet may restart repeatedly before `kubeadm init` or `kubeadm join`. This is expected because `/var/lib/kubelet/config.yaml` does not exist yet.

### 3.6 Verify node identity and connectivity

Hostnames, MAC addresses, and `product_uuid` values must be unique:

```bash
hostname
ip link
sudo cat /sys/class/dmi/id/product_uuid
```

Test both node addresses:

```bash
ping -c 3 192.168.0.161
ping -c 3 192.168.0.162
```

### 3.7 Firewall approach for this lab

For this isolated learning environment on a trusted LAN, the simplest approach is to disable UFW on both VMs:

```bash
sudo ufw status
sudo ufw disable
```

For production, configure the firewall instead of disabling it:

| Node | Kubernetes ports |
|---|---|
| Control plane | TCP `6443`, `2379-2380`, `10250`, `10257`, `10259` |
| Worker | TCP `10250`, `10256` |

Calico may require additional rules depending on its dataplane and encapsulation mode, including BGP `179/TCP` and IP-in-IP protocol `4`.

---

## 4. Take pre-kubeadm snapshots

Shut down both VMs and take these Proxmox snapshots:

- `k8s-master-1-pre-kubeadm`
- `k8s-worker-1-pre-kubeadm`

Take the snapshots after Ubuntu, kernel settings, containerd, and Kubernetes packages are ready, but before initializing or joining the cluster. They provide a clean return point for the later `kubeadm init phase` exercise.

---

## 5. Initialize the control plane with a kubeadm config file

This section transfers the prepared configuration from the workstation to
`k8s-master-1` and then initializes the control plane.

### 5.1 Transfer the kubeadm configuration

On `k8s-master-1`, check the installed kubeadm version:

```bash
kubeadm version -o short
```

The local repository keeps the configuration and manifest files used by this
installation inside the `Kubeadm/` directory:

- [`kubeadm-config.yaml`](./kubeadm-config.yaml)
- [`calico.yaml`](./calico.yaml)
- [`crictl.yaml`](./crictl.yaml)

The repository is not cloned onto the Kubernetes nodes. From the repository
root on the workstation, copy only the kubeadm configuration to the master VM:

```bash
scp Kubeadm/kubeadm-config.yaml \
  ubuntu@192.168.0.161:/home/ubuntu/kubeadm-config.yaml
```

The kubeadm configuration contains:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.0.161
  bindPort: 6443
nodeRegistration:
  name: k8s-master-1
  criSocket: unix:///run/containerd/containerd.sock
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
clusterName: kubeadm-lab
kubernetesVersion: v1.36.3
controlPlaneEndpoint: "192.168.0.161:6443"
networking:
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
  dnsDomain: cluster.local
apiServer:
  certSANs:
    - k8s-master-1
    - 192.168.0.161
  extraArgs:
    - name: kubelet-certificate-authority
      value: /etc/kubernetes/pki/ca.crt
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
serverTLSBootstrap: true
```

For this installation, `kubeadm version -o short` returns `v1.36.3`, so the
configuration uses `kubernetesVersion: v1.36.3`.

<details>
<summary><strong>kubeadm and Kubernetes version skew policy</strong></summary>

The patch versions do not have to be identical for kubeadm compatibility; the
supported skew is primarily defined by the minor version. For example, kubeadm
`1.36.x` can create Kubernetes `1.36.x` or `1.35.x`. Nevertheless, Kubernetes
recommends matching kubeadm with the control-plane components, kube-proxy, and
kubelet. For a new cluster, keeping all of them on `1.36.3` is the clearest
choice. The `kubernetesVersion` field selects the kube-apiserver,
kube-controller-manager, kube-scheduler, and kube-proxy image versions; it does
not change the already-installed kubeadm binary.

See the official
[kubeadm version skew policy](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#version-skew-policy).

</details>

The configuration enables CA-signed kubelet serving certificates from the first boot. Serving CSRs remain `Pending` until the approver is installed after Calico. During this bootstrap window, normal API operations such as `kubectl get` and `apply` work, but kubelet-proxied operations such as `logs` and `exec` may fail certificate validation.

<details>
<summary><strong>How kubelet serving TLS, CSR approval, and API server verification work</strong></summary>

Kubelet uses two different certificate paths:

- `kubernetes.io/kube-apiserver-client-kubelet` is the kubelet **client** certificate for `kubelet → API server`. The standard kubeadm bootstrap and rotation path approves it automatically so that the node can join and operate.
- `kubernetes.io/kubelet-serving` is the kubelet **serving** certificate for clients connecting to `kubelet:10250`, including Metrics Server and the API server proxy used by `kubectl logs`, `exec`, `attach`, and port-forward. Kubernetes does not approve this signer automatically.

Without serving TLS bootstrap, kubelet uses a self-signed serving certificate. It is not signed by the Kubernetes CA and may not contain the node IP/DNS SANs expected by clients such as Metrics Server.

The complete setup has three independent parts:

1. `serverTLSBootstrap: true` makes kubelet request a Kubernetes-CA-signed serving certificate containing its node identity and requested SANs.
2. `kubelet-csr-approver` validates and approves those serving CSRs.
3. `--kubelet-certificate-authority=/etc/kubernetes/pki/ca.crt` makes the API server verify the certificate presented by kubelet on connections to `:10250`.

`serverTLSBootstrap: true` does not approve or sign a certificate by itself:

1. Kubelet creates a private key locally.
2. Kubelet submits a `kubernetes.io/kubelet-serving` CSR.
3. The request remains `Pending` until an authorized approver accepts it.
4. The Kubernetes signer issues the serving certificate.
5. Kubelet stores the certificate and key under its certificate directory, normally `/var/lib/kubelet/pki`, and presents that certificate on port `10250`.
6. Kubelet requests a replacement before expiry; renewed serving CSRs also require approval.

| CSR signer | Purpose | Default approval |
|---|---|---|
| `kubernetes.io/kube-apiserver-client-kubelet` | Authenticates `kubelet → API server`; required for node join and normal node operation | Automatically approved by the kubeadm-installed bootstrap/rotation path |
| `kubernetes.io/kubelet-serving` | Authenticates the kubelet HTTPS server to clients connecting to `:10250` | Not automatically approved by Kubernetes; handled here by `kubelet-csr-approver` |

The serving approver therefore does **not** approve a node joining the cluster. A node's client CSR can already be `Approved,Issued` while its separate serving CSR is still `Pending`.

CSR approval and TLS verification are also separate operations. The approver decides whether the CA may issue the requested certificate. The API server flag decides whether that certificate must be trusted and valid on each API server-to-kubelet connection.

Without the flag, the API server still uses HTTPS but does not authenticate the kubelet endpoint. This is vulnerable to a man-in-the-middle attack if an attacker also redirects or intercepts API server-to-kubelet traffic through mechanisms such as ARP, DNS, or routing manipulation. A forged certificate alone does not redirect the traffic. Metrics Server performs its own kubelet certificate verification, which is why it can reject a self-signed kubelet certificate even while `kubectl logs` still works.

Official references:

- <https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#kubelet-serving-certs>
- <https://kubernetes.io/docs/concepts/architecture/control-plane-node-communication/#api-server-to-kubelet>

</details>

Validate the configuration and inspect the images kubeadm will use:

```bash
# Run on k8s-master-1.
sudo kubeadm config validate \
  --config /home/ubuntu/kubeadm-config.yaml
sudo kubeadm config images list \
  --config /home/ubuntu/kubeadm-config.yaml
```

`kubeadm init` pulls the required images automatically, so a separate image
pull is not required for this online installation.

<details>
<summary><strong>Optional: pre-pull the control-plane images</strong></summary>

Pre-pulling can expose registry or container-runtime problems before
initialization and is required when preparing an offline installation. It does
not change the resulting cluster:

```bash
sudo kubeadm config images pull \
  --config /home/ubuntu/kubeadm-config.yaml
```

This subcommand is documented in the official
[`kubeadm config images pull` reference](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-config/#kubeadm-config-images-pull).
The Kubernetes documentation specifically uses pre-pulling when
[running kubeadm without an Internet connection](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#without-internet-connection).

</details>

### Pod CIDR and Calico IP pool

`networking.podSubnet` describes the Pod address pool to kubeadm and Kubernetes. Calico's IP pool describes the addresses Calico assigns to Pods. Both use `10.244.0.0/16` because they are two declarations of the same Pod network, not two overlapping networks.

`serviceSubnet` is used for virtual ClusterIP Service addresses and must be different from the Pod CIDR.

### 5.2 Initialize the control plane

```bash
# Run on k8s-master-1.
sudo kubeadm init --config /home/ubuntu/kubeadm-config.yaml
```

Save the `kubeadm join ...` command printed at the end. A new join command can also be generated later.

### 5.3 Copy the kubeconfig to the workstation

The rest of this guide runs `kubectl` on the workstation, not on the master VM.
The API endpoint in `admin.conf` is `192.168.0.161:6443`, which is reachable
from the workstation on this LAN.

First stage a user-readable copy on `k8s-master-1`:

```bash
sudo cp /etc/kubernetes/admin.conf \
  /home/ubuntu/kubeadm-lab-admin.conf
sudo chown ubuntu:ubuntu /home/ubuntu/kubeadm-lab-admin.conf
sudo chmod 600 /home/ubuntu/kubeadm-lab-admin.conf
```

On the workstation, copy it into the local kubeconfig directory:

```bash
mkdir -p "$HOME/.kube"
scp ubuntu@192.168.0.161:/home/ubuntu/kubeadm-lab-admin.conf \
  "$HOME/.kube/kubeadm-lab-admin.conf"
chmod 600 "$HOME/.kube/kubeadm-lab-admin.conf"
```

Preserve any existing contexts and merge the new configuration into the
default `$HOME/.kube/config`:

```bash
NEW_CONTEXT="$(
  KUBECONFIG="$HOME/.kube/kubeadm-lab-admin.conf" \
    kubectl config current-context
)"

if [ -f "$HOME/.kube/config" ]; then
  cp "$HOME/.kube/config" \
    "$HOME/.kube/config.backup-$(date +%Y%m%d-%H%M%S)"
  KUBECONFIG="$HOME/.kube/config:$HOME/.kube/kubeadm-lab-admin.conf" \
    kubectl config view --flatten > "$HOME/.kube/config.merged"
  mv "$HOME/.kube/config.merged" "$HOME/.kube/config"
else
  cp "$HOME/.kube/kubeadm-lab-admin.conf" "$HOME/.kube/config"
fi

chmod 600 "$HOME/.kube/config"
kubectl config use-context "$NEW_CONTEXT"
kubectl config get-contexts
rm "$HOME/.kube/kubeadm-lab-admin.conf"
```

After confirming that the copy succeeded, remove the temporary file from the
master VM:

```bash
ssh ubuntu@192.168.0.161 \
  'rm /home/ubuntu/kubeadm-lab-admin.conf'
```

`admin.conf` grants cluster-admin privileges. Keep the local files private and
do not commit them to Git.

<details>
<summary><strong>Alternative: run kubectl as root on the master VM</strong></summary>

If cluster administration is performed in a root shell on `k8s-master-1`, use:

```bash
sudo apt install -y kubectl
export KUBECONFIG=/etc/kubernetes/admin.conf
```

This export applies only to the current shell. The main workflow in this guide
uses the kubeconfig copied to the workstation.

</details>

Check the cluster:

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
```

At this point:

- The control-plane node is expected to be `NotReady`.
- CoreDNS Pods are expected to be `Pending`.
- No CNI/Pod network has been installed yet.

---

## 6. Install the Calico CNI

This lab uses Calico's manifest installation so that its components are easy to inspect. Tigera recommends the Operator for new production clusters. The prepared [`calico.yaml`](./calico.yaml) manifest is stored next to this guide and already uses `10.244.0.0/16`.

Run the following commands on the workstation from the repository root. The
kubeconfig imported in Section 5.3 allows the workstation's `kubectl` to reach
the new API server:

```bash
kubectl cluster-info
kubectl config current-context
```

Calico can automatically detect a kubeadm Pod CIDR. In the prepared manifest,
`CALICO_IPV4POOL_CIDR` is explicitly set to the same CIDR as
`networking.podSubnet`. Inspect the local manifest:

```bash
grep -n -A1 'CALICO_IPV4POOL_CIDR' Kubeadm/calico.yaml
```

The indentation under the container's `env` list must look like this:

```yaml
            - name: CALICO_IPV4POOL_CIDR
              value: "10.244.0.0/16"
```

Apply the manifest:

```bash
kubectl apply -f Kubeadm/calico.yaml
```

Watch the system Pods:

```bash
kubectl get pods -n kube-system -w
```

Then verify:

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

The control-plane node should become `Ready`; Calico and CoreDNS Pods should be `Running`.

---

## 7. Join the worker

Generate a current join command on the control plane:

```bash
kubeadm token create --print-join-command
```

Run the resulting command with `sudo` on `k8s-worker-1`:

```bash
sudo kubeadm join 192.168.0.161:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<CA_PUBLIC_KEY_HASH>
```

Watch it from the control plane:

```bash
kubectl get nodes -w
```

The worker should register first and become `Ready` after its Calico Pod is ready.

> During `kubeadm join`, the `kubernetes.io/kube-apiserver-client-kubelet` CSR used for node bootstrap is automatically approved by the kubeadm-installed bootstrap approval path. This is separate from the `kubernetes.io/kubelet-serving` CSR used for the kubelet HTTPS serving certificate.

### Install kubelet-csr-approver

The initial kubeadm configuration makes every kubelet request a CA-signed serving certificate, but Kubernetes deliberately leaves `kubernetes.io/kubelet-serving` CSRs `Pending`. Manual approval is not a permanent solution because certificate rotation and future nodes create new CSRs.

Without an approver, each serving request would need to be reviewed and approved manually:

```bash
kubectl get csr
kubectl describe csr <CSR_NAME>
kubectl certificate approve <CSR_NAME>
```

Check whether Helm is already installed:

```bash
helm version --short
```

If Helm 3 is missing, install it on the workstation with the Helm project's
official `get-helm-3` script. Download the script as a file so it can be
inspected before execution:

```bash
curl -fsSL -o get_helm.sh \
  https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
less get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
rm get_helm.sh
helm version --short
```

This is the script installation method documented in the official
[Helm 3 installation guide](https://helm.sh/docs/v3/intro/install/#from-script).
Helm uses the workstation's current kubeconfig and context, so it does not need
to be installed on either Kubernetes node.

Install [`kubelet-csr-approver`](https://github.com/postfinance/kubelet-csr-approver):

```bash
helm repo add kubelet-csr-approver \
  https://postfinance.github.io/kubelet-csr-approver
helm repo update

helm install kubelet-csr-approver \
  kubelet-csr-approver/kubelet-csr-approver \
  --namespace kube-system \
  --set providerRegex='^k8s-.*$' \
  --set providerIpPrefixes='192.168.0.0/24' \
  --set bypassDnsResolution='true'
```

`providerRegex` restricts acceptable node hostnames, while `providerIpPrefixes` restricts the IP SANs that may be signed. `bypassDnsResolution=true` is used because this lab does not provide LAN DNS records for individual Kubernetes node names. The approver still validates the requester, Common Name, signer, hostname, and SANs.

Wait for the controller and verify the serving CSRs:

```bash
kubectl rollout status deployment/kubelet-csr-approver -n kube-system
kubectl get csr
```

The expected distinction is:

```text
kubernetes.io/kube-apiserver-client-kubelet   Approved,Issued  # automatic node join
kubernetes.io/kubelet-serving                 Approved,Issued  # kubelet-csr-approver
```

Old duplicate serving CSRs can remain visible because CSRs are historical API objects. What matters is that each current node has an approved serving CSR and presents the signed certificate.

The API server verification flag was already installed by `kubeadm init`; the approver does not add or modify that flag. After approval, verify the API server-to-kubelet path:

```bash
kubectl get --raw \
  "/api/v1/nodes/$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')/proxy/healthz"

kubectl logs -n kube-system \
  deployment/kubelet-csr-approver --tail=20
```

---

## 8. Verify the cluster

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get --raw='/readyz?verbose'
kubectl get csr
```

Run a basic Pod networking and DNS test:

```bash
kubectl run net-test \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sleep 3600

kubectl wait --for=condition=Ready pod/net-test --timeout=120s
kubectl exec net-test -- nslookup kubernetes.default
kubectl delete pod net-test
```

The short Service name uses the DNS search domains injected into the Pod and
therefore works with the configured cluster domain. Its complete name is
`kubernetes.default.svc.<dnsDomain>`; for example,
`kubernetes.default.svc.cluster.local` when `dnsDomain: cluster.local`, or
`kubernetes.default.svc.kubeadm.lab` when `dnsDomain: kubeadm.lab`. Querying a
different cluster domain correctly returns `NXDOMAIN`.

Confirm that a workload can run on the worker:

```bash
kubectl create deployment nginx --image=nginx:alpine
kubectl scale deployment nginx --replicas=2
kubectl get pods -o wide
kubectl delete deployment nginx
```

---

## 9. Inspect what kubeadm created

### Control-plane files

```bash
sudo find /etc/kubernetes -maxdepth 2 -type f -ls
sudo ls -la /etc/kubernetes/manifests
sudo sed -n '1,240p' /etc/kubernetes/manifests/kube-apiserver.yaml
sudo sed -n '1,220p' /etc/kubernetes/manifests/etcd.yaml
```

The files under `/etc/kubernetes/manifests` are static Pod manifests. Kubelet watches this directory and asks the container runtime to run the API server, controller manager, scheduler, and local etcd.

### Kubeconfig files

```bash
sudo ls -l /etc/kubernetes/*.conf
sudo kubectl --kubeconfig=/etc/kubernetes/admin.conf config view --minify
sudo kubectl --kubeconfig=/etc/kubernetes/controller-manager.conf config view --minify
sudo kubectl --kubeconfig=/etc/kubernetes/scheduler.conf config view --minify
```

Do not expose kubeconfig files containing private keys in terminal captures, screenshots, or Git.

### Certificates

```bash
sudo find /etc/kubernetes/pki -maxdepth 2 -type f -ls
sudo openssl x509 \
  -in /etc/kubernetes/pki/apiserver.crt \
  -noout -subject -issuer -dates -ext subjectAltName

sudo kubeadm certs check-expiration
kubectl get csr
```

### Container runtime and kubelet

```bash
sudo crictl pods
sudo crictl ps -a
sudo crictl images
sudo journalctl -u kubelet -n 200 --no-pager
sudo journalctl -u containerd -n 100 --no-pager
```

### Ports and networking

```bash
sudo ss -lntp
ip -br link
ip route
ip rule
sudo iptables-save
```

Inspect the Calico resources:

```bash
kubectl get pods -n kube-system -l k8s-app=calico-node -o wide
kubectl get daemonset calico-node -n kube-system -o yaml
kubectl get configmap calico-config -n kube-system -o yaml
```

---

## 10. Rebuild the cluster with kubeadm phases

Revert both VMs to the `pre-kubeadm` snapshots from Section 4. This is cleaner than relying only on `kubeadm reset -f`, because reset may not remove every network interface, route, or iptables rule created by a CNI.

In the Proxmox web interface:

1. Stop the VM.
2. Open `VM → Snapshots`, select its `pre-kubeadm` snapshot, and click **Rollback**.
3. Repeat for the other VM, then start both VMs again.

Rollback discards every VM change made after the selected snapshot.

Inspect the phase commands provided by the installed kubeadm version:

```bash
kubeadm init phase --help
kubeadm init phase certs --help
kubeadm init phase kubeconfig --help
kubeadm init phase control-plane --help
```

`kubeadm init phase --help` only lists and explains the available phases.
Running a concrete phase is **not a preview**: it performs that part of the
initialization workflow and changes files or cluster state. For example,
`kubeadm init phase certs all` really writes the PKI files, while
`control-plane all` really writes the static Pod manifests.

Some phase commands provide `--dry-run`, which prints what would be done
without applying the changes. Check the installed version's help before using
it:

```bash
kubeadm init phase certs all --help
kubeadm init phase certs all \
  --config /home/ubuntu/kubeadm-config.yaml \
  --dry-run
```

Study the main sequence:

1. `preflight`
2. `certs`
3. `kubeconfig`
4. `kubelet-start`
5. `etcd local`
6. `control-plane`
7. `wait-control-plane`
8. `upload-config`
9. `upload-certs`
10. `mark-control-plane`
11. `bootstrap-token`
12. `addon kube-proxy`
13. `addon coredns`

After each phase, inspect the generated files and running processes:

```bash
sudo find /etc/kubernetes -maxdepth 3 -type f -ls
sudo crictl ps -a
sudo journalctl -u kubelet -n 100 --no-pager
sudo ss -lntp
```

Use the `--help` output from the installed kubeadm release when running the phases. Not every phase is independent or safe to run in an arbitrary order.

---

## Official references

- Ubuntu Server: <https://ubuntu.com/download/server>
- Proxmox VE Administration Guide: <https://pve.proxmox.com/pve-docs/pve-admin-guide.pdf>
- Installing kubeadm: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/>
- Container runtimes: <https://kubernetes.io/docs/setup/production-environment/container-runtimes/>
- Creating a cluster with kubeadm: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/>
- kubeadm v1beta4 config API: <https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/>
- kubeadm join: <https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/>
- kubeadm init phases: <https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init-phase/>
- Calico on-premises installation: <https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises>
