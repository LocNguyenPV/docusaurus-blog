# Meta-Arguments: Những "Siêu năng lực" ẩn giấu trong HCL

Ở bài trước, chúng ta đã làm quen với cấu trúc cơ bản của một Block trong HCL:
`resource "loại" "tên" { cấu_hình }`

Ví dụ, để tạo một máy ảo, bạn phải khai báo `ami`, `instance_type`... Đó là những **Arguments (Tham số)** riêng biệt của từng loại tài nguyên.

Ngoài ra, HCL còn cung cấp cho chúng ta một bộ công cụ quyền lực khác, có thể áp dụng cho **MỌI** loại resource, giúp thay đổi hành vi của chúng. Đó chính là **Meta-Arguments**.

Hôm nay, chúng ta sẽ cùng khám phá "Tứ đại quyền lực" của Meta-Arguments: `count`, `for_each`, `depends_on` và `lifecycle`.

![HCL four](./images/meta-args/image.png)

---

## Meta Arguments với Arguments

Hãy tưởng tượng:

- **Arguments:** "Tạo cho tôi máy ảo với HĐH linux version x.x trên AWS".
- **Meta-Arguments:**
  - Tạo cho tôi 5 cái máy ảo với HĐH linux version x.x trên AWS
  - Chỉ tạo máy ảo khi database tạo xong

## 1. `count`

Đây là **meta-argument** phổ biến nhất. Thay vì copy-paste code để tạo 5 cái máy ảo giống hệt nhau, bạn chỉ cần dùng `count`.

```hcl
# Tạo 3 con EC2 server tên là `web-0`, `web-1`, `web-2`.

resource "aws_instance" "web" {
  count         = 3 # <--- Meta-argument
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  tags = {
    # count.index sẽ chạy từ 0, 1, 2
    Name = "web-${count.index}"
  }
}

```

**Ưu điểm:** Nhanh, gọn.
**Nhược điểm:** `count` hoạt động dựa trên số thứ tự (index). Nếu bạn xóa con `web-1` (ở giữa), Terraform sẽ dồn con `web-2` xuống thành `web-1` và tạo lại con mới.

---

## 2. `for_each`

Ra đời sau `count` để khắc phục nhược điểm về index. `for_each` cho phép bạn lặp qua một **Map** (Danh sách Key-Value) hoặc một **Set** (Tập hợp chuỗi).

```hcl
# Tạo 3 EC2 server tên là `web-0`, `web-1`, `web-2`.
variable "server_name" {
  type    = set(string)
  default = ["web-0", "web-1", "web-2"]
}

resource "aws_instance" "web" {
  for_each = var.server_name # <--- Meta-argument
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  tags = {
    Name = each.key
  }
}
```

:::tip[Tại sao lại xịn hơn `count`]
Nếu bạn xóa "web-1" khỏi danh sách, Terraform chỉ xóa đúng instance "web-1". Những instances khác vẫn an toàn, không bị đổi index, không bị ảnh hưởng.
:::

---

## 3. `depends_on`

Trong thực tế việc triển khai một instance/service này phụ thuộc vào việc triển khai một instance/service khác là không hiếm. Để giải quyết bài toán này trong IaC, HCL cung cấp một siêu tham số là `depends_on`

```hcl
# Tạo Database (S3/RDS) trước, sau khi xong mới tạo EC2
resource "aws_s3_bucket" "database" {
  bucket = "my-app-data"
}

resource "aws_instance" "app" {
  ami           = "ami-12345"
  instance_type = "t2.micro"

  # Bắt buộc chờ S3 xong mới được chạy
  depends_on = [ aws_s3_bucket.database ]
}

```

---

## 4. `lifecycle`

Block `lifecycle` giúp bạn can thiệp vào vòng đời sinh-tử của tài nguyên. Nó có 3 quyền năng chính:

### a. `create_before_destroy` (An toàn là bạn)

Mặc định khi sửa đổi lớn (như đổi AMI), Terraform sẽ **Xóa cũ -> Tạo mới**. Điều này gây Downtime (chết trang web).
Dùng tùy chọn này, Terraform sẽ **Tạo mới -> Chạy ổn -> Rồi mới xóa cũ**.

```hcl
lifecycle {
  create_before_destroy = true
}

```

### b. `prevent_destroy` (Chống "tay nhanh hơn não")

Cực kỳ quan trọng cho Database. Nếu ai đó lỡ tay chạy `terraform destroy`, Terraform sẽ báo lỗi và từ chối xóa tài nguyên này.

```hcl
lifecycle {
  prevent_destroy = true
}

```

### c. `ignore_changes` (Làm ngơ sự thay đổi)

Dùng khi có một số thông số bị thay đổi bởi bên ngoài (ví dụ: Auto Scaling Group tự tăng số lượng máy, hoặc Tags do hệ thống khác gắn vào) và bạn không muốn Terraform reset lại nó mỗi khi chạy code.

```hcl
lifecycle {
  ignore_changes = [ tags, instance_count ]
}

```

---

## 5. `provider` (Bonus): Phân thân đa vùng

Dùng khi bạn muốn một file code nhưng tạo tài nguyên ở 2 vùng (Region) khác nhau (VD: 1 máy ở Mỹ, 1 máy ở Sing).

```hcl
# Provider mặc định (Singapore)
provider "aws" {
  region = "ap-southeast-1"
}

# Provider phụ (US) - dùng alias
provider "aws" {
  alias  = "us"
  region = "us-east-1"
}

# Tài nguyên dùng provider phụ
resource "aws_instance" "us_server" {
  provider      = aws.us # <--- Meta-argument trỏ về alias
  ami           = "ami-us-12345"
  instance_type = "t2.micro"
}

```

---

## Tổng kết

Meta-Arguments chính là thứ phân biệt giữa một người biết viết Terraform và một người **thành thạo** Terraform.

- Dùng `count`/`for_each` để code gọn gàng (DRY).
- Dùng `depends_on` để kiểm soát luồng chạy.
- Dùng `lifecycle` để bảo vệ hệ thống khỏi sai sót.

Hy vọng bài viết này giúp bạn nắm vững những "siêu tham số" này để áp dụng vào dự án thực tế. Ở bài tiếp theo, chúng ta sẽ cùng bắt tay vào thực hành!

**Happy Coding! 👨‍💻**
