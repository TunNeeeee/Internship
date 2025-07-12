---
title: "Khởi tạo Cluster EKS trên AWS"
date: 2025-07-01
weight: 2
chapter: true
pre: "<b> 2. </b>"
---

## Tổng quan

Trong chương này, bạn sẽ học cách **tạo môi trường Kubernetes thực tế trên AWS bằng Amazon EKS**. Đây là bước bắt buộc để triển khai các biện pháp bảo mật container trong các phần sau.

---
## 🎯 Mục tiêu của chương này

Trong chương này, bạn sẽ học cách **tạo môi trường Kubernetes thực tế trên AWS bằng Amazon EKS**. Đây là bước bắt buộc để triển khai các biện pháp bảo mật container trong các phần sau.

---

## 📚 Nội dung chính

- **2.1 Tạo EKS Cluster**  
  Hướng dẫn từng bước tạo một cluster EKS bằng `eksctl`, bao gồm IAM role, VPC, node group.

- **2.2 Triển khai workload mẫu**  
  Deploy một ứng dụng mẫu (nginx + backend) lên EKS để làm môi trường test cho các lab tiếp theo.

---

## ✅ Kết quả sau chương này

- Tạo được 1 cluster Kubernetes thực tế chạy trên AWS
- Cấu hình đầy đủ quyền IAM và kết nối với `kubectl`
- Deploy thành công ứng dụng mẫu để dùng cho các lab bảo mật sau

---

🚀 *Hãy bắt đầu với phần 2.1 – Tạo EKS Cluster nhé!*