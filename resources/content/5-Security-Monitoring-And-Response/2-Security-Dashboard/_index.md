---
title: "Security Dashboard - EKS Security Monitoring"
date: 2025-07-15
weight: 2
chapter: false
pre: " 5.2 "
---

## Main Content

- [Security Dashboard Overview](#security-dashboard-overview)
- [Setting up Amazon GuardDuty](#setting-up-amazon-guardduty)
- [Configuring AWS Config Rules](#configuring-aws-config-rules)
- [Integrating CloudWatch Security Dashboard](#integrating-cloudwatch-security-dashboard)
- [Alerting and Automation](#alerting-and-automation)

---

## Security Dashboard Overview

Security Dashboard consolidates security information from multiple sources to provide a comprehensive view of the EKS cluster's security posture.

### Architecture Overview:
```
EKS Cluster ──→ GuardDuty ──→ Security Hub ──→ CloudWatch Dashboard
     │              │              │              │
     ├─→ Config ─────┤              │              │
     └─→ Falco ──────┴──────────────┘              │
                                                   │
SNS Alerts ←───────────────────────────────────────┘
```

---

## Setting up Amazon GuardDuty

### Enable GuardDuty for EKS

```bash
# Enable GuardDuty
aws guardduty create-detector --enable

# Get detector ID
DETECTOR_ID=$(aws guardduty list-detectors --query 'DetectorIds[0]' --output text)

# Enable EKS Protection
aws guardduty update-detector \
    --detector-id $DETECTOR_ID \
    --kubernetes-audit-logs-configuration Enable=true
```

### Configure GuardDuty findings processor

```yaml
# guardduty-processor.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: guardduty-processor
  namespace: security-monitoring
data:
  process-findings.py: |
    import json
    import boto3
    
    def lambda_handler(event, context):
        detail = event['detail']
        finding_type = detail['type']
        severity = detail['severity']
        
        # Process EKS-related findings
        if 'Kubernetes' in finding_type:
            send_to_cloudwatch(detail)
            
        # Send alert for critical findings
        if severity >= 7.0:
            send_critical_alert(detail)
            
        return {'statusCode': 200}
    
    def send_to_cloudwatch(finding):
        cloudwatch = boto3.client('cloudwatch')
        cloudwatch.put_metric_data(
            Namespace='EKS/Security',
            MetricData=[{
                'MetricName': 'SecurityFinding',
                'Value': finding['severity'],
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'FindingType', 'Value': finding['type']}
                ]
            }]
        )
    
    def send_critical_alert(finding):
        sns = boto3.client('sns')
        sns.publish(
            TopicArn='arn:aws:sns:ap-southeast-1:ACCOUNT:eks-security-alerts',
            Message=f"Critical EKS Security Alert: {finding['type']}",
            Subject='Critical EKS Security Alert'
        )
```

---

## Configuring AWS Config Rules

### Deploy EKS Config Rules

```bash
# Enable Config
aws configservice put-configuration-recorder \
    --configuration-recorder name=eks-config-recorder,roleARN=arn:aws:iam::ACCOUNT:role/aws-service-role/config.amazonaws.com/AWSServiceRoleForConfig

# Create EKS-specific rules
aws configservice put-config-rule \
    --config-rule '{
        "ConfigRuleName": "eks-cluster-logging-enabled",
        "Source": {
            "Owner": "AWS",
            "SourceIdentifier": "EKS_CLUSTER_LOGGING_ENABLED"
        },
        "InputParameters": "{\"desiredLogTypes\":\"api,audit,authenticator\"}"
    }'

aws configservice put-config-rule \
    --config-rule '{
        "ConfigRuleName": "eks-secrets-encrypted",
        "Source": {
            "Owner": "AWS", 
            "SourceIdentifier": "EKS_SECRETS_ENCRYPTED"
        }
    }'
```

### Custom Config Rule for Pod Security

```python
# pod-security-rule.py
import json
import boto3
from kubernetes import client, config

def lambda_handler(event, context):
    # Load kubeconfig
    config.load_incluster_config()
    v1 = client.CoreV1Api()
    
    # Check pod security standards
    non_compliant_pods = []
    pods = v1.list_pod_for_all_namespaces()
    
    for pod in pods.items:
        if pod.metadata.namespace in ['kube-system', 'kube-public']:
            continue
            
        violations = check_pod_security(pod)
        if violations:
            non_compliant_pods.append({
                'name': pod.metadata.name,
                'namespace': pod.metadata.namespace,
                'violations': violations
            })
    
    # Return compliance result
    return {
        'complianceType': 'NON_COMPLIANT' if non_compliant_pods else 'COMPLIANT',
        'annotation': f"Found {len(non_compliant_pods)} non-compliant pods"
    }

def check_pod_security(pod):
    violations = []
    
    # Check if pod runs as root
    if pod.spec.security_context and pod.spec.security_context.run_as_user == 0:
        violations.append("Runs as root user")
    
    # Check privileged containers
    for container in pod.spec.containers:
        if container.security_context and container.security_context.privileged:
            violations.append(f"Container {container.name} is privileged")
    
    return violations
```

---

## Integrating CloudWatch Security Dashboard

### Create Security Dashboard

```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["EKS/Security", "SecurityFinding", "FindingType", "Kubernetes.Cluster"],
          ["EKS/Security/Falco", "SecurityAlert", "Priority", "CRITICAL"],
          ["AWS/Config", "ComplianceByConfigRule", "ConfigRuleName", "eks-cluster-logging-enabled"]
        ],
        "period": 300,
        "stat": "Sum",
        "region": "ap-southeast-1",
        "title": "Security Overview",
        "view": "timeSeries"
      }
    },
    {
      "type": "log",
      "properties": {
        "query": "SOURCE '/aws/eks/security/falco'\n| fields @timestamp, rule, priority, output\n| filter priority = \"CRITICAL\"\n| sort @timestamp desc\n| limit 10",
        "region": "ap-southeast-1",
        "title": "Recent Critical Alerts",
        "view": "table"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["EKS/Security/Pods", "PrivilegedPods", "Namespace", "demo2"],
          [".", "RootContainers", ".", "."]
        ],
        "period": 300,
        "stat": "Average",
        "region": "ap-southeast-1",
        "title": "Pod Security Metrics"
      }
    }
  ]
}
```

### Deploy Dashboard

```bash
# Create dashboard
aws cloudwatch put-dashboard \
    --dashboard-name "EKS-Security-Dashboard" \
    --dashboard-body file://eks-security-dashboard.json

# Create custom metrics for pod security
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: security-metrics-collector
  namespace: security-monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: security-metrics-collector
rules:
- apiGroups: [""]
  resources: ["pods", "namespaces"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: security-metrics-collector
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: security-metrics-collector
subjects:
- kind: ServiceAccount
  name: security-metrics-collector
  namespace: security-monitoring
EOF
```

---

## Alerting and Automation

### Create CloudWatch Alarms

```bash
# Alarm for Critical Security Findings
aws cloudwatch put-metric-alarm \
    --alarm-name "EKS-Critical-Security-Findings" \
    --alarm-description "Critical security findings detected" \
    --metric-name SecurityFinding \
    --namespace EKS/Security \
    --statistic Sum \
    --period 300 \
    --threshold 1 \
    --comparison-operator GreaterThanOrEqualToThreshold \
    --evaluation-periods 1 \
    --alarm-actions arn:aws:sns:ap-southeast-1:ACCOUNT:eks-security-alerts

# Alarm for Falco Critical Alerts
aws cloudwatch put-metric-alarm \
    --alarm-name "EKS-Falco-Critical-Alerts" \
    --alarm-description "Falco critical alerts detected" \
    --metric-name SecurityAlert \
    --namespace EKS/Security/Falco \
    --statistic Sum \
    --period 300 \
    --threshold 1 \
    --comparison-operator GreaterThanOrEqualToThreshold \
    --evaluation-periods 1 \
    --alarm-actions arn:aws:sns:ap-southeast-1:ACCOUNT:eks-security-alerts

# Alarm for Pod Security Violations
aws cloudwatch put-metric-alarm \
    --alarm-name "EKS-Pod-Security-Violations" \
    --alarm-description "Pod security violations detected" \
    --metric-name PrivilegedPods \
    --namespace EKS/Security/Pods \
    --statistic Average \
    --period 600 \
    --threshold 0 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions arn:aws:sns:ap-southeast-1:ACCOUNT:eks-security-alerts
```

### Automated Response

```python
# security-response-automation.py
import json
import boto3
from kubernetes import client, config

def lambda_handler(event, context):
    """Automated response to security alerts"""
    
    # Parse CloudWatch alarm
    message = json.loads(event['Records'][0]['Sns']['Message'])
    alarm_name = message['AlarmName']
    
    # Load kubeconfig
    config.load_incluster_config()
    v1 = client.CoreV1Api()
    
    # Automated responses based on alarm type
    if "Pod-Security-Violations" in alarm_name:
        quarantine_violating_pods(v1)
    elif "Critical-Security-Findings" in alarm_name:
        isolate_suspicious_workloads(v1)
    elif "Falco-Critical-Alerts" in alarm_name:
        scale_down_suspicious_deployments(v1)
    
    return {'statusCode': 200}

def quarantine_violating_pods(v1):
    """Quarantine pods with security violations"""
    pods = v1.list_pod_for_all_namespaces()
    
    for pod in pods.items:
        if has_security_violation(pod):
            # Add quarantine label
            v1.patch_namespaced_pod(
                name=pod.metadata.name,
                namespace=pod.metadata.namespace,
                body={
                    'metadata': {
                        'labels': {
                            'security.policy/quarantine': 'true'
                        }
                    }
                }
            )

def isolate_suspicious_workloads(v1):
    """Isolate workloads with security findings"""
    # Implementation for workload isolation
    pass

def scale_down_suspicious_deployments(v1):
    """Scale down deployments with critical alerts"""
    # Implementation for scaling down
    pass
```

### Create SNS Topic and Subscription

```bash
# Create SNS topic
aws sns create-topic --name eks-security-alerts

# Subscribe email
aws sns subscribe \
    --topic-arn arn:aws:sns:ap-southeast-1:ACCOUNT:eks-security-alerts \
    --protocol email \
    --notification-endpoint security@company.com

# Subscribe Lambda function
aws sns subscribe \
    --topic-arn arn:aws:sns:ap-southeast-1:ACCOUNT:eks-security-alerts \
    --protocol lambda \
    --notification-endpoint arn:aws:lambda:ap-southeast-1:ACCOUNT:function:security-response-automation
```

---

## Summary

Security Dashboard provides:

1. **Centralized security overview**: Integrates information from GuardDuty, Config, and Falco
2. **Real-time alerting**: Rapid detection and response to threats
3. **Automation**: Automated response to security incidents
4. **Compliance monitoring**: Track compliance with security policies
5. **Investigation and analysis**: Store and analyze security events

This dashboard helps security teams gain comprehensive visibility into EKS cluster security posture and respond promptly to threats.