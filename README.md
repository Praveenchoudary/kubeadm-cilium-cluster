# Kubernetes Cluster Installation (kubeadm) with Cilium CNI

**Version:** Kubernetes v1.35
**CNI:** Cilium

---

## 2.1 Overview

This guide walks through building a Kubernetes cluster using **kubeadm**, then installing **Cilium** as the Container Network Interface (CNI).

```
        Control Plane Node
        +------------------+
        |  kube-apiserver   |
        |  etcd             |
        |  scheduler        |
        |  controller-mgr   |
        +------------------+
                │
      ---------------------------
      │            │            │
   Worker1      Worker2      Worker3
   kubelet      kubelet      kubelet
   Cilium       Cilium       Cilium
```

Cilium replaces kube-proxy (optional) and provides networking, network policy, and observability using eBPF.

---

## 2.2 Prerequisites

- 1x Control plane node (min 2 vCPU / 4 GB RAM)
- 2x or more Worker nodes (min 2 vCPU / 4 GB RAM)
- Ubuntu 22.04 / 24.04 (or similar)
- Swap disabled on all nodes
- Unique hostname, MAC address, and product_uuid for every node
- All nodes able to communicate with each other
- Root or sudo access

**Ports required (Control Plane):**

| Protocol | Port Range | Purpose            |
|----------|------------|---------------------|
| TCP      | 6443       | Kubernetes API server |
| TCP      | 2379-2380  | etcd                |
| TCP      | 10250-10259| kubelet / scheduler / controller |

**Ports required (Worker Nodes):**

| Protocol | Port Range | Purpose      |
|----------|------------|--------------|
| TCP      | 10250      | kubelet      |
| TCP      | 30000-32767| NodePort Services |

---

## 2.3 Disable Swap

Run on **all nodes**:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

---

## 2.4 Load Kernel Modules

Run on **all nodes**:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

---

## 2.5 Configure sysctl

Run on **all nodes**:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

---

## 2.6 Install Container Runtime (containerd)

Run on **all nodes**:

```bash
sudo apt update
sudo apt install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## 2.7 Install kubeadm, kubelet, kubectl (v1.35)

Run on **all nodes**:

```bash
sudo apt install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## 2.8 Initialize the Control Plane

Run on **control plane node only**.

> Note: `--skip-phases=addon/kube-proxy` is optional — use it only if you want Cilium to fully replace kube-proxy.

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --upload-certs
```

After init completes, configure kubectl access:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Save the `kubeadm join` command printed in the output — you'll need it for the worker nodes.

---

## 2.9 Join Worker Nodes

Run on **each worker node**, using the join command generated in the previous step:

```bash
sudo kubeadm join <CONTROL_PLANE_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

If the token has expired, generate a new one from the control plane:

```bash
kubeadm token create --print-join-command
```

---

## 2.10 Install Cilium CNI

Install the Cilium CLI (on control plane node):

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --fail --remote-name-all \
  https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz

sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz
```

Install Cilium into the cluster:

```bash
cilium install
```

If kube-proxy was skipped during `kubeadm init`, install Cilium in kube-proxy replacement mode:

```bash
cilium install \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=<CONTROL_PLANE_IP> \
  --set k8sServicePort=6443
```

---

## 2.11 Validate the Cluster

```bash
cilium status --wait
kubectl get nodes -o wide
kubectl get pods -A
```

All nodes should show `Ready`, and all `kube-system` / `cilium` pods should be `Running`.

```
Control Plane   Ready   v1.35.x
Worker1         Ready   v1.35.x
Worker2         Ready   v1.35.x
```

---

## 2.12 Run Cilium Connectivity Test (Optional)

```bash
cilium connectivity test
```

This deploys test workloads across nodes to validate pod-to-pod networking, service routing, and policy enforcement.

---

## 2.13 Summary

| Component  | Role                                  |
|------------|----------------------------------------|
| kubeadm    | Bootstraps the cluster                 |
| containerd | Container runtime                      |
| Cilium     | Pod networking, network policy, eBPF dataplane |
| kubectl    | Cluster management CLI                 |

With this cluster in place, it can later be connected to the Ceph cluster (from Day 1) to provide persistent storage via **Ceph RBD** or **CephFS** for Kubernetes workloads.
