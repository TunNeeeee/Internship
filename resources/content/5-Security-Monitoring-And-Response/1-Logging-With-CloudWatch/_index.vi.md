---
title: "Ghi log container vào CloudWatch với Fluent Bit"
date: 2025-07-15
weight: 1
chapter: false
pre: " 5.1 "
---

## Nội dung chính

- [Tổng quan Fluent Bit + CloudWatch](#tổng-quan-fluent-bit--cloudwatch)
- [Chuẩn bị môi trường](#chuẩn-bị-môi-trường)
- [Triển khai Fluent Bit lên EKS](#triển-khai-fluent-bit-lên-eks)
- [Cấu hình CloudWatch Output](#cấu-hình-cloudwatch-output)
- [Kiểm tra log trên CloudWatch](#kiểm-tra-log-trên-cloudwatch)
- [Troubleshooting](#troubleshooting)
- [Ghi chú & thực hành mở rộng](#ghi-chú--thực-hành-mở-rộng)

---

## Tổng quan Fluent Bit + CloudWatch

**Fluent Bit** là một trình thu thập và chuyển tiếp log nhẹ, hiệu quả cao. Trên EKS, nó giúp:

- Thu thập log từ tất cả container trong cluster
- Chuyển đổi và enrichment log với metadata Kubernetes
- Gửi log đến CloudWatch Logs với format JSON chuẩn
- Hỗ trợ filtering và routing log theo namespace, labels

**Ưu điểm:**
- **Gọn nhẹ**: Chỉ ~15MB memory footprint
- **Hiệu suất cao**: Xử lý hàng nghìn events/giây
- **Linh hoạt**: Hỗ trợ nhiều đích đến (CloudWatch, Elasticsearch, Kafka, S3...)
- **Cloud-native**: Tích hợp sẵn với Kubernetes

---

## Chuẩn bị môi trường

### Kiểm tra EKS cluster
```bash
# Kiểm tra cluster status
kubectl cluster-info

# Kiểm tra nodes
kubectl get nodes

# Tạo namespace logging
kubectl create namespace logging
```

### Tạo test application (optional)
```bash
# Tạo log generator để test
kubectl create deployment log-generator --image=busybox --namespace=default
kubectl patch deployment log-generator -p '{"spec":{"template":{"spec":{"containers":[{"name":"busybox","command":["sh","-c","while true; do echo \"[$(date)] Test log message #$((i++)) - This is a test log for CloudWatch\"; sleep 10; done"],"image":"busybox"}]}}}}'
```

---

## Triển khai Fluent Bit lên EKS

### Bước 1: Tạo IAM role cho Fluent Bit

**Tạo IAM Policy** với quyền ghi CloudWatch Logs:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:PutLogEvents",
        "logs:DescribeLogStreams",
        "logs:DescribeLogGroups",
        "logs:CreateLogStream",
        "logs:CreateLogGroup"
      ],
      "Resource": "*"
    }
  ]
}
```

**Tạo IAM Role và gán policy:**

```bash
# Tạo trust policy cho IRSA
cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/oidc.eks.ap-southeast-1.amazonaws.com/id/YOUR_OIDC_ID"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-southeast-1.amazonaws.com/id/YOUR_OIDC_ID:sub": "system:serviceaccount:logging:fluent-bit-sa"
        }
      }
    }
  ]
}
EOF

# Tạo IAM Role
aws iam create-role \
    --role-name FluentBitRole \
    --assume-role-policy-document file://trust-policy.json

# Gán policy
aws iam attach-role-policy \
    --role-name FluentBitRole \
    --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/FluentBitCloudWatchPolicy
```

### Bước 2: Cài Fluent Bit bằng Helm

```bash
# Thêm Fluent Bit Helm repository
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

# Cài đặt Fluent Bit với CloudWatch enabled
helm install fluent-bit fluent/fluent-bit \
  --namespace logging \
  --create-namespace \
  --set serviceAccount.create=true \
  --set serviceAccount.name=fluent-bit-sa \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws:iam::YOUR_ACCOUNT_ID:role/FluentBitRole"
```

### Bước 3: Kiểm tra trạng thái deployment

```bash
# Kiểm tra pod status
kubectl get pods -n logging

# Xem logs của Fluent Bit
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit --tail=100

# Kiểm tra service account
kubectl describe serviceaccount fluent-bit-sa -n logging
```

---

## Cấu hình CloudWatch Output

### Tạo ConfigMap cho Fluent Bit

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            docker
        Tag               kube.*
        Refresh_Interval  5
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Keep_Log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On

    [OUTPUT]
        Name                cloudwatch_logs
        Match               *
        region              ap-southeast-1
        log_group_name      /aws/containerinsights/your-cluster-name/application
        log_stream_prefix   fluent-bit-
        auto_create_group   On
        log_retention_days  7

  parsers.conf: |
    [PARSER]
        Name   docker
        Format json
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
        Time_Keep On
```

### Apply configuration

```bash
kubectl apply -f fluent-bit-config.yaml

# Restart Fluent Bit để load config mới
kubectl rollout restart daemonset/fluent-bit -n logging
```

---

## Kiểm tra log trên CloudWatch

### 1. Truy cập CloudWatch Console

