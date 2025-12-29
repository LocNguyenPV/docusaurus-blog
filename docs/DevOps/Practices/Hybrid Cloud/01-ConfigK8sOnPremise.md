# Cài đặt và cấu hình K8s trên hệ thống on-premise

:::tip[TLDR;]

- Cấu hình cụm K8s (1 master và 2 worker) trên on-premise
- Cài **Ingress-NGINX Controller** trên cluster on-prem.
- Expose qua **Service type NodePort** để truy cập từ internet vào các service nội bộ.
- Test nhanh bằng một app demo (`hello-nginx`) để đảm bảo Ingress hoạt động đúng trước khi dùng cho Harbor, GitLab, Jenkins ở các phase sau.

:::

## Cài NGINX Ingress Controller trên on-prem K8s (Kubeadm)

Vì cụm này là Kubeadm trên VM (không có LoadBalancer managed như GKE), mô hình triển khai tương đương “bare-metal style”: NodePort + firewall GCP + External IP VM.

![alt text](./images/day01/image.png)

## 1. Cài đặt K8s

Xem lại [link](https://blog.codebyluke.io.vn/docs/DevOps/Kubernetes/deploy_onpremis)

## 2. Thiết kế Ingress cho Kubeadm on-prem

### 2.1. Mô hình mạng

Trên GCP:

- Project: `hybrid-devops-lab` (ví dụ).
- VPC: `devops-hybrid`.
- Subnet: `kubeadm-subnet` (10.0.0.0/20) cho 3 VM Kubeadm.
- Firewall rules:
  - `allow-internal`: cho phép 10.0.0.0/16.
  - `allow-ssh`: cho phép SSH từ IP cá nhân.
  - `allow-ingress-nginx-nodeport`: mở TCP 30131 (HTTP) và 32690 (HTTPS) ra ngoài.

Trên cụm Kubeadm:

- Pod network: Flannel với CIDR 10.244.0.0/16.
- Ingress-NGINX chạy trong namespace `ingress-nginx`.
- Service `ingress-nginx-controller`:
  - Type: NodePort.
  - 80 → 30131 (HTTP).
  - 443 → 32690 (HTTPS).

---

## 3. Các bước triển khai chính

### 3.1. Tạo namespace & cài Ingress-NGINX

Trên **node master**:

```bash
kubectl create namespace ingress-nginx
```

Cài Ingress-NGINX với profile “baremetal”/NodePort:

```bash
kubectl apply -f \
https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml
```

Kiểm tra:

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

Kết quả mong đợi:

```bash
NAME                      TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller  NodePort   10.100.111.50    <none>        80:30131/TCP,443:32690/TCP   ...
```

:::note[Node port]
Khi sử dụng `node port` để truy cập, port sẽ nằm trong khoảng **30000-32767**
:::

### 3.2. Mở firewall GCP cho NodePort

Mặc định firewall chỉ cho SSH, nên phải mở thêm NodePort:

```
gcloud compute firewall-rules create allow-ingress-nginx-nodeport \
  --network=devops-hybrid \
  --allow=tcp:30131,tcp:32690 \
  --source-ranges=0.0.0.0/0
```

:::danger
Trong môi trường production, nên giới hạn `source-ranges` về IP công ty
:::

Lấy External IP của VM:

```bash
gcloud compute instances list --project hybrid-devops-lab
```

Ghi lại `EXTERNAL_IP` của `kubeadm-master` (hoặc một worker).

### 3.3. Deploy app demo và Ingress rule

Tạo namespace cho app test:

```bash
kubectl create namespace ingress-test
```

Tạo deployment + service NGINX:

```bash
kubectl -n ingress-test create deployment hello-nginx --image=nginx
kubectl -n ingress-test expose deployment hello-nginx \
  --port=80 --target-port=80
```

Tạo Ingress:

```bash
cat <<EOF | kubectl -n ingress-test apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-nginx
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: hello.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-nginx
            port:
              number: 80
EOF
```

Trên máy local, sửa file `hosts`:

- Windows: `C:\Windows\System32\drivers\etc\hosts`
- Linux/macOS: `/etc/hosts`

Thêm:

```yaml
<EXTERNAL_IP_VM>    hello.local
```

Test:

- Trình duyệt: `http://hello.local:30131`
- Hoặc:

```bash
curl -H "Host: hello.local" http://<EXTERNAL_IP_VM>:30131
```

Nếu thấy trang mặc định của NGINX, Ingress đã hoạt động đúng.

---

## 4. Lỗi & cách sửa

### 4.1. Không truy cập được NodePort từ ngoài (timeout)

**Triệu chứng:**

- Truy cập `http://<EXTERNAL_IP>:30131` không phản hồi (connection timed out).

**Nguyên nhân:**

- Thiếu firewall rule cho NodePort trong VPC `devops-hybrid`.

**Cách sửa:**

- Tạo rule:

```
gcloud compute firewall-rules create allow-ingress-nginx-nodeport \
  --network=devops-hybrid \
  --allow=tcp:30131,tcp:32690 \
  --source-ranges=0.0.0.0/0
```

- Xác nhận rule xuất hiện, rồi test lại.

---

### 4.2. 404 Not Found khi truy cập Ingress

**Triệu chứng:**

- Truy cập được NodePort (không timeout), nhưng Ingress trả 404.

**Nguyên nhân thường gặp:**

1. **Sai Host header**

   - Ingress được cấu hình `host: hello.local`, nhưng bạn truy cập `http://<EXTERNAL_IP>:30131` mà không sửa hosts hoặc không gửi header `Host: hello.local`.
   - Controller không tìm thấy rule match → trả 404.

2. **Ingress backend không khớp với Service**
   - Ingress backend trỏ vào `name: hello-nginx`, `port: 80` nhưng service thật khác tên hoặc khác port.

**Cách sửa:**

- Đảm bảo file `hosts` map đúng:

```
<EXTERNAL_IP_VM>    hello.local
```

- Truy cập qua `http://hello.local:30131` hoặc dùng curl với header `Host: hello.local`.

- Kiểm tra Ingress:

```
kubectl describe ingress hello-nginx -n ingress-test
```

Xem phần:

```
Rules:
  Host  hello.local
    Path  /
      Backend: hello-nginx:80
```

- Kiểm tra service:

```
kubectl get svc -n ingress-test
```

Xem `NAME` và `PORT(S)` của service `hello-nginx` có đúng 80/TCP hay không.

Nếu sai, sửa lại YAML Ingress hoặc Service cho trùng nhau rồi `kubectl apply` lại.

---
