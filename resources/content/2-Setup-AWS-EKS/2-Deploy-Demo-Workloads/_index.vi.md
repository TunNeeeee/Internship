---

title : "Triển khai workload mẫu trên EKS"
date : 2025-07-07
weight : 2
chapter : false
pre : "<b> 2.2 </b>"
---

### Nội dung chính

* [Tạo namespace riêng cho demo](#tạo-namespace-riêng-cho-demo)
* [Triển khai ứng dụng nginx và backend](#triển-khai-ứng-dụng-nginx-và-backend)
* [Kiểm tra kết nối và chuẩn bị cho các lab tiếp theo](#kiểm-tra-kết-nối-và-chuẩn-bị-cho-các-lab-tiếp-theo)

---

### Tạo namespace riêng cho demo

Namespace giúp cô lập tài nguyên giữa các ứng dụng khác nhau. Đầu tiên, tạo một namespace tên là `demo`:

```bash
kubectl create namespace demo
```

📸 Kết quả hiển thị:

![Create Namespace Demo](/images/2.2/2.2.1.png)

Sau đó kiểm tra lại:

```bash
kubectl get ns
```

📸 Kết quả:

![kubectl get ns](/images/2.2/2.2.2.png)

---

### Triển khai ứng dụng nginx và backend

Ứng dụng mẫu gồm 2 Pod:

* **nginx** (dùng làm frontend)
* **http-echo** (dùng làm backend)

Tạo file `demo-app.yaml` với nội dung sau:

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

▶️ Áp dụng cấu hình:

```bash
kubectl apply -f demo-app.yaml
```

📸 Kết quả sau khi apply:

![kubectl apply demo-app](/images/2.2/2.2.3.png)

---

### Kiểm tra kết nối và chuẩn bị cho các lab tiếp theo

Chạy lệnh sau để kiểm tra trạng thái các Pod:

```bash
kubectl get pods -n demo
```

📸 Kết quả hiển thị:

![kubectl get pods -n demo](/images/2.2/2.2.4.png)

✅ Bạn sẽ thấy cả 2 Pod `nginx` và `backend` đang ở trạng thái **Running**.

> Ứng dụng mẫu này sẽ được sử dụng xuyên suốt trong các phần tiếp theo như: NetworkPolicy, Falco, OPA...

---

### Tuỳ chọn: Expose nginx ra ngoài

Để truy cập nginx từ bên ngoài (ví dụ từ trình duyệt hoặc curl), bạn có thể expose Pod qua LoadBalancer:

```bash
kubectl expose pod nginx --port=80 --type=LoadBalancer -n demo
```

Ghi chú:

* Câu lệnh này sẽ tạo một `Service` có địa chỉ IP public (nếu chạy trên cloud như EKS)
* Bạn có thể dùng lệnh `kubectl get svc -n demo` để lấy địa chỉ truy cập.

---

🎉 **Bạn đã hoàn tất phần triển khai workload mẫu – sẵn sàng cho các lab bảo mật tiếp theo!**
