---
title: "Container Logging to CloudWatch with Fluent Bit"
date: 2025-07-15
weight: 1
chapter: false
pre: " 5.1 "
---

## Main Content

- [Fluent Bit + CloudWatch Overview](#fluent-bit--cloudwatch-overview)
- [Environment Preparation](#environment-preparation)
- [Deploying Fluent Bit to EKS](#deploying-fluent-bit-to-eks)
- [Configuring CloudWatch Output](#configuring-cloudwatch-output)
- [Checking Logs on CloudWatch](#checking-logs-on-cloudwatch)
- [Troubleshooting](#troubleshooting)
- [Notes & Extended Practice](#notes--extended-practice)

---

## Fluent Bit + CloudWatch Overview

**Fluent Bit** is a lightweight, high-performance log collector and forwarder. On EKS, it helps:

- Collect logs from all containers in the cluster
- Transform and enrich logs with Kubernetes metadata
- Send logs to CloudWatch Logs in standard JSON format
- Support filtering and routing logs by namespace, labels

**Advantages:**
- **Lightweight**: Only ~15MB memory footprint
- **High Performance**: Process thousands of events per second
- **Flexible**: Support multiple destinations (CloudWatch, Elasticsearch, Kafka, S3...)
- **Cloud-native**: Built-in integration with Kubernetes

---

## Environment Preparation

### Check EKS cluster
```bash
# Check cluster status
kubectl cluster-info

# Check nodes
kubectl get nodes

# Create logging namespace
kubectl create namespace logging
```

### Create test application (optional)
```bash
# Create log generator for testing
kubectl create deployment log-generator --image=busybox --namespace=default
kubectl patch deployment log-generator -p '{"spec":{"template":{"spec":{"containers":[{"name":"busybox","command":["sh","-c","while true; do echo \"[$(date)] Test log message #$((i++)) - This is a test log for CloudWatch\"; sleep 10; done"],"image":"busybox"}]}}}}'
```

---

## Deploying Fluent Bit to EKS

### Step 1: Create IAM role for Fluent Bit

**Create IAM Policy** with CloudWatch Logs write permissions:

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

**Create IAM Role and attach policy:**

```bash
# Create trust policy for IRSA
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

# Create IAM Role
aws iam create-role \
    --role-name FluentBitRole \
    --assume-role-policy-document file://trust-policy.json

# Attach policy
aws iam attach-role-policy \
    --role-name FluentBitRole \
    --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/FluentBitCloudWatchPolicy
```

### Step 2: Install Fluent Bit using Helm

```bash
# Add Fluent Bit Helm repository
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

# Install Fluent Bit with CloudWatch enabled
helm install fluent-bit fluent/fluent-bit \
  --namespace logging \
  --create-namespace \
  --set serviceAccount.create=true \
  --set serviceAccount.name=fluent-bit-sa \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws:iam::YOUR_ACCOUNT_ID:role/FluentBitRole"
```

### Step 3: Check deployment status

```bash
# Check pod status
kubectl get pods -n logging

# View Fluent Bit logs
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit --tail=100

# Check service account
kubectl describe serviceaccount fluent-bit-sa -n logging
```

---

## Configuring CloudWatch Output

### Create ConfigMap for Fluent Bit

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

# Restart Fluent Bit to load new config
kubectl rollout restart daemonset/fluent-bit -n logging
```

---

## Checking Logs on CloudWatch

### 1. Access CloudWatch Console

1. Open [CloudWatch Console](https://console.aws.amazon.com/cloudwatch/)
2. Select **Log groups** from the left menu
3. Find log group: `/aws/containerinsights/your-cluster-name/application`

### 2. Check log streams

```bash
# Create some test logs
kubectl run test-pod --image=nginx --restart=Never
kubectl logs test-pod

# Check logs from demo2 namespace
kubectl logs -n demo2 -l app=nginx --tail=10
kubectl logs -n demo2 -l app=backend --tail=10
```

### 3. View logs on CloudWatch

**Example log format on CloudWatch:**

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

### 4. Filtering logs by namespace

**View only demo2 namespace logs:**

```bash
# Add filter in Fluent Bit config
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

### Common issues and solutions

**1. Fluent Bit fails to start:**
```bash
# Check error logs
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit

# Check permissions
kubectl describe pod -n logging -l app.kubernetes.io/name=fluent-bit
```

**2. No logs visible on CloudWatch:**
```bash
# Check AWS credentials
kubectl exec -n logging deployment/fluent-bit -- env | grep AWS

# Check network connectivity
kubectl exec -n logging deployment/fluent-bit -- nslookup logs.ap-southeast-1.amazonaws.com

# View detailed Fluent Bit logs
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit | grep -i "cloudwatch\|error"
```

**3. Incorrect log format:**
```bash
# Check parser configuration
kubectl get configmap fluent-bit-config -n logging -o yaml

# Test parser locally
kubectl exec -n logging deployment/fluent-bit -- fluent-bit --dry-run --config /fluent-bit/etc/fluent-bit.conf
```

**4. Permission errors:**
```bash
# Check IAM role
aws sts get-caller-identity

# Check service account annotation
kubectl describe serviceaccount fluent-bit-sa -n logging | grep Annotations
```

### Useful debug commands

```bash
# View realtime logs
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit -f

# Check metrics
kubectl port-forward -n logging svc/fluent-bit 2020:2020
curl localhost:2020/api/v1/metrics

# View current configuration
kubectl exec -n logging deployment/fluent-bit -- cat /fluent-bit/etc/fluent-bit.conf
```

---

## Notes & Extended Practice

### Performance optimization

```yaml
# Increase buffer size for high-volume logs
[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Mem_Buf_Limit     100MB
    Buffer_Max_Size   1MB
    Buffer_Chunk_Size 64KB
```

### Log routing by environment

```yaml
# Route logs by namespace
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
# Reduce retention to save costs
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

For larger systems, consider:

```bash
# Install OpenTelemetry Operator
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml

# Configure OpenTelemetry Collector
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

**Important Notes:**
- Monitor CloudWatch Logs costs (charged per GB ingested)
- Consider appropriate log retention policy
- Use log aggregation and sampling for production workloads
- Combine with CloudWatch Insights to query and analyze logs

---

👉 **Next:** [5.2 Security Dashboard →](../2-Security-Dashboard/) to set up monitoring and alerting for cluster security.