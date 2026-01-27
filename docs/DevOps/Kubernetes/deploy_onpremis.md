---
sidebar_position: 4
---

# Deploy Kubernetes On-Premises From Zero

## Resource Preparation

| Name         | IP            | Role          | RAM | CPU | OS               |
| ------------ | ------------- | ------------- | --- | --- | ---------------- |
| k8s-master-1 | 192.168.1.111 | Control Plane | 3   | 2   | Ubuntu 22.04 LTS |
| k8s-master-2 | 192.168.1.112 | Worker        | 3   | 2   | Ubuntu 22.04 LTS |
| k8s-master-3 | 192.168.1.113 | Worker        | 3   | 2   | Ubuntu 22.04 LTS |

---

## Implementation Steps

### 1 Update system

```bash
sudo apt update -y && sudo apt upgrade -y
```

### 2 Add hosts (above all 3 virtual machines)

Open file host to config ip address

```bash
vim /etc/hosts/
```

Add those configs into file

```bash
192.168.1.111 k8s-master-1
192.168.1.112 k8s-master-2
192.168.1.113 k8s-master-3
```

Saved and check

```bash
ping <ip-addr>
```

### 3 Create a new user

> We should create/switch to another user (avoid using root) to install kubernetes

Create `devops` user and add to `sudo` group

```bash
adduser devops
usermod -aG sudo devops
```

Switch to `devops`

```bash
su devops
cd /home/devops
```

You need to install docker first, you can following [here](../Docker/linux-installation.md)

### 4 Turn off swap

> Because Kubernetes require actual RAM to ensure about performance and stable

```bash
# turn off swap forever
sudo sed -i '/swap/s/^/#/' /etc/fstab
```

### 5 Download and add module into kernel

System will download `overlay` & `br_netfilter` package manually, this is two modules which require for k8s implement

- `overlay`: Support overlay filesystem, it's necessary for containerd to manage container image.
- `br_netfilter`: Allow filter network package on bridge network, it's very important for Kubernetes to handle traffic between container.

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

To make system automatically install two modules above after restart, we need modify config file by command below

```bash
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```

### 6 Config network

Those config to make sure:

- Kubernetes can manage traffic between containers and Pods.
- Firewall rules can apply to traffic via bridge network.
- The system can routing package between nodes in cluster.

```bash
echo "net.bridge.bridge-nf-call-ip6tables = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
echo "net.bridge.bridge-nf-call-iptables = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
```

Apply `sysctl` config

```bash
sudo sysctl --system
```

### 7 Install & config containerd

This is container runtime which is use by Kubernetes for manage container

```bash
sudo apt install -y containerd.io
```

After installed, we need config `containerd` use **Systemd** to manage resource group (cgroup) => This make resource management become consistent.

```bash
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

Restart `containerd`

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 8 Install Kubernetes package

Config to download from legit source

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Install packages (kubelet, kubeadm, kubectl)

```bash
sudo apt install -y kubelet kubeadm kubectl
```

For avoid auto upgrade version => Ensure package versions are consistent.

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## Deploy model (1 master - 2 worker)

![1master2workers](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tmpjtzh0r2bzmt0f0e47.png)

<figCaption>Standard model</figCaption>

### Init Kubernetes on master node

```bash
sudo kubeadm init
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

After the initial setup, it'll have command follow format below - use it to add **worker**

```bash
sudo kubeadm join <ip-master-node>:6443 --token <your_token> --discovery-token-ca-cert-hash <your_sha>
```

## Add worker node

Copy command from **master node** and paste it to machine where role determine is **worker** role (maybe you need to run it with `sudo`)

## Verify

Run `kubectl get node -o wide` in **master node** to check **worker** added

![verify-workers](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ysnih0l0lmtcqi7rdi7n.png)

<figCaption>Example output</figCaption>

## Conclusion

Just one of over 35 ways to install Kubernetes. In the future, to add more workers to your system, simply run

```bash
sudo kubeadm join <ip-master-node>:6443 --token <your_token> --discovery-token-ca-cert-hash <your_sha>
```

In the next post of series, I will show you how to use cloud for Kubernetes
Happy Coding!
