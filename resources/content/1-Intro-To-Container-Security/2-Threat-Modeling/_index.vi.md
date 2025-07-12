---
title : "Mô hình mối đe doạ (Threat Modeling)"
date : 2025-07-07
weight : 2
chapter : false
pre : "<b> 1.2 </b>"
---

**Nội dung chính:**

- [Mô hình STRIDE là gì?](#mô-hình-stride-là-gì)
- [Xác định bề mặt tấn công container](#xác-định-bề-mặt-tấn-công-container)
- [Vẽ sơ đồ mô hình đe doạ](#vẽ-sơ-đồ-mô-hình-đe-doạ)
- [Thực hành mô hình hoá mối đe doạ với container app](#thực-hành-mô-hình-hoá-mối-đe-doạ-với-container-app)

---

## Mô hình STRIDE là gì?

**STRIDE** là một mô hình giúp xác định các loại mối đe doạ trong hệ thống. Nó gồm:

| Thành phần | Mối đe doạ                       |
|------------|----------------------------------|
| S          | Spoofing – Giả mạo danh tính     |
| T          | Tampering – Can thiệp dữ liệu    |
| R          | Repudiation – Chối bỏ hành vi    |
| I          | Information Disclosure – Rò rỉ TT|
| D          | Denial of Service – Từ chối dịch vụ |
| E          | Elevation of Privilege – Leo thang đặc quyền |

> 🧠 STRIDE giúp phân tích từng thành phần trong hệ thống để xác định lỗ hổng tiềm ẩn.

---

## Xác định bề mặt tấn công container

Khi triển khai ứng dụng container, cần xác định các điểm có thể bị khai thác:

- **Container Image:** chứa lỗ hổng (CVE), mã độc
- **Registry:** không được bảo vệ → dễ bị đẩy/pull trái phép
- **Kubernetes API:** bị tấn công qua RBAC sai
- **Pod chạy với root:** tăng rủi ro leo thang
- **Thông tin nhạy cảm:** hardcoded secret, ENV var

---

## Vẽ sơ đồ mô hình đe doạ

Bạn có thể dùng các công cụ sau để minh hoạ:

- ✏️ [Draw.io](https://draw.io)
- 🔐 [Microsoft Threat Modeling Tool](https://www.microsoft.com/en-us/security/blog/2020/11/18/threat-modeling-tool-updates/)

👉 Sơ đồ nên gồm:

- Người dùng (developer, attacker)
- Các thành phần: registry, CI/CD, cluster, pod, secrets, DB
- Ghi chú lỗ hổng tương ứng với STRIDE

📸 *Chụp ảnh sơ đồ threat modeling để minh hoạ cho tài liệu sau.*

---

## Thực hành mô hình hoá mối đe doạ với container app

1. Chọn một ứng dụng container hoá bạn sẽ triển khai ở lab sau (VD: nginx + flask + mongo)
2. Phân tích từng bước từ build → deploy → chạy → kết nối mạng
3. Liệt kê mối đe doạ với STRIDE và đặt biện pháp phòng chống

| Bước        | Mối đe doạ (STRIDE) | Phòng chống                            |
|-------------|----------------------|----------------------------------------|
| Push image  | Tampering            | Ký image, dùng registry riêng          |
| Chạy pod    | Elevation Privilege  | Không chạy với root, drop capabilities |
| Kết nối DB  | Information Disclosure | NetworkPolicy, không expose DB        |

---

📘 **Kết quả mong đợi:**
- Hiểu rõ các dạng đe doạ bảo mật container
- Tự tay mô hình hoá mối đe doạ cho ứng dụng bạn sắp triển khai
- Có sơ đồ threat modeling sẵn sàng dùng trong workshop

