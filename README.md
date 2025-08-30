# Kubernetes Cluster Setup on Debian 13 (Trixie) with containerd

This guide will walk you through installing a Kubernetes (K8s) cluster on Debian 13 ("Trixie") using the latest versions of **containerd** and Kubernetes tools. The guide is tailored for a local environment with the following host and network configuration:

- **Subnet:** `192.168.20.0/24`
- **Master:** `192.168.20.34` (hostname: `k8s-master`)
- **Worker 1:** `192.168.20.35` (hostname: `k8s-worker01`)
- **Worker 2:** `192.168.20.36` (hostname: `k8s-worker02`)

The cluster will use a kubeadm config YAML for initialization, with all networking settings specified there.

---

## 1. Set Hostnames and Hosts File

On each node, set the hostname:
```sh
hostnamectl set-hostname k8s-master      # On master
hostnamectl set-hostname k8s-worker01    # On worker 1
hostnamectl set-hostname k8s-worker02    # On worker 2
```

Add all nodes to `/etc/hosts` on **every node**:
```
192.168.20.34 k8s-master
192.168.20.35 k8s-worker01
192.168.20.36 k8s-worker02
```

---

## 2. Kernel Modules and Sysctl

```sh
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```

---

## 3. Install containerd

```sh
apt update && apt install -y ca-certificates curl gnupg lsb-release
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | \
  gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

apt update
apt install -y containerd.io
```

Configure containerd:
```sh
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd
```

---

## 4. Disable Swap

```sh
swapoff -a
sed -i '/swap/d' /etc/fstab
```

---

## 5. Install Kubernetes Tools

```sh
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \
  tee /etc/apt/sources.list.d/kubernetes.list

apt update
apt install -y kubelet kubeadm kubectl
```