1. Mở [CloudWatch Console](https://console.aws.amazon.com/cloudwatch/)
2. Chọn **Log groups** từ menu bên trái
3. Tìm log group: `/aws/containerinsights/your-cluster-name/application`

### 2. Kiểm tra log streams

```bash
# Tạo một số test logs
kubectl run test-pod --image=nginx --restart=Never
kubectl logs test-pod

# Kiểm tra logs của demo2 namespace
kubectl logs -n demo2 -l app=nginx --tail=10
kubectl logs -n demo2 -l app=backend --tail=10
```

### 3. Xem logs trên CloudWatch

**Ví dụ log format trên CloudWatch:**

```json
{
  "time": "2025-07-16T05:39:39.654Z",
  "stream": "stdout",
  "log": "[Wed Jul 16 05:39:39 UTC 2025] Test log message #1251 - This is a test log for CloudWatch",
  "kubernetes": {
    "pod_name": "nginx-5869d7778c-db2lm",
    "namespace_name": "demo2",
    "container_name": "nginx",
    "labels": {
      "app": "nginx",
      "version": "v1.0"
    },
    "host": "ip-192-168-39-33.ap-southeast-1.compute.internal"
  }
}
```

### 4. Filtering logs theo namespace

**Chỉ xem logs của demo2 namespace:**

```bash
# Thêm filter trong Fluent Bit config
[FILTER]
    Name    grep
    Match   kube.*
    Regex   kubernetes.namespace_name demo2

[OUTPUT]
    Name                cloudwatch_logs
    Match               *
    region              ap-southeast-1
    log_group_name      /aws/containerinsights/demo2-logs
    log_stream_prefix   demo2-
    auto_create_group   On
```

---

## Troubleshooting

### Vấn đề thường gặp và cách khắc phục

**1. Fluent Bit không start được:**
```bash
# Kiểm tra logs lỗi
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit

# Kiểm tra permissions
kubectl describe pod -n logging -l app.kubernetes.io/name=fluent-bit
```

**2. Không thấy logs trên CloudWatch:**
```bash
# Kiểm tra AWS credentials
kubectl exec -n logging deployment/fluent-bit -- env | grep AWS

# Kiểm tra network connectivity
kubectl exec -n logging deployment/fluent-bit -- nslookup logs.ap-southeast-1.amazonaws.com

# Xem chi tiết logs của Fluent Bit
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit | grep -i "cloudwatch\|error"
```

**3. Log format không đúng:**
```bash
# Kiểm tra parser configuration
kubectl get configmap fluent-bit-config -n logging -o yaml

# Test parser locally
kubectl exec -n logging deployment/fluent-bit -- fluent-bit --dry-run --config /fluent-bit/etc/fluent-bit.conf
```

**4. Lỗi permissions:**
```bash
# Kiểm tra IAM role
aws sts get-caller-identity

# Kiểm tra service account annotation
kubectl describe serviceaccount fluent-bit-sa -n logging | grep Annotations
```

### Debug commands hữu ích

```bash
# Xem realtime logs
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit -f

# Kiểm tra metrics
kubectl port-forward -n logging svc/fluent-bit 2020:2020
curl localhost:2020/api/v1/metrics

# Xem cấu hình hiện tại
kubectl exec -n logging deployment/fluent-bit -- cat /fluent-bit/etc/fluent-bit.conf
```

---

## Ghi chú & thực hành mở rộng

### Tối ưu hóa performance

```yaml
# Tăng buffer size cho high-volume logs
[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Mem_Buf_Limit     100MB
    Buffer_Max_Size   1MB
    Buffer_Chunk_Size 64KB
```

### Log routing theo environment

```yaml
# Route logs theo namespace
[OUTPUT]
    Name                cloudwatch_logs
    Match               kube.var.log.containers.*production*
    log_group_name      /aws/containerinsights/production-logs

[OUTPUT]
    Name                cloudwatch_logs
    Match               kube.var.log.containers.*staging*
    log_group_name      /aws/containerinsights/staging-logs
```

### Structured logging

```yaml
# Parse JSON logs
[FILTER]
    Name    parser
    Match   kube.*
    Parser  json
    Key_Name log
```

### Cost optimization

```yaml
# Giảm retention để tiết kiệm chi phí
[OUTPUT]
    Name                cloudwatch_logs
    Match               *
    log_retention_days  3
    
# Exclude debug logs
[FILTER]
    Name    grep
    Match   *
    Exclude log (DEBUG|TRACE)
```

### Monitoring Fluent Bit

```yaml
# Enable metrics endpoint
[SERVICE]
    HTTP_Server   On
    HTTP_Listen   0.0.0.0
    HTTP_Port     2020

# Prometheus metrics
[OUTPUT]
    Name  prometheus_exporter
    Match *
    Host  0.0.0.0
    Port  2021
```

### Alternative: OpenTelemetry Collector

Đối với hệ thống lớn hơn, có thể xem xét:

```bash
# Cài OpenTelemetry Operator
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml

# Cấu hình OpenTelemetry Collector
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
spec:
  config: |
    receivers:
      filelog:
        include: ["/var/log/containers/*.log"]
    
    exporters:
      awscloudwatchlogs:
        region: ap-southeast-1
        log_group_name: "/aws/containerinsights/otel-logs"
    
    service:
      pipelines:
        logs:
          receivers: [filelog]
          exporters: [awscloudwatchlogs]
```

**Lưu ý quan trọng:**
- Theo dõi chi phí CloudWatch Logs (tính theo GB ingested)
- Cân nhắc log retention policy phù hợp
- Sử dụng log aggregation và sampling cho production workloads
- Kết hợp với CloudWatch Insights để query và analyze logs

---

👉 **Tiếp theo:** [5.2 Security Dashboard →](../2-Security-Dashboard/) để thiết lập monitoring và alerting cho cluster security.