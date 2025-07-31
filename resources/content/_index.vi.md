---
title: "Củng cố bảo mật Container và bảo vệ khi chạy (Runtime Protection)"
weight: 3
---

## 🔐 Giới thiệu

Trong phần này, bạn sẽ tìm hiểu cách **củng cố (hardening)** môi trường container nhằm giảm thiểu rủi ro bảo mật, đồng thời áp dụng các kỹ thuật **giám sát và bảo vệ ở thời điểm runtime** để phát hiện và phản ứng kịp thời với các hành vi bất thường.

Khi ứng dụng đã chạy, container vẫn có thể là mục tiêu tấn công nếu không có lớp bảo vệ phù hợp. Do đó, việc kết hợp cả **biện pháp chủ động (hardening)** và **biện pháp phản ứng (runtime detection)** là cực kỳ quan trọng.

---

## 🧩 Nội dung bạn sẽ học

- Những cấu hình nên áp dụng để harden container và Kubernetes Pod (user, capabilities, readonlyRootFilesystem, seccomp, apparmor…)
- Sử dụng công cụ như **Trivy** để kiểm tra lỗ hổng trước khi triển khai
- Cài đặt và cấu hình **Falco** để phát hiện các hành vi bất thường trong container
- Mô phỏng và phân tích các cuộc tấn công thời gian thực
- Áp dụng các policy runtime để giảm thiểu rủi ro khi container bị khai thác

---

## 🎯 Kết quả mong đợi

Sau khi hoàn thành phần này, bạn sẽ:

- Biết cách thiết lập cấu hình bảo mật chuẩn cho container/POD  
- Hiểu vai trò của các công cụ giám sát và phát hiện như Falco  
- Có khả năng triển khai hệ thống bảo vệ runtime trong môi trường thực tế  
- Nhận diện sớm các hành vi đáng ngờ và đưa ra phản ứng phù hợp

---

🚀 *Hãy sẵn sàng củng cố môi trường container của bạn trước các mối đe doạ thực tế!*
