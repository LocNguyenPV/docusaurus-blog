Checklist 4 ngày dưới đây đã được bỏ toàn bộ ref/link và trình bày lại ở dạng markdown để bạn dùng trực tiếp.

***

## Day 1 – Vào được AWS và tạo được EKS cluster

**Mục tiêu:**  
Đăng nhập AWS bằng IAM user, cấu hình CLI, tạo được EKS cluster và `kubectl get nodes` chạy OK.

### 1. Tài khoản & IAM user

- [ ] Tạo tài khoản AWS, đăng nhập AWS Console.  
- [ ] Tạo IAM user (ví dụ: `lab-admin`):  
  - [ ] Cho phép đăng nhập AWS Management Console.  
  - [ ] Gán policy `AdministratorAccess` (CHỈ dùng cho lab, không dùng cho production).  
- [ ] Đăng nhập lại AWS Console bằng user `lab-admin` (không dùng root nữa).

### 2. Access Key & cài tool trên máy

- [ ] Tạo Access Key cho IAM user `lab-admin`:  
  - [ ] Chọn loại sử dụng cho CLI.  
  - [ ] Lưu Access Key ID + Secret Access Key ở nơi an toàn.  

- [ ] Cài công cụ trên Windows:  
  - [ ] Cài AWS CLI v2.  
  - [ ] Tải `eksctl.exe` và `kubectl.exe` và đặt vào `C:\tool`.  
  - [ ] Thêm `C:\tool` vào PATH trong Environment Variables.  

- [ ] Cấu hình AWS CLI:  
  - [ ] Chạy `aws configure`.  
  - [ ] Nhập Access Key, Secret Key.  
  - [ ] Default region: `ap-southeast-1`.  
  - [ ] Default output format: `json`.  
  - [ ] Kiểm tra:  
    - [ ] `aws sts get-caller-identity`  
    - [ ] `kubectl version --client`  
    - [ ] `eksctl version`  

### 3. IAM Role và key pair cho EKS

- [ ] Tạo IAM Role cho EKS control plane:  
  - [ ] IAM → Roles → Create role.  
  - [ ] Trusted entity: AWS service.  
  - [ ] Service: EKS, use case: “EKS - Cluster”.  
  - [ ] Giữ policy `AmazonEKSClusterPolicy`.  
  - [ ] Đặt tên: `lab-eks-cluster` (hoặc tên bạn thích).  
  - [ ] Lưu ARN role.

- [ ] Tạo IAM Role cho worker node:  
  - [ ] IAM → Roles → Create role.  
  - [ ] Trusted entity: AWS service.  
  - [ ] Service: EC2.  
  - [ ] Gán các policy:  
    - [ ] `AmazonEKSWorkerNodePolicy`  
    - [ ] `AmazonEC2ContainerRegistryReadOnly`  
    - [ ] `AmazonEKS_CNI_Policy`  
  - [ ] Đặt tên: `lab-eks-worker`.  
  - [ ] Lưu ARN role.

- [ ] Tạo EC2 Key Pair:  
  - [ ] EC2 → Key Pairs → Create key pair.  
  - [ ] Name: `lab-key`.  
  - [ ] Type: RSA, file `.pem`.  
  - [ ] Tải và lưu file `.pem`.

### 4. File cấu hình EKS và tạo cluster

- [ ] Tạo folder lab (ví dụ: `D:\EKS-Lab`).  
- [ ] Tạo file `lab-cluster.yaml` với nội dung logic sau (tùy chỉnh giá trị cụ thể):  
  - metadata:  
    - name: tên cluster (ví dụ: `lab-eks-cluster`)  
    - region: `ap-southeast-1`  
    - version: `"1.32"` (hoặc version phù hợp)  
  - vpc:  
    - `nat.gateway: Disable` (lab, tiết kiệm chi phí)  
    - `clusterEndpoints.publicAccess: true`  
    - `clusterEndpoints.privateAccess: false`  
  - iam:  
    - `serviceRoleARN`: ARN role `lab-eks-cluster`  
    - `withOIDC: true`  
  - addons: `vpc-cni`, `coredns`, `kube-proxy`, `metrics-server`  
  - managedNodeGroups:  
    - name: `student-workers`  
    - instanceType: loại nhỏ (ví dụ: `t3.medium` hoặc `c7i-flex.large`)  
    - amiFamily: `Ubuntu2404`  
    - minSize: 1, maxSize: 2, desiredCapacity: 2  
    - volumeSize: 30  
    - ssh.allow: `true`  
    - ssh.publicKeyName: `lab-key`  
    - iam.instanceRoleARN: ARN role `lab-eks-worker`  

