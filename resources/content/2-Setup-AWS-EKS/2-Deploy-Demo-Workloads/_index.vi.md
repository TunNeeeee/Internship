---
title : "Triển khai workload mẫu trên EKS"
date : 2025-07-07
weight : 2
chapter : false
pre : "<b> 2.2 </b>"
---

**Nội dung chính:**

- [Tạo namespace riêng cho demo](#tạo-namespace-riêng-cho-demo)
- [Triển khai ứng dụng nginx và backend](#triển-khai-ứng-dụng-nginx-và-backend)
- [Kiểm tra kết nối và chuẩn bị cho các lab tiếp theo](#kiểm-tra-kết-nối-và-chuẩn-bị-cho-các-lab-tiếp-theo)

---

## Tạo namespace riêng cho demo

```bash
kubectl create namespace demo
```

## Triển khai ứng dụng nginx và backend

Tạo file demo-app.yaml với nội dung:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: demo
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: backend
  namespace: demo
spec:
  containers:
    - name: backend
      image: hashicorp/http-echo
      args:
        - "-text=Hello from backend"
      ports:
        - containerPort: 5678

```

## Kiểm tra kết nối và chuẩn bị cho các lab tiếp theo

```bash
kubectl get pods -n demo
```
✅ Bạn sẽ thấy 2 pod: nginx, backend đang ở trạng thái Running

📸 Chụp ảnh output terminal để chèn vào phần tài liệu workshop.

Ghi chú:
Ứng dụng mẫu này sẽ được dùng trong các lab sau (NetworkPolicy, Falco, OPA...)

Bạn có thể expose nginx bằng LoadBalancer để truy cập từ bên ngoài (tuỳ chọn):
```bash
kubectl expose pod nginx --port=80 --type=LoadBalancer -n demo
```
