# Bài 1: Xây dựng hệ thống Hybrid-Cloud "bất tử" - Tại sao và Như thế nào?

### Mở đầu: Cơn ác mộng lúc 2 giờ sáng

Hãy tưởng tượng bạn đang vận hành một ứng dụng thương mại điện tử trên Google Kubernetes Engine (GKE). Mọi thứ đều hoàn hảo cho đến một ngày, khu vực (Region) của Cloud gặp sự cố, hoặc đơn giản là hóa đơn cuối tháng "nhảy số" khiến bạn giật mình. 

![meme](image-1.png)

Đó chính là lý do mình bắt tay vào xây dựng hệ thống **Hybrid-Cloud**. Trong series này, mình sẽ cùng bạn đi qua hành trình từ việc đặt những viên gạch hạ tầng đầu tiên (On-premise & Cloud), xây dựng một Pipeline chuẩn chỉnh, cho đến khi hoàn thiện hệ thống có khả năng tự động chuyển vùng sự cố (Failover) mà không cần sự can thiệp của con người.

---

### 1. Hybrid-Cloud là gì? Tại sao phải làm khó mình?

Hybrid-Cloud không đơn thuần là việc "vứt" mỗi thứ một nơi. Đó là sự kết hợp nhuần nhuyễn giữa:

* **Public Cloud (GKE):** Nơi đặt các service ưu tiên sự ổn định, tốc độ và khả năng scale không giới hạn.
* **On-premise (Local Lab):** Nơi tận dụng tài nguyên có sẵn để làm môi trường staging hoặc làm "phao cứu sinh" khi Cloud gặp sự cố.

**Tại sao mình lại chọn mô hình này?**

1. **Tối ưu chi phí (Cost Optimization):** Chạy 2 cụm K8s trên Cloud để dự phòng là một sự xa xỉ. Việc dùng cụm On-premise làm Backup giúp mình tiết kiệm ít nhất 50% chi phí hạ tầng dự phòng.

2. **Độ sẵn sàng cao (High Availability):** Không có gì là tuyệt đối, kể cả cloud. Hệ thống này  giúp ứng dụng "bất tử" nếu chẳng may có sự cố ở phía cloud

3. **Làm chủ công nghệ:** Đây là bài test thực tế nhất cho kỹ năng điều phối (Orchestration) và tự động hóa (Automation).

---

### 2. Kiến trúc tổng thể: "Bộ não" và "Cánh tay"

Để hệ thống vận hành trơn tru, mình thiết kế luồng hoạt động dựa trên 3 trụ cột chính: **GitOps, Monitoring và Serverless Failover.**

#### Sơ đồ luồng hoạt động:

1. **Luồng CI/CD:** Dev push Code lên GitLab, Gilab trigger **Jenkins** qua webhook để build pipeline 
![alt text](image-5.png)

và update image trong harbor registry và update manifest on-premise, sau đó thông báo tới telegram cho QA vào test, sau khi QA test xong sẽ nhấn approve để push lên Artifact Registry

Code từ Gitlab -> **Jenkins** (xử lý build/test và đợi QA Approve) -> Cập nhật Manifest vào Git.
2. **Luồng GitOps:** **ArgoCD** (được cài ở cả GKE và On-premise) sẽ "soi" Git và đồng bộ ứng dụng lên cả 2 cụm.
3. **Luồng Failover (Điểm nhấn):**
* **Uptime Kuma** liên tục kiểm tra "nhịp tim" của GKE.
* Nếu GKE Down, Uptime Kuma bắn Webhook sang **Google Cloud Function**.
* Cloud Function gọi API của **Cloudflare** để bẻ hướng Traffic từ IP GKE sang IP On-premise.



> *[Chèn Sơ đồ kiến trúc tại đây: Vẽ luồng từ Cloudflare -> GKE/On-premise, có Uptime Kuma đứng ngoài]*

---

### 3. Bộ Stack công nghệ "Thực chiến"

Trong series này, chúng ta sẽ không nói lý thuyết suông. Mình sẽ cùng các bạn cài đặt và cấu hình:

* **Hạ tầng:** Google Kubernetes Engine (GKE) & On-premise K8s (VM/Physical).
* **Điều phối (Orchestration):** ArgoCD - Trái tim của mô hình GitOps.
* **Tự động hóa:** Jenkins Pipeline với bước QA Confirmation (Manual Approval) cực kỳ thực tế.
* **Giám sát:** Uptime Kuma - Giao diện trực quan, nhẹ nhàng nhưng đầy đủ võ thuật.
* **Chuyển vùng:** Python Script chạy trên Cloud Run/Functions kết hợp Cloudflare API.

---

### 4. Chúng ta sẽ đạt được gì sau series này?

Kết thúc hành trình này, bạn sẽ có trong tay:

* Một Pipeline CI/CD chuẩn chỉnh, có sự tham gia của con người (QA) và phân quyền chặt chẽ (RBAC).
* Khả năng quản lý đa cụm (Multi-cluster) bằng ArgoCD.
* Một hệ thống giám sát biết "nói" và biết "hành động" khi có sự cố xảy ra.
* Và quan trọng nhất, là tư duy giải quyết vấn đề của một kỹ sư DevOps thực thụ.

---

### Kết luận

Hệ thống Hybrid-Cloud không chỉ là một bài Lab, nó là giải pháp cho bài toán kinh tế và kỹ thuật hiện đại. Ở bài viết tiếp theo, mình sẽ đi sâu vào **Phân tích kỹ thuật và lựa chọn công cụ** - tại sao mình lại chọn Jenkins thay vì GitLab CI, hay tại sao Cloudflare lại là "chìa khóa" cho việc điều hướng traffic?

Cùng chờ đón nhé!