- [ ] Tạo cluster:  
  - [ ] Mở CMD/PowerShell trong thư mục chứa file yaml.  
  - [ ] Chạy `aws sts get-caller-identity` để chắc kết nối.  
  - [ ] Chạy `eksctl create cluster -f lab-cluster.yaml`.  
  - [ ] Đợi 15–20 phút tới khi báo cluster ready.  
  - [ ] Chạy `kubectl get nodes` → thấy node ở trạng thái `Ready`.

***

## Day 2 – Wildcard SSL với ACM và DNS

**Mục tiêu:**  
Có chứng chỉ wildcard ACM trạng thái `ISSUED`, DNS đã validate xong, sẵn sàng dùng cho ALB/NLB.

### 1. Request wildcard certificate bằng ACM

- [ ] Chọn domain bạn quản lý qua Cloudflare (ví dụ: `yourdomain.com`).  
- [ ] Mở CMD/PowerShell.  
- [ ] Chạy lệnh:  

  ```bash
  aws acm request-certificate \
    --domain-name "*.yourdomain.com" \
    --validation-method DNS \
    --region ap-southeast-1 \
    --output text
  ```

- [ ] Lưu ARN chứng chỉ trả về.

### 2. Lấy record DNS để xác thực

- [ ] Chạy lệnh:

  ```bash
  aws acm describe-certificate \
    --certificate-arn <ARN> \
    --region ap-southeast-1 \
    --query Certificate.DomainValidationOptions[0].ResourceRecord
  ```

- [ ] Ghi lại `Name`, `Type`, `Value` (CNAME).

### 3. Tạo CNAME trên Cloudflare

- [ ] Đăng nhập Cloudflare → chọn domain.  
- [ ] DNS → Add record:  
  - [ ] Type: CNAME  
  - [ ] Name: giá trị `Name` (bỏ dấu `.` cuối nếu có)  
  - [ ] Target: giá trị `Value`  
  - [ ] Lưu record.

### 4. Kiểm tra trạng thái chứng chỉ

- [ ] Sau 1–2 phút, chạy lại:

  ```bash
  aws acm describe-certificate \
    --certificate-arn <ARN> \
    --region ap-southeast-1 \
    --query Certificate.Status
  ```

- [ ] Đảm bảo trạng thái là `ISSUED`.  
- [ ] Ghi lại ARN này để dùng ở Day 3 (ArgoCD + ALB) và Day 4 (Nginx + NLB + app).

***

## Day 3 – Public ArgoCD qua AWS ALB Ingress

**Mục tiêu:**  
Truy cập được ArgoCD qua HTTPS trên domain riêng, dùng AWS ALB Ingress.

### 1. Bật OIDC và tag subnet cho ALB

- [ ] Bật OIDC (nếu chưa bật trong Day 1):

  ```bash
  eksctl utils associate-iam-oidc-provider \
    --cluster lab-eks-cluster \
    --region ap-southeast-1 \
    --approve
  ```

- [ ] Tag subnet public:  
  - [ ] Vào VPC → Subnets.  
  - [ ] Tìm 3 subnet public của cluster (tên dạng `eksctl-<cluster>-clusterSubnetPublic...`).  
  - [ ] Với mỗi subnet, thêm/kiểm tra tag:  
    - Key: `kubernetes.io/role/elb`  
    - Value: `1`.

### 2. IAM Policy + ServiceAccount cho AWS Load Balancer Controller

- [ ] Tải IAM policy JSON:

  ```bash
  curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
  ```

- [ ] Tạo IAM policy:

  ```bash
  aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicyFinal \
    --policy-document file://iam_policy.json
  ```

- [ ] Lưu ARN policy trả về.

- [ ] Tạo IAM service account:

  ```bash
  eksctl create iamserviceaccount \
    --cluster=lab-eks-cluster \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --role-name=AmazonEKSLoadBalancerControllerRoleFinal \
    --attach-policy-arn=<ARN_policy> \
    --region=ap-southeast-1 \
    --override-existing-serviceaccounts \
    --approve
  ```

### 3. Cài AWS Load Balancer Controller bằng Helm

- [ ] Đảm bảo `helm` đã có trong `PATH`.  
- [ ] Thêm repo:

  ```bash
  helm repo add eks https://aws.github.io/eks-charts
  helm repo update
  ```

- [ ] Cài đặt controller:

  ```bash
  helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=lab-eks-cluster \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller
  ```

- [ ] Kiểm tra:

  ```bash
  kubectl get deployment -n kube-system aws-load-balancer-controller
  ```

  READY phải là `2/2`.

### 4. Tạo Ingress cho ArgoCD

