# Bài 2: Phân tích kỹ thuật - Chọn "vũ khí" tối ưu cho bài toán Hybrid-Cloud

Ở bài trước, chúng ta đã có cái nhìn tổng quan về mô hình Hybrid-Cloud. Tuy nhiên, đứng trước hàng trăm công cụ DevOps hiện nay, việc lựa chọn một "Stack" công nghệ vừa mạnh mẽ, vừa tiết kiệm, lại phải ăn khớp với nhau là một bài toán không hề đơn giản.

Trong bài viết này, mình sẽ cùng các bạn "mổ xẻ" lý do tại sao bộ Stack này lại được chọn để xây dựng hệ thống "bất tử".

---

### 1. Jenkins: Sự linh hoạt của một "Lão làng"

Nhiều bạn sẽ hỏi: *"Tại sao không dùng GitLab CI hay GitHub Actions cho hiện đại?"*

Câu trả lời nằm ở **khả năng kiểm soát**. Trong mô hình Hybrid-Cloud, mình cần một công cụ có thể kết nối mượt mà giữa môi trường Cloud và Lab On-premise.

* **Manual Approval:** Tính năng `input` của Jenkins (kết hợp với Role-based Strategy) cho phép mình tạo ra một chốt chặn QA thực thụ. Bản build chỉ được lên GKE khi có sự xác nhận của đúng người có thẩm quyền.
* **Tùy biến không giới hạn:** Với Scripted Pipeline (Groovy), mình có thể viết bất cứ logic phức tạp nào để xử lý việc đẩy Image và cập nhật Manifest cho nhiều cụm K8s cùng lúc.

### 2. ArgoCD: "Linh hồn" của triết lý GitOps

Thay vì dùng `kubectl apply` một cách thủ công và rời rạc, mình chọn **ArgoCD**. Tại sao?

* **Trạng thái mong muốn (Desired State):** ArgoCD đảm bảo rằng những gì khai báo trên Git là những gì đang chạy trên cả GKE và On-premise.
* **Self-healing:** Nếu ai đó lỡ tay sửa cấu hình trực tiếp trên cụm K8s, ArgoCD sẽ ngay lập tức phát hiện và "đè" cấu hình chuẩn từ Git về. Đây là chìa khóa để giữ sự đồng nhất giữa hai môi trường Cloud và Local.

### 3. Uptime Kuma: Đơn giản là sức mạnh

Để giám sát sự cố (Monitoring), mình không chọn Prometheus hay Grafana vì chúng quá cồng kềnh cho mục đích Failover.

* **Nhẹ và nhanh:** Cài đặt chỉ mất vài phút bằng Docker.
* **Native Webhook:** Uptime Kuma hỗ trợ gửi Webhook "cực nhạy". Ngay khi GKE có dấu hiệu "hụt hơi", nó sẽ lập tức kích hoạt chuỗi phản ứng dây chuyền để chuyển hướng traffic mà không cần cấu hình lằng nhằng.

### 4. Cloudflare API & Cloud Run: "Bộ não" điều hướng

Đây là phần thú vị nhất. Thay vì mua các gói Load Balancer Global đắt đỏ, mình sử dụng:

* **Cloudflare DNS:** Tận dụng tốc độ cập nhật DNS cực nhanh và API mạnh mẽ.
* **Serverless (Cloud Functions/Cloud Run):** Mình viết một script Python nhỏ để làm "trạm trung chuyển". Script này chỉ thức dậy khi nhận được tín hiệu từ Uptime Kuma, thực hiện đổi IP trên Cloudflare rồi lại đi ngủ.
* **Chi phí:** Gần như bằng **0**.

---

### 💡 Tư duy chọn công cụ: "Đúng hơn là Tốt nhất"

Trong DevOps, không có công cụ tốt nhất, chỉ có công cụ phù hợp nhất với bài toán.

* Bạn cần **an toàn**? Hãy dùng Jenkins với QA Approval.
* Bạn cần **đồng bộ**? Hãy dùng ArgoCD.
* Bạn cần **tiết kiệm**? Hãy dùng Serverless để Failover.

Việc kết hợp những mảnh ghép này lại với nhau tạo nên một hệ thống không chỉ hoạt động tốt mà còn cực kỳ tối ưu về mặt vận hành và ngân sách.

---

### Kết luận

Lựa chọn xong "vũ khí" là chúng ta đã đi được 50% chặng đường. Ở bài viết tiếp theo, mình sẽ cùng các bạn bắt tay vào xây dựng **"Xương sống" của hệ thống - Jenkins Pipeline**. Chúng ta sẽ cùng viết code để biến quy trình từ Build, Test đến bước phê duyệt của QA trở nên tự động và chuyên nghiệp nhất.

Hẹn gặp lại các bạn ở Bài 3!

---

**Gợi ý cho Phong:**

1. **Hình ảnh:** Ở bài này, Phong nên có một bảng so sánh hoặc một tấm hình chụp các Logo công cụ (Jenkins, ArgoCD, GKE, Uptime Kuma, Cloudflare) liên kết với nhau.
2. **Meme:** Phong có thể chèn thêm một cái Meme về việc "Chọn công cụ phù hợp" để bài viết bớt căng thẳng.
3. **Tiếp theo:** Phong có muốn mình chuẩn bị nội dung cho **Bài 3: Jenkins Pipeline chi tiết** (có kèm code mẫu QA Approval và Noti Telegram) không?

Bạn thấy nội dung bài 2 này đã đủ "đô" về mặt kỹ thuật chưa?