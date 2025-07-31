---

title: "Kiểm tra tuân thủ CIS Benchmark với kube-bench"
date: 2025-07-13
weight: 2
chapter: false
pre: "<b> 3.2 </b>"
-------------------

## Nội dung chính

* [Giới thiệu về CIS Benchmark và kube-bench](#giới-thiệu)
* [Triển khai kube-bench trên EKS](#triển-khai)
* [Phân tích kết quả kiểm tra và khuyến nghị](#phân-tích-kết-quả)

---

## Giới thiệu

**CIS Kubernetes Benchmark** là bộ quy tắc bảo mật chuẩn hóa do Center for Internet Security ban hành. Nó bao gồm các mục kiểm tra từ API Server, Scheduler, Etcd, Kubelet...

**kube-bench** là công cụ của Aqua Security để tự động hóa việc kiểm tra các điểm đó trên cluster Kubernetes.

---

## Triển khai kube-bench trên EKS

### Bước 1: Tạo file job yaml

Tạo file `kube-bench-job.yaml` như sau:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
spec:
  template:
    spec:
      containers:
      - name: kube-bench
        image: aquasec/kube-bench:latest
        command: ["kube-bench", "--benchmark", "eks"]
        volumeMounts:
        - name: var-lib-kubelet
          mountPath: /var/lib/kubelet
          readOnly: true
        - name: etc-systemd
          mountPath: /etc/systemd
          readOnly: true
        - name: var-lib-etcd
          mountPath: /var/lib/etcd
          readOnly: true
      restartPolicy: Never
      volumes:
      - name: var-lib-kubelet
        hostPath:
          path: /var/lib/kubelet
      - name: etc-systemd
        hostPath:
          path: /etc/systemd
      - name: var-lib-etcd
        hostPath:
          path: /var/lib/etcd
  backoffLimit: 0

```

### Bước 2: Áp dụng Job lên cluster

```bash
kubectl apply -f kube-bench-job.yaml
```
![CIS-Benchmark-With-KubeBench](/images/3.2/3.2.1.png)
⏳ Chờ vài phút để Job chạy xong.

### Bước 3: Xem kết quả kiểm tra

```bash
kubectl logs job/kube-bench
```
![CIS-Benchmark-With-KubeBench](/images/3.2/3.2.2.png)
![CIS-Benchmark-With-KubeBench](/images/3.2/3.2.3.png)
---

## Phân tích kết quả

Output sẽ hiển thị các mục kiểm tra theo chuẩn CIS, ví dụ:

```
== Summary total ==
4 checks PASS
4 checks FAIL
43 checks WARN
```

❌ [FAIL]
- --anonymous-auth=false: tắt quyền truy cập ẩn danh
- --authorization-mode=Webhook: cấu hình authorization an toàn
- --make-iptables-util-chains=true: xử lý các chuỗi iptables đúng cách

🔧 Cách khắc phục: Sửa file kubelet.service trên từng node theo hướng dẫn systemctl daemon-reload + restart kubelet.

⚠️ [WARN]
- Sử dụng quá nhiều wildcard trong RBAC (roles, clusterroles)
- Không có NetworkPolicy cho các namespace
- PSP chưa cấu hình kỹ → cho phép container đặc quyền (privileged)
- Chưa cấu hình chuẩn IAM / ECR / Secrets trên AWS

✅ [PASS]
- Một số tệp cấu hình kubelet đặt quyền sở hữu và permission đúng (644, root:root)
---

## Ghi chú bổ sung

* kube-bench có thể chạy ở mode `host`, `daemonset`, hoặc `job`, tùy mục đích.
* Chạy riêng node có label bằng cách dùng nodeSelector
* Có thể export kết quả sang JSON:

```bash
kubectl logs job/kube-bench -o json > cis-result.json
```

✅ Best Practice tổng quát:
- Luôn kiểm tra định kỳ theo CIS Benchmark
- Không dùng config mặc định của Kubernetes / Kubelet
- Cố gắng giảm mức độ cảnh báo WARN → FAIL → PASS theo thứ tự ưu tiên
- Tự động hóa kiểm tra bằng cách tích hợp kube-bench vào CI/CD hoặc cron job
