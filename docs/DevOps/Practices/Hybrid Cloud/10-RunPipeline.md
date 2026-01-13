:::tip[Lưu ý cho Custom Domain]

Nếu pipeline không thể nhìn thấy và sử dụng các `credentials` trong **Custom Domain**, bạn nên kiểm tra lại cấu hình **Specification** khi tạo Domain, nếu có cấu hình `HostName` thì nên đảm bảo đúng với Domain của bạn. Ví dụ:

- **Specification:** Hostname
- **Include:** `*.codebyluke.io.vn` (hoặc liệt kê cụ thể: `git.codebyluke.io.vn`, `registry.codebyluke.io.vn`).

:::
