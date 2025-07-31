---
title: "Tăng cường an ninh Container"
date: 2025-07-13
weight: 3
chapter: false
pre: "<b> 3. </b>"
-------------


## Tổng quan

Phần này tập trung vào **các biện pháp tăng cường bảo mật container** ở tầng cơ bản nhất gồm:

* Quét lỗ hổng container image (Trivy)
* Kiểm tra tuân thủ chuẩn bảo mật (CIS Benchmark với kube-bench)
* Áp dụng chính sách mạng để giới hạn traffic giữa các Pod (Kubernetes NetworkPolicy)

---
