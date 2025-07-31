---
title: "Quét lỗ hổng container image bằng Trivy (qua Docker)"
date: 2025-07-13
weight: 1
chapter: false
pre: "<b> 3.1 </b>"
---

### Nội dung chính

- [Giới thiệu Trivy](#giới-thiệu-trivy)
- [Cài đặt và sử dụng Trivy với Docker trên Windows](#cài-đặt-và-sử-dụng-trivy-với-docker-trên-windows)
- [Quét image Docker và phân tích kết quả](#quét-image-docker-và-phân-tích-kết-quả)
- [Ghi chú và Best Practice](#ghi-chú-và-best-practice)

---

### Giới thiệu Trivy

**Trivy** là công cụ mã nguồn mở hỗ trợ quét:

- Lỗ hổng bảo mật trong container image (CVE)
- Lỗi cấu hình trong IaC như Kubernetes YAML, Terraform, v.v.
- Hỗ trợ định dạng đầu ra chuẩn: table, JSON, SARIF...

---

### Cài đặt và sử dụng Trivy với Docker trên Windows

#### ✅ Điều kiện

- Đã cài đặt [Docker Desktop cho Windows](https://www.docker.com/products/docker-desktop/) (WSL2 hoặc Hyper-V)
- Đảm bảo Docker đang chạy

---

#### 🐳 Chạy Trivy trực tiếp từ Docker

Câu lệnh mẫu (quét image `nginx`):

```powershell
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock `
    -v $env:USERPROFILE\.trivy-cache:/root/.cache/ `
    aquasec/trivy:latest image nginx
```
💡 Giải thích:
- --rm: xóa container sau khi chạy xong
- -v /var/run/docker.sock:/var/run/docker.sock: cho phép Trivy truy cập image trên máy host
- -v $env:USERPROFILE\.trivy-cache:/root/.cache/: mount thư mục cache để tiết kiệm thời gian quét lần sau
- aquasec/trivy:latest: sử dụng image chính thức từ Docker Hub
- image nginx: chỉ định lệnh và tên image cần quét

🖼 Ví dụ thực tế với image có lỗ hổng
```powershell
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock `
    -v $env:USERPROFILE\.trivy-cache:/root/.cache/ `
    aquasec/trivy:latest image nginxinc/nginx-unprivileged:1.24.0-alpine-slim
```
![Image-Scanning-With-Trivy](/images/3.1/3.1.1.png)
### Quét image Docker và phân tích kết quả
Sau khi chạy lệnh, bạn sẽ nhận được kết quả dạng bảng:
Total: 27 (UNKNOWN: 2, LOW: 2, MEDIUM: 20, HIGH: 3)

| Library     | Vulnerability   | Severity | Installed  | Fixed       | Title                                                      |
|-------------|------------------|----------|------------|-------------|------------------------------------------------------------|
| libcrypto3  | CVE-2024-6119    | HIGH     | 3.1.4-r6   | 3.1.7-r0    | openssl: DoS trong X.509 name checks                       |
| nginx       | CVE-2023-44487   | HIGH     | 1.24.0-r1  | 1.24.0-r7   | HTTP/2 vulnerability dẫn đến tấn công DDoS                |
| busybox     | CVE-2023-42366   | MEDIUM   | 1.36.1-r5  | 1.36.1-r6   | Heap-buffer-overflow trong busybox awk                    |

⚠️ Dù đây là image slim và phổ biến, nhưng vẫn tồn tại nhiều lỗ hổng bảo mật (HIGH, MEDIUM). Điều này cho thấy việc quét định kỳ là rất cần thiết.

Khi chạy lệnh quét, bạn sẽ nhận được bảng chi tiết các gói bị ảnh hưởng, cấp độ nghiêm trọng (Severity), và phiên bản đã vá.
Ví dụ phân tích:
- CVE-2024-6119 (HIGH): ảnh hưởng đến openssl gây từ chối dịch vụ (DoS)
- CVE-2023-44487 (HIGH): ảnh hưởng HTTP/2, có thể bị DDoS
- CVE-2023-42366 (MEDIUM): lỗi tràn bộ đệm (heap buffer overflow)

✅ Cách xử lý
- Luôn ưu tiên sửa lỗi CRITICAL và HIGH
- Nếu có thể, dùng phiên bản mới hơn của image (1.24.0-r7, 1.25.x, ...)
- Tránh dùng :latest, nên cố định phiên bản để kiểm soát an toàn

### Ghi chú và Best Practice
Xuất file kết quả:
```powershell
docker run --rm `
  -v /var/run/docker.sock:/var/run/docker.sock `
  -v "$env:USERPROFILE/.trivy-cache:/root/.cache" `
  -v "$PWD:/root/report" `
  aquasec/trivy:latest image -f json -o /root/report/result.json nginx
``` 

➡️ Kết quả JSON sẽ lưu tại D:\pj_aws\result.json

#### Tích hợp CI/CD:
Bạn có thể tích hợp lệnh quét trên vào:
- GitHub Actions
- GitLab CI/CD
- Jenkins, CircleCI...

✅ Giúp kiểm tra bảo mật tự động mỗi lần build hoặc deploy.

🎯 Kết luận: Trivy là công cụ không thể thiếu khi triển khai container production. Bạn nên quét image thường xuyên, sử dụng image chính thức, giữ bản cập nhật, và tích hợp CI/CD để đảm bảo an toàn cho hệ thống.