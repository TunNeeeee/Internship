---

title: "Giám sát runtime với Falco (Windows)"
date: 2025-07-15
weight: 2
chapter: false
pre: "<b> 4.2 </b>"
-------------------

## Nội dung chính

* [Giới thiệu Falco](#giới-thiệu-falco)
* [Cài đặt Falco bằng Helm](#cài-đặt-falco-bằng-helm)
* [Kiểm tra cảnh báo runtime](#kiểm-tra-cảnh-báo-runtime)

---

## Giới thiệu Falco

**Falco** là công cụ giám sát hành vi thời gian thực trong Kubernetes. Nó giúp phát hiện:

* Truy cập bất thường đến hệ thống (vd: `/etc/passwd`)
* Sử dụng shell trong container (`bash`, `sh`)
* Ghi đè file nhạy cảm

Falco sử dụng kernel syscall để ghi nhận hành vi và phát cảnh báo dựa trên rule.

---

## Cài đặt Falco bằng Helm

### 🔧 Yêu cầu:
* Đã cài **kubectl**, **eksctl**, **AWS CLI**, và có cluster EKS hoạt động

### Bước 1: Cài đặt Helm

1. Truy cập trang [ https://github.com/helm/helm/releases/latest]( https://github.com/helm/helm/releases/latest)
2. Tải về file: `windows-amd64.zip`
3. Giải nén → Lấy file `helm.exe`
4. Tạo thư mục ví dụ: `C:\Program Files\helm\` → copy `helm.exe` vào
5. Thêm đường dẫn vào `Environment Variables > Path`
6. Mở PowerShell/CMD và kiểm tra:
Xác minh:

```bash
helm version
```

### Bước 2: Thêm repo và cài Falco:

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
kubectl create ns falco
helm install falco falcosecurity/falco -n falco
```

📸 *Chụp ảnh pod Falco đang chạy ở namespace `falco` để minh họa tài liệu.*

---

## Kiểm tra cảnh báo runtime

### Bước 1: Mở shell bất thường để tạo alert

```bash
kubectl run -i --tty attacker --image=alpine -- sh
```

Sau đó chạy trong terminal:

```sh
touch /etc/passwd
```

### Bước 2: Xem log Falco

```bash
kubectl logs -l app=falco -n falco
```

✅ Kết quả mong đợi:

```log
Falco Alert: Write below etc detected (user=root command=touch /etc/passwd)
```

📸 *Chụp log Falco có chứa dòng cảnh báo để đưa vào báo cáo.*

---

## Ghi chú

* Falco không ngăn chặn hành vi, chỉ cảnh báo.
* Muốn tự động phản ứng (4.3), bạn cần kết hợp với Falcosidekick hoặc các công cụ khác.

👉 **Tiếp theo:** [4.3 Tự động phản ứng với vi phạm bằng Falcosidekick →](../3-Automated-Remediation/)
