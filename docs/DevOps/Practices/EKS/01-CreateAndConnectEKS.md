Day 1: Làm quen AWS và tạo cụm EKS đầu tiên (dành cho người mới 100%)

---

## Mở đầu: Vì sao nên bắt đầu với EKS?

Nếu bạn là developer hoặc engineer muốn bước chân vào thế giới cloud, đặc biệt là Kubernetes trên AWS, rất dễ bị “ngợp” bởi quá nhiều dịch vụ và khái niệm mới. Day 1 này được thiết kế cho đúng đối tượng “chưa biết gì về AWS”, giúp bạn đi trọn một vòng: từ tạo tài khoản, chuẩn bị môi trường, đến việc nhìn thấy cụm EKS đầu tiên với lệnh `kubectl get nodes`.

Mục tiêu cuối ngày: bạn đăng nhập được vào AWS bằng IAM user riêng, cấu hình xong AWS CLI trên máy, và tạo được một EKS cluster lab chạy được.

![alt text](image.png)

---

## Bước 1: Tạo tài khoản và user riêng để không “chơi” bằng root

Sau khi đăng ký tài khoản AWS, phần lớn người mới sẽ có xu hướng dùng luôn tài khoản root (email + password đăng ký ban đầu). Đây là thói quen cực kỳ xấu trong môi trường production, nhưng ngay từ lab bạn cũng nên tập tư duy đúng.

Ý tưởng chuẩn:

- Tài khoản root: dùng rất ít, chỉ cho việc quản trị cấp cao.
- Tài khoản IAM user: dùng hằng ngày để thao tác (kể cả lab).

Ngay khi đăng nhập lần đầu:

- Bật MFA cho root để bảo vệ tài khoản gốc.
- Tạo IAM user, ví dụ `lab-admin`, cho phép đăng nhập console, và gán quyền `AdministratorAccess` để giảm ma sát khi học, đồng thời ghi rõ trong tài liệu đây là “lab only”.

![alt text](image-4.png)
![alt text](image-5.png)
![alt text](image-6.png)


---

## Bước 2: Tạo Access Key và kết nối máy của bạn với AWS

Để dùng AWS qua command line (AWS CLI, eksctl, Terraform, v.v.), bạn cần một “chiếc chìa khóa” gồm:

- Access Key ID
- Secret Access Key

Chìa khóa này gắn với IAM user `lab-admin`, và máy cá nhân sẽ dùng nó để “tự giới thiệu” với AWS.

Tư duy:

- Máy ngoài AWS → cần Access Key.
- Máy trong AWS (EC2, Lambda) → ưu tiên IAM Role, không hard-code Access Key.

Quy trình:

- Mở IAM → Users → chọn `lab-admin`.
- Tab “Security credentials” → tạo Access Key cho mục đích CLI.
- Ghi lại Access Key ID và Secret Access Key vào nơi an toàn, không commit lên git, không gửi qua chat công khai.

![alt text](image-1.png)
![alt text](image-2.png)
![alt text](image-3.png)

---

## Bước 3: Chuẩn bị công cụ trên máy – AWS CLI, kubectl, eksctl

Giờ bạn biến máy cá nhân thành “trạm điều khiển” cho AWS:

- AWS CLI v2: để dùng lệnh `aws ...`.
- `kubectl`: để giao tiếp với Kubernetes cluster.
- `eksctl`: để tạo và quản lý EKS cluster bằng lệnh, thay vì click tay.

Pattern triển khai:

- Tạo thư mục `C:\tool`.
- Đưa `kubectl.exe` và `eksctl.exe` vào `C:\tool`.
- Thêm `C:\tool` vào biến môi trường PATH (System Properties → Advanced → Environment Variables).

Sau khi cài:

- Chạy `aws --version` để kiểm tra CLI.
- Chạy `kubectl version --client` để kiểm tra kubectl.
- Chạy `eksctl version` để kiểm tra eksctl.

> Gợi ý hình:
>
> - Hình 5: Wizard cài AWS CLI (màn hình “Completed”).
> - Hình 6: Cửa sổ Environment Variables → PATH có `C:\tool`.
> - Hình 7: Terminal hiển thị kết quả `aws --version`, `kubectl version --client`, `eksctl version`.

