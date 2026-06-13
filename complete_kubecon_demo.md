# Simulating RK3588 Edge Nodes with KubeEdge — Complete Reproduction Guide

**KubeCon + CloudNativeCon India 2026**
*Sachin Jha, LFX Intern, CNCF (KubeEdge)*

> This guide was built from a real end-to-end setup. Every error encountered during the process has been accounted for. Follow the steps exactly in order and you will have a working KubeEdge cluster with a QEMU-emulated ARM64 edge node.

---

## What You'll Build

```
┌──────────────────────────────┐        ┌──────────────────────────────┐
│       AWS VM 1 (x86)         │        │       AWS VM 2 (x86)         │
│                              │        │                              │
│  Ubuntu 22.04                │◄──────►│  Ubuntu 22.04 (host)         │
│  Kubernetes v1.29            │        │  QEMU ARM64 VM (guest)       │
│  KubeEdge CloudCore v1.17    │        │    └─ Ubuntu 22.04 aarch64   │
│                              │        │    └─ Mosquitto MQTT broker  │
│                              │        │    └─ KubeEdge EdgeCore       │
└──────────────────────────────┘        └──────────────────────────────┘
        CloudCore VM                            Edge Host VM
```

The RK3588 SoC uses Cortex-A76 performance cores. QEMU emulates this exact core giving a native `aarch64` environment — same instruction set, same kernel ABI, same behavior as a real RK3588 board for everything above the hardware abstraction layer.

---

## Instance Selection

### VM 1 — CloudCore

**Type: `t3.medium` (2 vCPU, 4 GB RAM)**

Kubernetes control plane requires at least 2 vCPU. A t3.micro fails kubeadm preflight. A t3.small has too little RAM once etcd and kube-apiserver are both running.

| Field | Value |
|---|---|
| Instance type | `t3.medium` |
| OS | Ubuntu 22.04 LTS |
| Storage | 20 GB gp3 |

**Security Group — inbound rules (add all of these before starting):**

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 22 | TCP | Your IP | SSH |
| 6443 | TCP | 0.0.0.0/0 | Kubernetes API server |
| 10000 | TCP | 0.0.0.0/0 | KubeEdge CloudCore websocket |
| 10002 | TCP | 0.0.0.0/0 | KubeEdge CloudCore HTTPS |

> Add all four inbound rules before running any commands. Missing port 10000 or 10002 will cause EdgeCore to start but never connect to CloudCore, and `edge1` will stay `NotReady` indefinitely.

---

### VM 2 — Edge Host

**Type: `t3.xlarge` (4 vCPU, 16 GB RAM)**

This is the most important sizing decision in the entire setup. QEMU ARM64 emulation is CPU-intensive and will pin one full core at 100% continuously. On top of that, the guest VM needs 2 GB RAM, and once EdgeCore starts it pulls container images and runs workloads.

On a `t3.medium` or `t3.large` you will hit these issues:
- EdgeCore starts but `edge1` stays `NotReady` indefinitely
- SSH into the guest VM becomes unresponsive after EdgeCore starts
- QEMU gets OOM-killed by the host kernel

A `t3.xlarge` gives enough headroom for stable, reproducible operation.

| Field | Value |
|---|---|
| Instance type | `t3.xlarge` |
| OS | Ubuntu 22.04 LTS |
| Storage | 20 GB gp3 |

**Security Group — inbound rules:** SSH (22) only. The edge node connects outbound to CloudCore on ports 10000 and 10002 — no inbound ports needed on VM 2.

---

## Part 1 — CloudCore Setup (VM 1)

SSH into VM 1:
```bash
ssh -i your-key.pem ubuntu@<VM1_PUBLIC_IP>
```

### Step 1 — Fix Kernel Parameters

```bash
sudo swapoff -a

sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

### Step 2 — Install Containerd

```bash
sudo apt update
sudo apt install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl enable containerd
sudo systemctl restart containerd
```

### Step 3 — Install Kubernetes Tools

```bash
sudo apt install -y apt-transport-https ca-certificates curl

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Step 4 — Initialize the Cluster

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

After it completes:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Step 5 — Install Flannel CNI

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

Wait 30 seconds, then verify:

```bash
kubectl get nodes
```

