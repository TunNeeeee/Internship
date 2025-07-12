---
title : "Tạo Cluster EKS trên AWS"
date : 2025-07-07
weight : 1
chapter : false
pre : "<b> 2.1 </b>"
---

**Nội dung chính:**

- [Bước 1: Cài đặt công cụ trên Windows](#bước-1-cài-đặt-công-cụ-trên-windows)
- [Bước 2: Tạo file cấu hình cluster](#bước-2-tạo-file-cấu-hình-cluster)
- [Bước 3: Tạo cluster bằng lệnh](#bước-3-tạo-cluster-bằng-lệnh)
- [Bước 4: Kiểm tra kết nối với kubectl](#bước-4-kiểm-tra-kết-nối-với-kubectl)
- [Ghi chú và lưu ý](#ghi-chú-và-lưu-ý)


---

## Bước 1: Cài đặt công cụ trên Windows

1. **AWS CLI** 

Truy cập: [https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-windows.html](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-windows.html) → Tải file `.msi`, cài đặt như phần mềm bình thường. Sau đó mở **CMD** và kiểm tra:
```powershell
aws –version
```
Nếu hiện version là cài thành công.

⚙️ Cấu hình tài khoản AWS:

- Tạo IAM USER
- Tạo Access keys/ Secret Access Key
- Mở powershell 
```powershell
aws configure
```
- Sau đó nhập các thông tin:
    - AWS Access Key ID
    - AWS Secret Access Key
    - Region: ap-southeast-1
    - Output format: json

2. Cài đặt **kubectl** 

Truy cập link chính thức:
👉 https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/
Hoặc dùng choco nếu đã có Chocolatey:
```powershell
choco install kubernetes-cli
```
Kiểm tra lại bằng: 
```powershell
kubectl version –client
```

3. Cài đặt **eksctl**
Truy cập:
👉 https://eksctl.io/introduction/#installation
Tải file .exe → Đổi tên thành eksctl.exe và:
- Copy vào thư mục như C:\Program Files\eksctl
- Thêm thư mục đó vào Environment Variables > PATH

Hoặc dùng choco nếu đã có Chocolatey:
```powershell
choco install eksctl
```
- Kiểm tra lại bằng: 
```powershell
eksctl version
```


## Bước 2: Tạo file cấu hình cluster

Mở Notepad hoặc VS Code, tạo file eks-cluster.yaml với nội dung:
```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: secure-eks
  region: ap-southeast-1

nodeGroups:
  - name: ng-1
    instanceType: t3.medium
    desiredCapacity: 2
    ssh:
      allow: true
```
Lưu lại cùng thư mục nơi bạn mở terminal sau này.

## Bước 3: Tạo cluster bằng lệnh

Mở PowerShell hoặc terminal trong VS Code, chuyển đến thư mục chứa file eks-cluster.yaml, sau đó chạy:
 ```bash
 eksctl create cluster -f eks-cluster.yaml
```
⏳ Quá trình tạo có thể mất 10–15 phút.
eksctl sẽ tạo: VPC, IAM role, Security Group, Control Plane, EC2 Node…

## Bước 4: Kiểm tra kết nối với kubectl

Sau khi cluster tạo xong, xác minh kết nối:

```bash
kubectl get nodes
```
✅ Bạn sẽ thấy 2 node worker đang chạy – EKS đã hoạt động sẵn sàng!

📸 Chụp ảnh terminal kết quả để sử dụng trong báo cáo workshop.

## Ghi chú

- eksctl sẽ tự tạo VPC và node group nếu chưa có
- Nên dùng region ap-southeast-1 (Singapore) nếu bạn ở Việt Nam để giảm độ trễ