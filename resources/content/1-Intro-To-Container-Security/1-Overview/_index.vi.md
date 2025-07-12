---
title : "Tổng quan về Bảo mật Container"
date : 2025-07-07
weight : 1
chapter : false
pre : "<b> 1.1 </b>"
---

**Nội dung chính:**

- [Container là gì và vì sao cần bảo mật?](#container-là-gì-và-vì-sao-cần-bảo-mật)
- [Các lớp bảo mật container](#các-lớp-bảo-mật-container)
- [Mục tiêu cuối cùng của workshop](#mục-tiêu-cuối-cùng-của-workshop)

---

## Container là gì và vì sao cần bảo mật?

Container là một phương pháp đóng gói ứng dụng kèm theo các thư viện và cấu hình để chạy được ở bất kỳ đâu. Tuy nhiên, do tính chất nhẹ và chia sẻ kernel nên container tiềm ẩn nhiều rủi ro bảo mật:

- Container có thể truy cập file hệ thống host nếu không được giới hạn đúng cách
- Tấn công “container escape” có thể ảnh hưởng toàn bộ node
- Lỗi cấu hình (exposed port, privilege mode) dễ bị khai thác
- Các image từ nguồn không xác thực có thể chứa mã độc

---

## Các lớp bảo mật container

Bảo mật container không chỉ ở runtime mà cần bảo vệ toàn bộ vòng đời ứng dụng:

| Giai đoạn     | Biện pháp                         |
|---------------|-----------------------------------|
| Build         | Quét image (Trivy), loại bỏ lỗ hổng |
| Deploy        | Kiểm soát chính sách (OPA, limit) |
| Runtime       | Phát hiện bất thường (Falco)     |
| Network       | Giới hạn truy cập (NetworkPolicy)|
| Monitoring    | Cảnh báo & phản hồi sự cố         |

---

## Mục tiêu cuối cùng của workshop

Cuối workshop, bạn sẽ:

- Cài đặt cluster EKS với ứng dụng container hoá
- Thực hiện quét lỗ hổng, kiểm tra benchmark (CIS)
- Áp dụng NetworkPolicy, OPA, Falco để bảo vệ container
- Triển khai hệ thống alerting và tự động hóa khắc phục
- Hiểu và áp dụng mô hình mối đe doạ (STRIDE, ATT&CK)

> 🧠 Bạn sẽ chụp ảnh lại từng bước và đưa vào nội dung workshop để hướng dẫn người khác triển khai môi trường bảo mật container toàn diện.