- [ ] Viết file `argocd-ingress.yaml`, các điểm chính:  
  - apiVersion: `networking.k8s.io/v1`  
  - kind: `Ingress`  
  - metadata:  
    - namespace: `argocd`  
    - annotations (ví dụ):  
      - `kubernetes.io/ingress.class: alb`  
      - `alb.ingress.kubernetes.io/scheme: internet-facing`  
      - `alb.ingress.kubernetes.io/target-type: ip`  
      - `alb.ingress.kubernetes.io/certificate-arn: <ARN_ACM>`  
      - `alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80,"HTTPS":443}]'`  
      - annotation để redirect HTTP → HTTPS (tuỳ cấu hình bạn muốn).  
  - spec:  
    - rules → http → paths → backend: `service: argocd-server`, port `80`.

- [ ] Apply Ingress:

  ```bash
  kubectl apply -f argocd-ingress.yaml
  ```

- [ ] Restart controller (cho chắc nó thấy Ingress mới):

  ```bash
  kubectl delete pod -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
  ```

### 5. Cấu hình DNS truy cập ArgoCD

- [ ] Lấy DNS ALB:

  ```bash
  kubectl get ingress -n argocd
  ```

  Copy giá trị cột ADDRESS (ALB DNS).

- [ ] Cloudflare → DNS → Add CNAME:  
  - [ ] Name: ví dụ `argocd-aws.yourdomain.com`.  
  - [ ] Target: ALB DNS.  

- [ ] Truy cập `https://argocd-aws.yourdomain.com` và login ArgoCD.

***

## Day 4 – App qua ArgoCD + Helm + Nginx Ingress (NLB)

**Mục tiêu:**  
Deploy app bằng ArgoCD + Helm, public qua NLB + Nginx + HTTPS.

### 1. Cài Nginx Ingress Controller với NLB + ACM

- [ ] Thêm repo:

  ```bash
  helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
  helm repo update
  ```

- [ ] Cài chart (chỉnh ARN):

  ```bash
  helm install nginx-ingress ingress-nginx/ingress-nginx ^
    --namespace ingress-nginx ^
    --create-namespace ^
    --set controller.service.type=LoadBalancer ^
    --set controller.service.annotations."service.beta.kubernetes.io/aws-load-balancer-type"="nlb" ^
    --set controller.service.annotations."service.beta.kubernetes.io/aws-load-balancer-scheme"="internet-facing" ^
    --set controller.service.annotations."service.beta.kubernetes.io/aws-load-balancer-ssl-cert"="<ARN_ACM>" ^
    --set controller.service.annotations."service.beta.kubernetes.io/aws-load-balancer-ssl-ports"="443" ^
    --set controller.service.targetPorts.https=http
  ```

### 2. Mở NodePort range cho lab

- [ ] EC2 → chọn 1 instance worker.  
- [ ] Tab Security → Security groups → click vào SG.  
- [ ] Inbound rules → Edit inbound rules:  
  - [ ] Add rule:  
    - Type: Custom TCP  
    - Port range: `30000-32767`  
    - Source: `0.0.0.0/0` (LAB ONLY)  
    - Description: `Allow Nginx Ingress Traffic`  

### 3. Namespace & secret cho app

- [ ] Tạo namespace app:

  ```bash
  kubectl create ns ha-config
  ```

- [ ] Tạo secret để kéo image từ registry (ví dụ Harbor):

  ```bash
  kubectl create secret docker-registry harbor-registry-creds ^
    --docker-server=harbor.yourdomain.com ^
    --docker-username='username' ^
    --docker-password='PASSWORD' ^
    --docker-email='admin@local' ^
    -n ha-config
  ```

### 4. Deploy app qua ArgoCD + Helm

- [ ] Trong ArgoCD UI:  
  - [ ] Add repo chứa Helm chart của app.  
  - [ ] Create Application:  
    - [ ] Chọn repo, path/chart.  
    - [ ] Namespace: `ha-config`.  
    - [ ] Sync policy: manual hoặc auto.  
  - [ ] Bấm Sync để ArgoCD deploy app.

### 5. DNS cho app (NLB)

- [ ] Lấy ingress/NLB DNS:

  ```bash
  kubectl get ingress -A
  ```

  Tìm ingress của app hoặc controller, copy ADDRESS (NLB DNS).

- [ ] Cloudflare → DNS → Add CNAME:  
  - [ ] Name: ví dụ `haconfig.yourdomain.com`.  
  - [ ] Target: NLB DNS.  

- [ ] Truy cập `https://haconfig.yourdomain.com` → app phải chạy.

### 6. Cleanup (mỗi khi không dùng nữa, để giữ budget)

- [ ] Xoá cluster lab:

  ```bash
  eksctl delete cluster -f lab-cluster.yaml
  ```

  hoặc

  ```bash
  eksctl delete cluster --name lab-eks-cluster --region ap-southeast-1
  ```

- [ ] EC2 Console:  
  - [ ] Đảm bảo không còn EC2 instance, NLB/ALB, EBS volume, security group phát sinh mà không cần thiết.  
- [ ] IAM, ACM certificate, DNS record có thể giữ lại cho buổi sau (chi phí rất nhỏ).