Expected output:
```
NAME            STATUS   ROLES           AGE   VERSION
ip-x-x-x-x     Ready    control-plane   1m    v1.29.x
```

Do not proceed until STATUS shows `Ready`.

### Step 6 — Install KubeEdge CloudCore

```bash
wget https://github.com/kubeedge/kubeedge/releases/download/v1.17.0/keadm-v1.17.0-linux-amd64.tar.gz
tar -zxvf keadm-v1.17.0-linux-amd64.tar.gz

# The binary is nested two directories deep — use the exact path below
sudo cp keadm-v1.17.0-linux-amd64/keadm/keadm /usr/local/bin/keadm
sudo chmod +x /usr/local/bin/keadm

# Install helm (required by keadm init)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

export CLOUDCORE_IP=$(curl -s http://checkip.amazonaws.com)
echo "CloudCore IP: $CLOUDCORE_IP"

sudo keadm init \
  --advertise-address=$CLOUDCORE_IP \
  --kubeedge-version=1.17.0 \
  --kube-config=/home/ubuntu/.kube/config
```

Expected output includes:
```
CLOUDCORE started
STATUS: deployed
```

If you see `context deadline exceeded` or `cloudcore release already exists`, run this cleanup and retry:

```bash
sudo helm uninstall cloudcore -n kubeedge --kubeconfig /home/ubuntu/.kube/config
kubectl delete namespace kubeedge --ignore-not-found
sleep 20
sudo keadm init \
  --advertise-address=$CLOUDCORE_IP \
  --kubeedge-version=1.17.0 \
  --kube-config=/home/ubuntu/.kube/config
```

### Step 7 — Get the Join Token

```bash
sleep 15
keadm gettoken --kube-config=/home/ubuntu/.kube/config
```

Copy the full token string. You will need it in Part 2, Step 10.

---

## Part 2 — Edge Node Setup (VM 2)

SSH into VM 2:
```bash
ssh -i your-key.pem ubuntu@<VM2_PUBLIC_IP>
```

### Step 1 — Install QEMU and Tools

```bash
sudo apt update
sudo apt install -y \
  qemu-system-arm \
  qemu-efi-aarch64 \
  cloud-image-utils \
  cloud-guest-utils
```

### Step 2 — Download the ARM64 Cloud Image

```bash
mkdir -p ~/qemu-edge
cd ~/qemu-edge

wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-arm64.img
```

This downloads Ubuntu 22.04 LTS for ARM64 (~650 MB). Keep the original file — it is your backup if you ever need to rebuild the VM from scratch.

### Step 3 — Resize the Disk to 10 GB Before First Boot

The default image is 2.2 GB. Installing containerd, mosquitto, and KubeEdge requires ~3 GB. Resize now before booting — doing it after the fact requires manually freeing space.

```bash
cp jammy-server-cloudimg-arm64.img cloud.img
qemu-img resize cloud.img 10G
```

### Step 4 — Create cloud-init Seed Image

```bash
[ -f ~/.ssh/id_rsa.pub ] || ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa -N ""

PUBKEY=$(cat ~/.ssh/id_rsa.pub)

# Verify the key was read correctly — must show your real public key
echo "Key starts with: $(echo $PUBKEY | cut -c1-30)..."

cat > user-data <<EOF
#cloud-config
users:
  - name: ubuntu
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - ${PUBKEY}
ssh_pwauth: false
disable_root: true
EOF

cat > meta-data <<EOF
instance-id: edge-vm-01
local-hostname: edge1
EOF

cloud-localds seed.img user-data meta-data
```

### Step 5 — Boot the ARM64 VM

```bash
nohup qemu-system-aarch64 \
  -machine virt \
  -cpu cortex-a76 \
  -smp 4 \
  -m 2048 \
  -nographic \
  -bios /usr/share/qemu-efi-aarch64/QEMU_EFI.fd \
  -drive file=cloud.img,if=virtio,format=qcow2 \
  -drive file=seed.img,if=virtio,format=raw \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=net0 \
  > ~/qemu-edge/qemu.log 2>&1 &

echo "VM PID: $!"
```