---

## Bước 4: Cấu hình AWS CLI – để lệnh của bạn “biết” đang là ai

AWS CLI hiện tại mới chỉ là “cái vỏ”. Bạn cần:

- Gắn nó với IAM user `lab-admin` thông qua Access Key.
- Chọn region mặc định (lab dùng `ap-southeast-1`).
- Chọn định dạng output (json).

Chỉ cần lệnh:

```bash
aws configure
```

Và nhập:

- AWS Access Key ID
- AWS Secret Access Key
- Default region name: `ap-southeast-1`
- Default output format: `json`

Sau đó, test:

```bash
aws sts get-caller-identity
```

Nếu trả về một JSON có Account, Arn là bạn đã kết nối thành công.

> Gợi ý hình:
>
> - Hình 8: Terminal lúc nhập `aws configure` (các prompt 4 dòng).
> - Hình 9: Terminal với output `aws sts get-caller-identity`.

---

## Bước 5: Thiết kế quyền cho EKS – IAM Role cho control plane và worker

EKS là Kubernetes được AWS quản lý, nên control plane và worker cần quyền để “gọi” các dịch vụ AWS khác (EC2, VPC, ELB, IAM, EBS,…).

Bạn sẽ tạo 2 role:

1. **Role cho EKS control plane (Cluster Role)**

   - Use case: “EKS - Cluster”.
   - Policy: `AmazonEKSClusterPolicy`.

2. **Role cho worker node (Node Role)**
   - Service: EC2.
   - Policies:
     - `AmazonEKSWorkerNodePolicy`
     - `AmazonEC2ContainerRegistryReadOnly`
     - `AmazonEKS_CNI_Policy`

Sau khi tạo, hãy ghi lại ARN của 2 role này để dùng trong file cấu hình cluster.

> Gợi ý hình:
>
> - Hình 10: Màn hình IAM → Create role, chọn service EKS, use case “EKS - Cluster”.
> - Hình 11: Màn hình Add permissions cho worker role với 3 policy đã tick.

---

## Bước 6: Tạo key pair EC2 – “chìa khóa SSH” vào node

Để sau này có thể SSH vào worker node (debug, kiểm tra cấu hình), bạn nên tạo key pair:

- EC2 → Key Pairs → Create key pair.
- Name: `lab-key`.
- Key pair type: RSA.
- Private key file format: `.pem`.
- Lưu file `.pem` trên máy (không chia sẻ).

Key pair này sẽ được tham chiếu trong phần cấu hình node group của EKS.

> Gợi ý hình:
>
> - Hình 12: Màn hình “Create key pair” (Name, Type, Format).

---

## Bước 7: Viết file cấu hình EKS với eksctl

Thay vì click tay từng bước, bạn mô tả cluster bằng YAML (hạ tầng-as-code). Ví dụ file `lab-cluster.yaml` được tổ chức như sau:

- **metadata**:

  - `name`: tên cluster (ví dụ `lab-eks-cluster`).
  - `region`: `ap-southeast-1`.
  - `version`: version Kubernetes.

- **vpc**:

  - `nat.gateway: Disable` để tiết kiệm chi phí (lab only).
  - `clusterEndpoints.publicAccess: true`, `privateAccess: false`.

- **iam**:

  - `serviceRoleARN`: ARN role cluster đã tạo.
  - `withOIDC: true`.

- **addons**:

  - `vpc-cni`, `coredns`, `kube-proxy`, `metrics-server`.

- **managedNodeGroups**:
  - name: `student-workers`.
  - instanceType: một loại nhỏ/rẻ (ví dụ `t3.medium` hoặc `c7i-flex.large`).
  - amiFamily: `Ubuntu2404`.
  - minSize, maxSize, desiredCapacity.
  - ssh.allow: `true`, ssh.publicKeyName: `lab-key`.
  - iam.instanceRoleARN: ARN role worker.

> Gợi ý hình:
>
> - Hình 13: Screenshot VS Code/IDE với file `lab-cluster.yaml`, highlight/box đỏ:
>   - ARN role cluster
>   - ARN role worker
>   - tên key pair
>   - dòng `nat.gateway: Disable` kèm comment “LAB ONLY”.

