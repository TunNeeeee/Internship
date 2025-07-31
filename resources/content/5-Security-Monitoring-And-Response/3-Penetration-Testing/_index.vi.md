---
title: " Kiểm tra thâm nhập"
date: 2025-07-15
weight: 3
chapter: false
pre: "5.3"
---

## Nội dung chính

* [Giới thiệu về Penetration Testing](#giới-thiệu)
* [Chuẩn bị môi trường kiểm tra](#chuẩn-bị-môi-trường-kiểm-tra)
* [Kiểm tra bảo mật Pod và Container](#kiểm-tra-bảo-mật-pod-và-container)
* [Kiểm tra cấu hình RBAC](#kiểm-tra-cấu-hình-rbac)
* [Kiểm tra Network Policy](#kiểm-tra-network-policy)
* [Sử dụng Kube-Hunter](#sử-dụng-kube-hunter)
* [Báo cáo và khắc phục](#báo-cáo-và-khắc-phục)

---

## Giới thiệu

**Penetration Testing** (kiểm tra thâm nhập) là quá trình đánh giá bảo mật hệ thống bằng cách mô phỏng các cuộc tấn công thực tế. Đối với Kubernetes, chúng ta cần kiểm tra:

* Cấu hình bảo mật của cluster
* Quyền truy cập và phân quyền
* Lỗ hổng trong container image
* Cấu hình network và policy
* Compliance với các chuẩn bảo mật

---

## Chuẩn bị môi trường kiểm tra

### Bước 1: Tạo namespace riêng cho testing

```bash
kubectl create namespace pentest
kubectl config set-context --current --namespace=pentest
```

### Bước 2: Tạo vulnerable pod để test

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vulnerable-pod
  namespace: pentest
spec:
  securityContext:
    runAsUser: 0
    runAsGroup: 0
  containers:
  - name: test-container
    image: nginx:latest
    securityContext:
      privileged: true
      allowPrivilegeEscalation: true
      capabilities:
        add:
        - SYS_ADMIN
        - NET_ADMIN
    volumeMounts:
    - name: host-root
      mountPath: /host
  volumes:
  - name: host-root
    hostPath:
      path: /
```

```bash
kubectl apply -f vulnerable-pod.yaml
```

---

## Kiểm tra bảo mật Pod và Container

### Bước 1: Kiểm tra privilege escalation

```bash
# Truy cập vào pod
kubectl exec -it vulnerable-pod -- /bin/bash

# Kiểm tra quyền root
whoami
id

# Kiểm tra khả năng truy cập host filesystem
ls /host
cat /host/etc/passwd
```

### Bước 2: Kiểm tra container breakout

```bash
# Trong pod, kiểm tra mount points
mount | grep -E "(proc|sys|dev)"

# Kiểm tra capabilities
capsh --print

# Thử escape container
chroot /host /bin/bash
```

### Bước 3: Tạo pod với security context tốt hơn

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: pentest
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: secure-container
    image: nginx:1.21-alpine
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        memory: "128Mi"
        cpu: "100m"
      requests:
        memory: "64Mi"
        cpu: "50m"
```

```bash
kubectl apply -f secure-pod.yaml
```

---

## Kiểm tra cấu hình RBAC

### Bước 1: Tạo ServiceAccount với quyền cao

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: test-sa
  namespace: pentest
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: test-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: test-sa
  namespace: pentest
```

```bash
kubectl apply -f test-rbac.yaml
```

### Bước 2: Kiểm tra quyền truy cập

```bash
# Tạo pod với ServiceAccount
kubectl run test-rbac --image=bitnami/kubectl:latest \
  --serviceaccount=test-sa \
  --namespace=pentest \
  --rm -it --restart=Never \
  -- /bin/bash

# Trong pod, kiểm tra quyền
kubectl auth can-i --list
kubectl get secrets --all-namespaces
kubectl get nodes
```

### Bước 3: Kiểm tra token ServiceAccount

```bash
# Lấy token
kubectl get secret $(kubectl get serviceaccount test-sa -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.token}' | base64 -d

# Sử dụng token để truy cập API
curl -k -H "Authorization: Bearer <TOKEN>" https://<API_SERVER>/api/v1/namespaces
```

---

## Kiểm tra Network Policy

### Bước 1: Tạo pod test network

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: network-test-1
  namespace: pentest
  labels:
    app: test-app
spec:
  containers:
  - name: test
    image: busybox
    command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: network-test-2
  namespace: pentest
  labels:
    app: other-app
spec:
  containers:
  - name: test
    image: busybox
    command: ["sleep", "3600"]
```

```bash
kubectl apply -f network-test-pods.yaml
```

### Bước 2: Kiểm tra kết nối trước khi có policy

```bash
# Lấy IP của pod
kubectl get pods -o wide

# Test kết nối
kubectl exec network-test-1 -- ping -c 3 <IP_OF_POD_2>
kubectl exec network-test-1 -- wget -qO- http://<IP_OF_POD_2>:80
```

### Bước 3: Áp dụng Network Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: pentest
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

```bash
kubectl apply -f network-policy.yaml
```

### Bước 4: Kiểm tra lại kết nối

```bash
# Test kết nối sau khi có policy
kubectl exec network-test-1 -- ping -c 3 <IP_OF_POD_2>
# Kết quả: Timeout (bị chặn)
```

---

## Sử dụng Kube-Hunter

### Bước 1: Cài đặt Kube-Hunter

```bash
# Chạy Kube-Hunter trong container
kubectl run kube-hunter --image=aquasec/kube-hunter:latest \
  --namespace=pentest \
  --rm -it --restart=Never \
  -- --remote <CLUSTER_IP>
```

### Bước 2: Chạy Kube-Hunter từ bên ngoài

```bash
# Trên máy local
pip install kube-hunter

# Scan cluster
kube-hunter --remote <CLUSTER_ENDPOINT>
```

### Bước 3: Phân tích kết quả

Kube-Hunter sẽ báo cáo:
* Exposed services
* Insecure configurations
* Potential attack vectors
* Privilege escalation opportunities

---

## Báo cáo và khắc phục

### Tổng hợp lỗ hổng phát hiện

1. **High Risk**:
   - Privileged containers
   - Host filesystem access
   - Cluster-admin permissions

2. **Medium Risk**:
   - Latest image tags
   - Missing resource limits
   - Weak network policies

3. **Low Risk**:
   - Missing security contexts
   - Outdated images

### Khuyến nghị khắc phục

```yaml
# Pod Security Standards
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

```bash
kubectl apply -f secure-namespace.yaml
```

### Cleanup

```bash
# Xóa các tài nguyên test
kubectl delete namespace pentest
kubectl delete clusterrolebinding test-binding
```

---

## Tổng kết Workshop

🎉 **Chúc mừng!** Bạn đã hoàn thành workshop **"Container Security trên AWS EKS"**

**Những gì đã học:**
* Thiết lập EKS cluster an toàn
* Scanning image với Trivy
* Hardening với CIS Benchmark
* Quản lý policy với OPA Gatekeeper
* Monitoring với Falco
* Logging với CloudWatch
* Penetration testing

**Bước tiếp theo:**
* Áp dụng vào môi trường production
* Thiết lập CI/CD pipeline với security scanning
* Monitoring và alerting tự động
* Compliance với các chuẩn bảo mật

---

✅ **Ghi chú quan trọng:**

* Penetration testing chỉ nên thực hiện trên môi trường test
* Luôn có backup trước khi test
* Tuân thủ các quy định pháp lý về security testing
* Báo cáo và khắc phục ngay các lỗ hổng phát hiện

---

**Cảm ơn bạn đã tham gia workshop!** 🚀