---

title: "Quản lý chính sách với OPA Gatekeeper (Windows)"
date: 2025-07-15
weight: 1
chapter: false
pre: "<b> 4.1 </b>"
-------------------

## Nội dung chính

* [Giới thiệu OPA Gatekeeper](#giới-thiệu)
* [Cài đặt Helm trên Windows](#cài-đặt-helm-trên-windows)
* [Triển khai Gatekeeper trên EKS](#triển-khai-gatekeeper-trên-eks)
* [Viết và kiểm tra policy](#viết-và-kiểm-tra-policy)

---

## Giới thiệu

**OPA (Open Policy Agent)** là công cụ dùng để thực thi chính sách trên nhiều hệ thống. Gatekeeper là một dự án mở rộng giúp áp dụng OPA cho Kubernetes.

Gatekeeper giúp:

* Kiểm soát triển khai dựa trên quy tắc (constraint)
* Ghi log các hành vi vi phạm
* Ngăn chặn cấu hình không an toàn (ví dụ: cấm image có tag `:latest`)

---

## Cài đặt Helm trên Windows

1. Truy cập trang [https://github.com/helm/helm/releases](https://github.com/helm/helm/releases)
2. Tải về file: `helm-v3.x.x-windows-amd64.zip`
3. Giải nén → Lấy file `helm.exe`
4. Tạo thư mục ví dụ: `C:\Program Files\helm\` → copy `helm.exe` vào
5. Thêm đường dẫn vào `Environment Variables > Path`
6. Mở PowerShell/CMD và kiểm tra:

```bash
helm version
```

✅ Nếu hiện version là thành công.

---

## Triển khai Gatekeeper trên EKS

### Bước 1: Thêm Helm repo Gatekeeper

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update
```

### Bước 2: Cài đặt Gatekeeper

```bash
helm install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system \
  --create-namespace
```

### Bước 3: Kiểm tra pod

```bash
kubectl get pods -n gatekeeper-system
```

📸 Chụp ảnh trạng thái Running.

---

## Viết và kiểm tra policy

### Bước 1: Tạo file `template.yaml`

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredimagetag
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredImageTag
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredimagetag

        violation["image tag 'latest' is not allowed"] {
          container := input.review.object.spec.containers[_]
          endswith(container.image, ":latest")
        }
```

```bash
kubectl apply -f template.yaml
```

### Bước 2: Tạo file `constraint.yaml`

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredImageTag
metadata:
  name: disallow-latest-tag
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

```bash
kubectl apply -f constraint.yaml
```

### Bước 3: Tạo pod test bị vi phạm

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: nginx
      image: nginx:latest
```

```bash
kubectl apply -f test-pod.yaml
```

➡️ Kết quả: Pod bị từ chối do vi phạm chính sách.

---

✅ **Ghi chú**:

* Gatekeeper rất hiệu quả trong việc kiểm soát việc triển khai.
* Có thể mở rộng viết policy riêng theo yêu cầu.

---

Tiếp theo: [4.2 Giám sát Runtime với Falco →](/4-Policy-Enforcement-And-Runtime-Protection/2-Falco-Setup-And-Rule/)
