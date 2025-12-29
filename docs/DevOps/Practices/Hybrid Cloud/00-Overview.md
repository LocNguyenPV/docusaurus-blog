# Overview
![alt text](image.png)
## Điều kiện tiên quyết

- GCP account
- Có sẵn hạ tầng on-premise / giả lập trên GCP
- Google Cloud CLI [link](https://docs.cloud.google.com/sdk/docs/install-sdk)

## Hạ tầng on-premise yêu cầu

Để dựng được cụm on-premise giống trong series (Kubeadm + Ingress, sau này thêm Harbor, GitLab, Jenkins, ArgoCD), hạ tầng tối thiểu là:

### 1. Compute (máy chủ / VM)

- **Số lượng node:**
  - 1 node control-plane (master)
  - 2 node worker
- **Cấu hình tối thiểu mỗi node (lab/pet project):**
  - 2 vCPU
  - 4 GB RAM (8 GB sẽ thoải mái hơn, đặc biệt cho master nếu chạy thêm GitLab/Jenkins)
  - 30–50 GB disk (boot + data) trên mỗi VM
- **Tổng tối thiểu:** 3 × (2 vCPU, 4 GB RAM, 30–50 GB disk) là đủ để chạy Kubeadm cluster + Ingress + vài service DevOps cơ bản.

---

### 2. Network

- **1 mạng riêng** cho lab:
  - Nếu trên cloud: 1 VPC riêng, ví dụ `devops-hybrid`.
  - Nếu on-prem thật: 1 subnet nội bộ dành riêng cho cluster.
- **1 subnet** cho cụm Kubeadm, ví dụ:
  - CIDR: `10.0.0.0/20`.
- **Firewall / security rules tối thiểu:**
  - Cho phép traffic nội bộ trong subnet/VPC (ví dụ `10.0.0.0/16`).
  - Cho phép SSH (TCP 22) từ IP máy admin (máy của bạn).
  - Cho phép 2 port NodePort mà Ingress sẽ dùng (ví dụ 30131 cho HTTP, 32690 cho HTTPS) từ internet hoặc ít nhất từ IP bạn.

---

### 3. Phần mềm/Kubernetes stack trên mỗi node

- **Hệ điều hành:**
  - Ubuntu 22.04 LTS (hoặc distro Linux tương đương).
- **Container runtime:**
  - `containerd` (gợi ý cho Kubernetes mới).
- **Kubernetes components:**
  - `kubeadm`, `kubelet`, `kubectl` (version stable, ví dụ 1.30).
- **CNI plugin:**
  - Flannel hoặc Calico, với pod network CIDR (ví dụ `10.244.0.0/16`) để pod giao tiếp được với nhau.

---

### 4. Storage

- **Boot disk mỗi node:**
  - 30–50 GB là đủ cho OS + binary + log.
- **PersistentVolume cho ứng dụng (sau này):**
  - Ít nhất 1 StorageClass dùng disk bền vững (nếu trên GCP là Persistent Disk) để tạo PV cho:
    - Harbor (lưu image).
    - GitLab (database + repo).
    - Jenkins (JENKINS_HOME).

---

### 5. Entry point / truy cập từ ngoài

- **Ingress-NGINX Controller:**
  - Chạy trong namespace `ingress-nginx` trên cụm Kubeadm.
  - Service type `NodePort` mở 2 port (HTTP/HTTPS) để nhận traffic từ ngoài.
- **DNS tối thiểu:**
  - Có thể chỉ dùng file `hosts` trên máy dev/test:
    - `harbor.local`, `gitlab.local`, `jenkins.local`, `hello.local` → trỏ về External IP của một node on-prem (master hoặc worker).