What each flag does:
- `-cpu cortex-a76` — emulates RK3588 performance cores
- `-bios /usr/share/qemu-efi-aarch64/QEMU_EFI.fd` — ARM64 UEFI firmware. Without this flag the VM starts silently, port 2222 opens, but SSH hangs forever at banner exchange
- `-smp 4` — 4 virtual CPUs
- `-m 2048` — 2 GB RAM for the guest
- `hostfwd=tcp::2222-:22` — forwards host port 2222 to guest SSH port 22

### Step 6 — Wait for the VM to Boot

First boot takes 5-8 minutes because cloud-init runs. Use this loop — it retries every 15 seconds and prints "VM is up!" the moment SSH responds:

```bash
until ssh -i ~/.ssh/id_rsa -p 2222 ubuntu@127.0.0.1 \
  -o StrictHostKeyChecking=no \
  -o ConnectTimeout=10 \
  -o BatchMode=yes \
  echo "VM is up!"; do
  echo "Waiting... $(date)"
  sleep 15
done
```

Do not interrupt QEMU or restart it while waiting. The "Connection timed out during banner exchange" messages are normal during early boot.

### Step 7 — SSH In and Expand the Filesystem

```bash
ssh -i ~/.ssh/id_rsa -p 2222 ubuntu@127.0.0.1 -o StrictHostKeyChecking=no
```

Verify you are on ARM64:
```bash
uname -m
# Expected: aarch64
```

Free up space before expanding (growpart needs temporary disk space to work):
```bash
sudo apt clean
sudo journalctl --vacuum-size=50M
```

Expand the partition to use the full 10 GB:
```bash
sudo growpart /dev/vda 1
sudo resize2fs /dev/vda1
df -h
```

Expected after resize:
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1       9.6G  1.9G  7.7G  20% /
```

Do not proceed until you see at least 7 GB available.

### Step 8 — Install Containerd on the Edge VM

Use cgroupfs (not systemd) for the edge VM. Containerd 2.x with `SystemdCgroup = true` causes a sandbox creation failure during `keadm join` with the error:
```
expected cgroupsPath to be of format "slice:prefix:name" for systemd cgroups
```

Use the exact config below — do not modify it:

```bash
sudo apt update
sudo apt install -y containerd

sudo systemctl stop containerd
sudo rm -f /etc/containerd/config.toml
sudo mkdir -p /etc/containerd

cat <<EOF | sudo tee /etc/containerd/config.toml
version = 2

