---

title: "Áp dụng NetworkPolicy trong Kubernetes"
date: 2025-07-13
weight: 3
chapter: false
pre: "<b> 3.3 </b>"
-------------------

## Nội dung chính

* [Giới thiệu về NetworkPolicy](#giới-thiệu-về-networkpolicy)
* [Tạo namespace và Pod mẫu](#tạo-namespace-và-pod-mẫu)
* [Tạo và áp dụng NetworkPolicy](#tạo-và-áp-dụng-networkpolicy)
* [Kiểm tra kết quả áp dụng](#kiểm-tra-kết-quả-áp-dụng)

---

## Giới thiệu về NetworkPolicy

**NetworkPolicy** là tính năng của Kubernetes giúp bạn kiểm soát lưu lượng mạng vào/ra giữa các Pod.

Mặc định, mọi Pod trong một cluster có thể giao tiếp với nhau. Khi áp dụng NetworkPolicy, bạn có thể chặn mọi traffic và chỉ cho phép một số kết nối cụ thể.

---

## Tạo namespace và Pod mẫu

### Bước 1: Tạo namespace riêng

```bash
kubectl create ns secure-ns
```

### Bước 2: Tạo Pod mẫu (nginx)

```bash
kubectl run nginx --image=nginx -n secure-ns --expose --port=80
```

📌 Ghi chú: `--expose` sẽ tự tạo service để bạn dễ test truy cập.

---

## Tạo và áp dụng NetworkPolicy

### Bước 3: Tạo file `deny-all.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: secure-ns
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

Lệnh áp dụng:

```bash
kubectl apply -f deny-all.yaml
```

### Bước 4: Tạo một Pod test bên ngoài

```bash
kubectl run busybox --image=busybox:1.28 --rm -it --restart=Never -- sh
```

Thử `wget` vào nginx service:

```bash
wget -qO- nginx.secure-ns.svc.cluster.local
```

❌ Kết quả sẽ không truy cập được — đã bị chặn bởi NetworkPolicy.

---

## Kiểm tra kết quả áp dụng

Bạn có thể thử thêm các NetworkPolicy cho phép một số IP hoặc namespace cụ thể, ví dụ:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-ns
  namespace: secure-ns
spec:
  podSelector:
    matchLabels:
      run: nginx
  ingress:
  - from:
    - podSelector: {}
```

Sau đó thử truy cập lại từ Pod trong cùng namespace.

✅ Nếu thành công → bạn đã kiểm soát được traffic đúng như mong muốn.

---

## Ghi chú bổ sung

* Đảm bảo cluster đã bật CNI plugin hỗ trợ NetworkPolicy (Amazon EKS hỗ trợ sẵn)
* Sử dụng thêm công cụ như Calico nếu cần advanced policy
* Áp dụng NetworkPolicy là bước quan trọng trong mô hình Zero Trust cho Kubernetes