---

## Bước 8: Tạo cluster – khoảnh khắc “kubectl get nodes”

Giờ là lúc thực sự “bật” cluster:

1. Mở terminal trong thư mục chứa `lab-cluster.yaml`.
2. Kiểm tra lại:

   ```bash
   aws sts get-caller-identity
   ```

3. Chạy:

   ```bash
   eksctl create cluster -f lab-cluster.yaml
   ```

4. Đợi khoảng 15–20 phút. Bạn sẽ thấy log như:

   - tạo CloudFormation stack
   - tạo VPC
   - tạo nodegroup
   - tạo addons
   - cluster ready

5. Khi lệnh hoàn tất, chạy:

   ```bash
   kubectl get nodes
   ```

Nếu bạn thấy một hoặc vài dòng node với STATUS = `Ready`, nghĩa là bạn đã tạo thành công cụm EKS đầu tiên của mình.

> Gợi ý hình:
>
> - Hình 14: Terminal hiển thị log `eksctl create cluster -f lab-cluster.yaml` (đang tạo stack, nodegroup, addons,…).
> - Hình 15: Terminal với `kubectl get nodes` và 1–2 node `Ready`.

---

## Kết thúc Day 1: Tập thói quen “bật lên rồi tắt”

Vì đây là lab và budget giới hạn, đừng quên:

- Khi không học nữa, xoá cluster:

  ```bash
  eksctl delete cluster -f lab-cluster.yaml
  ```

- Thi thoảng vào EC2 Console kiểm tra xem còn instance, load balancer, volume “rác” nào chạy không.

Thành quả Day 1:

- Bạn đã biết cách dùng IAM user thay vì root.
- Máy local có thể nói chuyện với AWS qua AWS CLI.
- Bạn hiểu cơ bản IAM Role cho EKS control plane và worker.
- Bạn viết được file yaml để mô tả cluster.
- Bạn thấy được node EKS thật sự đang chạy.

---

## Image Checklist cho Day 1

Để bài blog trực quan và “có chứng cứ”, bạn có thể chuẩn bị các hình sau:

1. **Sơ đồ tổng quan kiến trúc lab**

   - Laptop → AWS → EKS (control plane + node group).

2. **IAM – Create user**

   - Màn hình tạo IAM user `lab-admin`, tick “Provide user access to AWS Management Console”.

3. **Gán AdministratorAccess cho user**

   - Màn hình chọn policy `AdministratorAccess`.

4. **Tạo Access Key**

   - Tab “Security credentials” của user, phần “Create access key”.
   - Màn hình hiển thị Access Key (che giá trị thật).

5. **Cài AWS CLI**

   - Wizard “Completed the AWS Command Line Interface v2 Setup”.

6. **PATH cho C:\tool**

   - Cửa sổ Environment Variables → PATH có `C:\tool`.

7. **Kiểm tra tool**

   - Terminal hiển thị `aws --version`, `kubectl version --client`, `eksctl version`.

8. **aws configure**

   - Terminal trong lúc nhập 4 dòng cấu hình.

9. **aws sts get-caller-identity**

   - JSON hiển thị Account, Arn (che số nếu muốn).

10. **IAM Role – EKS Cluster**

    - Màn hình Create role chọn service EKS, use case “EKS - Cluster”.

11. **IAM Role – Worker node**

    - Màn hình Add permissions với 3 policy worker đã tick.

12. **EC2 Key Pair**

    - Màn hình “Create key pair” với Name, Type = RSA, Format = .pem.

13. **File YAML trong editor**

    - VS Code/IDE mở `lab-cluster.yaml`, highlight các chỗ cần sửa (Account ID, ARN, key,…).

14. **eksctl create cluster**

    - Terminal chạy lệnh, log đang tạo CloudFormation stack, nodegroup, addons.

15. **kubectl get nodes**
    - Terminal với danh sách node, STATUS = Ready.

Chỉ cần bạn tự chụp lại các bước mình làm thực tế theo checklist này, bài blog sẽ vừa dễ theo dõi vừa tạo được niềm tin mạnh mẽ cho người mới bắt đầu.