[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".containerd]
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = false
EOF

sudo systemctl enable containerd
sudo systemctl start containerd
sudo systemctl status containerd | head -3
```

Expected:
```
● containerd.service - containerd container runtime
   Loaded: loaded (/lib/systemd/system/containerd.service; enabled)
   Active: active (running) ...
```

Do not proceed until containerd shows `active (running)`.

### Step 9 — Install Mosquitto MQTT Broker

EdgeCore requires a local MQTT broker. Without it, EdgeCore starts but repeatedly fails with `dial tcp 127.0.0.1:1883: connection refused` and the node never becomes Ready.

```bash
sudo apt install -y mosquitto
sudo systemctl enable mosquitto
sudo systemctl start mosquitto
sudo systemctl status mosquitto | head -3
```

Expected:
```
● mosquitto.service - Mosquitto MQTT Broker
   Loaded: loaded (/lib/systemd/system/mosquitto.service; enabled)
   Active: active (running) ...
```

Do not proceed until mosquitto shows `active (running)`.

### Step 10 — Join the KubeEdge Cluster

```bash
wget https://github.com/kubeedge/kubeedge/releases/download/v1.17.0/keadm-v1.17.0-linux-arm64.tar.gz
tar -zxvf keadm-v1.17.0-linux-arm64.tar.gz

# Binary is nested two directories deep
sudo cp keadm-v1.17.0-linux-arm64/keadm/keadm /usr/local/bin/keadm
sudo chmod +x /usr/local/bin/keadm

# Remove any leftover files from previous attempts
sudo rm -rf /etc/kubeedge

# Replace CLOUDCORE_PUBLIC_IP and TOKEN below with real values
# (do NOT keep the angle brackets < > — they will be interpreted by bash as redirection)
CLOUDCORE_PUBLIC_IP="51.21.250.125"        # <-- set this to VM 1's public IP
TOKEN="f278a860fcbe34c1d83eb7ec6f9bbdbf4afb10b21ce6048fe8c96beaa2497cbc.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3ODE0MjY0NzN9.L2Nbxdmfe-hCIrslCaUZsOVB4buCSRA9uoVi7EUwb-4"        # <-- set this to the token from keadm gettoken

sudo keadm join \
  --cloudcore-ipport=${CLOUDCORE_PUBLIC_IP}:10000 \
  --token=${TOKEN} \
  --kubeedge-version=1.17.0
```

This pulls a container image (~50 MB) and installs EdgeCore. On ARM64 emulation it takes 3-5 minutes. Expected final output:

```
KubeEdge edgecore is running, For logs visit: journalctl -u edgecore.service -xe
Install Complete!
```

---

## Part 3 — Verify the Setup

On **VM 1 (CloudCore)**:

```bash
kubectl get nodes
```

Expected:
```
NAME              STATUS   ROLES           AGE   VERSION
edge1             Ready    agent,edge      2m    v1.28.6-kubeedge-v1.17.0
ip-172-xx-xx-xx   Ready    control-plane   30m   v1.29.15
```

`edge1` may show `NotReady` for 1-2 minutes after joining while EdgeCore initializes. This is normal. If it stays `NotReady` beyond 5 minutes, see the Troubleshooting section.

### Deploy a Pod to the Edge Node

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-edge
  namespace: default
spec:
  nodeName: edge1
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
EOF

kubectl get pod nginx-edge -w
```

Expected progression:
```
NAME         READY   STATUS              RESTARTS   AGE
nginx-edge   0/1     Pending             0          0s
nginx-edge   0/1     ContainerCreating   0          5s
nginx-edge   1/1     Running             0          30s
```

### Verify ARM64 Architecture

```bash
kubectl describe node edge1 | grep -E "Architecture|OS Image|kubeedge"
```

Expected:
```
Architecture:        aarch64
OS Image:            Ubuntu 22.04.5 LTS
kubeedge-version=v1.17.0
```

---

## Saving the Start Script

Save this on VM 2 to restart the edge VM after any reboot or stop/start of the instance. On subsequent boots cloud-init does not run again so the VM comes up in 2-3 minutes.

```bash
cat > ~/qemu-edge/start-edge.sh << 'EOF'
#!/bin/bash
pkill -9 -f qemu-system || true
sleep 2

nohup qemu-system-aarch64 \
  -machine virt \
  -cpu cortex-a76 \
  -smp 4 \
  -m 2048 \
  -nographic \
  -bios /usr/share/qemu-efi-aarch64/QEMU_EFI.fd \
  -drive file=/home/ubuntu/qemu-edge/cloud.img,if=virtio,format=qcow2 \
  -drive file=/home/ubuntu/qemu-edge/seed.img,if=virtio,format=raw \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=net0 \
  > /home/ubuntu/qemu-edge/qemu.log 2>&1 &

echo "Edge VM started, PID: $!"
echo "Wait 2-3 mins, then: ssh -i ~/.ssh/id_rsa -p 2222 ubuntu@127.0.0.1"
EOF

chmod +x ~/qemu-edge/start-edge.sh
```

To start the VM anytime:
```bash
cd ~/qemu-edge && bash start-edge.sh
```

---

## Troubleshooting

**"Failed to get write lock on cloud.img"**
Another QEMU process is already running. Kill it and retry:
```bash
pkill -9 -f qemu-system
sleep 2
```

**SSH hangs at banner exchange / times out indefinitely**
The `-bios` flag is missing from the QEMU command. ARM64 VMs require UEFI firmware to boot. Without it, QEMU accepts the TCP connection on port 2222 but the VM never starts SSH. Verify `qemu-efi-aarch64` is installed and the `-bios /usr/share/qemu-efi-aarch64/QEMU_EFI.fd` flag is present.

**"No space left on device" when running apt install inside the VM**
The disk was not resized before first boot. Stop the VM, copy the original image back, resize, and redo Steps 5 onwards:
```bash
# On the host VM
pkill -9 -f qemu-system
cp jammy-server-cloudimg-arm64.img cloud.img
qemu-img resize cloud.img 10G
```

**growpart fails: "failed to make temp dir"**
The disk is 100% full and growpart cannot create a temp directory. Free space first:
```bash
sudo apt clean
sudo journalctl --vacuum-size=50M
sudo rm -rf /tmp/*
```
Then retry `sudo growpart /dev/vda 1`.

**kubeadm init fails: bridge-nf-call-iptables or ip_forward errors**
The kernel parameters were not applied. Run:
```bash
sudo modprobe br_netfilter
sudo sysctl -w net.bridge.bridge-nf-call-iptables=1
sudo sysctl -w net.ipv4.ip_forward=1
```
Then retry `sudo kubeadm init`.

**keadm init: "context deadline exceeded"**
CloudCore pod failed to start in time — usually an image pull timeout. Clean up and retry:
```bash
sudo helm uninstall cloudcore -n kubeedge --kubeconfig /home/ubuntu/.kube/config
kubectl delete namespace kubeedge --ignore-not-found
sleep 20
sudo keadm init \
  --advertise-address=$CLOUDCORE_IP \
  --kubeedge-version=1.17.0 \
  --kube-config=/home/ubuntu/.kube/config
```

**keadm init: "cloudcore release already exists"**
A previous failed or partial install left a Helm release behind. Run the same cleanup as above and retry.

**keadm gettoken: "secrets tokensecret not found"**
CloudCore did not finish initializing. Wait 15-20 seconds after `keadm init` completes, then retry `keadm gettoken`.

**keadm: "stat /root/.kube/config: no such file or directory"**
Always pass the explicit kubeconfig path:
```bash
--kube-config=/home/ubuntu/.kube/config
```

**keadm binary not found after extraction**
The binary is inside two nested directories. The correct path is:
```
keadm-v1.17.0-linux-amd64/keadm/keadm
```

**keadm join: "management directory /etc/kubeedge is not clean"**
A previous failed join left files behind. Remove them and retry:
```bash
sudo rm -rf /etc/kubeedge
```

**keadm join: "expected cgroupsPath to be of format slice:prefix:name"**
You used `SystemdCgroup = true` in the containerd config on the edge VM. Follow Step 8 exactly — use `SystemdCgroup = false`.

**EdgeCore running but edge1 stays NotReady — "dial tcp 127.0.0.1:1883: connection refused"**
Mosquitto MQTT broker is not installed or not running. Install and start it:
```bash
sudo apt install -y mosquitto
sudo systemctl enable mosquitto
sudo systemctl start mosquitto
sudo systemctl restart edgecore
```

**EdgeCore running but "not connected" to CloudCore**
Ports 10000 and 10002 are not open on VM 1's security group. Go to AWS Console → EC2 → VM 1 → Security Group → Inbound Rules and add both ports. Then restart edgecore:
```bash
sudo systemctl restart edgecore
```

**edge1 stays NotReady / guest VM SSH becomes unresponsive after EdgeCore starts**
The VM 2 host instance is too small. Resize to `t3.xlarge` in the AWS console, then restart and redo Part 2 from Step 5.

**edgecore fails to connect after VM 2 instance restart**
AWS assigns a new public IP on instance stop/start. On VM 1 get the new IP and regenerate the token:
```bash
export CLOUDCORE_IP=$(curl -s http://checkip.amazonaws.com)
sudo helm uninstall cloudcore -n kubeedge --kubeconfig /home/ubuntu/.kube/config
kubectl delete namespace kubeedge --ignore-not-found
sleep 20
sudo keadm init \
  --advertise-address=$CLOUDCORE_IP \
  --kubeedge-version=1.17.0 \
  --kube-config=/home/ubuntu/.kube/config
sleep 15
keadm gettoken --kube-config=/home/ubuntu/.kube/config
```
Then on the edge VM rejoin with the new token:
```bash
sudo systemctl stop edgecore
sudo rm -rf /etc/kubeedge
sudo keadm join \
  --cloudcore-ipport=<NEW_CLOUDCORE_IP>:10000 \
  --token=<NEW_TOKEN> \
  --kubeedge-version=1.17.0
```

---

*Session: Validating RK3588 for KubeEdge: Scalable ARM64 Edge Node Simulation Without Hardware*
*KubeCon + CloudNativeCon India 2026 — Friday June 19, 4:10–4:40 PM IST, Room 205*
