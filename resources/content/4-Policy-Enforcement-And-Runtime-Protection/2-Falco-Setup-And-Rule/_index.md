---
title: "Runtime Monitoring with Falco (Windows)"
date: 2025-07-15
weight: 2
chapter: false
pre: "<b> 4.2 </b>"
---

## Main Content

* [Introduction to Falco](#introduction-to-falco)
* [Installing Falco with Helm](#installing-falco-with-helm)
* [Testing runtime alerts](#testing-runtime-alerts)

---

## Introduction to Falco

**Falco** is a real-time behavioral monitoring tool for Kubernetes. It helps detect:

* Unusual system access (e.g., `/etc/passwd`)
* Shell usage in containers (`bash`, `sh`)
* Modification of sensitive files
* Network anomalies and suspicious connections
* Privilege escalation attempts

Falco uses kernel syscalls to record behavior and sends alerts based on predefined rules.

### Key Features

- **Real-time Detection**: Monitors syscalls at the kernel level
- **Cloud Native**: Native integration with Kubernetes
- **Flexible Rules**: Customizable rule engine using YAML
- **Multiple Outputs**: Supports various alert destinations
- **CNCF Project**: Graduated Cloud Native Computing Foundation project

### How Falco Works

1. **Kernel Module/eBPF**: Captures system calls from the kernel
2. **Rule Engine**: Evaluates syscalls against security rules
3. **Alert Generation**: Generates alerts for suspicious activities
4. **Output Channels**: Sends alerts to various destinations (logs, webhooks, etc.)

---

## Installing Falco with Helm

### 🔧 Prerequisites:
* **kubectl**, **eksctl**, **AWS CLI** installed and EKS cluster running
* **Helm** installed (see previous section for installation)

### Step 1: Verify Helm installation

```bash
helm version
```

Expected output:
```
version.BuildInfo{Version:"v3.x.x", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.x.x"}
```

### Step 2: Add Falco repository and install

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
```

### Step 3: Create namespace and install Falco

```bash
kubectl create ns falco
helm install falco falcosecurity/falco -n falco
```

### Step 4: Verify installation

```bash
kubectl get pods -n falco
```

Expected output:
```
NAME          READY   STATUS    RESTARTS   AGE
falco-xxxxx   1/1     Running   0          2m
```

📸 *Take a screenshot of Falco pod running in the `falco` namespace for documentation.*

### Step 5: Check Falco configuration

```bash
kubectl get configmap falco -n falco -o yaml
```

This shows the default rules and configuration.

---

## Testing runtime alerts

### Step 1: Create suspicious shell activity

```bash
kubectl run -i --tty attacker --image=alpine --rm --restart=Never -- sh
```

Once inside the container, run:

```sh
# Try to access sensitive files
touch /etc/passwd
cat /etc/shadow
ls /root

# Try to use networking tools
nc -l 8080
```

### Step 2: View Falco logs

```bash
kubectl logs -l app.kubernetes.io/name=falco -n falco --follow
```

✅ **Expected results:**

```log
{
  "output": "File below /etc opened for writing (user=root command=touch /etc/passwd file=/etc/passwd parent=sh gparent=<NA>)",
  "priority": "Error",
  "rule": "Write below etc",
  "time": "2025-07-15T10:30:45.123456789Z",
  "output_fields": {
    "evt.time": 1642248645123456789,
    "user.name": "root",
    "proc.cmdline": "touch /etc/passwd",
    "fd.name": "/etc/passwd"
  }
}
```

📸 *Take a screenshot of Falco logs containing the alert line for the report.*

### Step 3: Test additional scenarios

**Network activity monitoring:**

```bash
kubectl run network-test --image=nicolaka/netshoot --rm -it --restart=Never -- bash
```

Inside the container:
```bash
# These should trigger network-related alerts
nmap 10.0.0.1
nc 8.8.8.8 80
```

**File modification monitoring:**

```bash
kubectl run file-test --image=ubuntu --rm -it --restart=Never -- bash
```

Inside the container:
```bash
# These should trigger file-related alerts
echo "test" > /bin/ls
chmod +x /tmp/suspicious-script
```

### Step 4: Monitor different alert types

```bash
# Monitor all Falco alerts with timestamps
kubectl logs -l app.kubernetes.io/name=falco -n falco --follow --timestamps
```

Common alert patterns you might see:
- **Shell spawned in container**
- **File below /etc opened for writing**
- **Network connection attempted**
- **Sensitive file access**

---

## Understanding Falco Rules

### Default Rule Categories

1. **System Activity**
   - File access patterns
   - Process execution
   - Network connections

2. **Container Security**
   - Shell spawning
   - Privilege escalation
   - Suspicious binaries

3. **Kubernetes Security**
   - Pod operations
   - Service account usage
   - API server interactions

### Custom Rule Example

Create a custom rule file `custom-rules.yaml`:

```yaml
customRules:
  my-rules.yaml: |-
    - rule: Detect cryptocurrency mining
      desc: Detect potential cryptocurrency mining activity
      condition: >
        spawned_process and
        (proc.name in (xmrig, cpuminer, ccminer))
      output: >
        Cryptocurrency mining detected 
        (user=%user.name command=%proc.cmdline container=%container.name)
      priority: CRITICAL
```

Apply the custom rules:

```bash
helm upgrade falco falcosecurity/falco -n falco -f custom-rules.yaml
```

---

## Advanced Configuration

### Enable additional outputs

Update Falco to send alerts to multiple destinations:

```yaml
# values.yaml
falco:
  grpc:
    enabled: true
  grpcOutput:
    enabled: true
  httpOutput:
    enabled: true
    url: "http://webhook-server:8080/falco-alerts"
  jsonOutput: true
  jsonIncludeOutputProperty: true
```

```bash
helm upgrade falco falcosecurity/falco -n falco -f values.yaml
```

### Performance tuning

```yaml
# values.yaml for better performance
falco:
  syscallEventDrops:
    maxBurst: 1000
    rate: 0.03333
  priority: INFO  # Reduce verbosity
  bufferSize: 8  # Increase buffer size
```

---

## Monitoring and Alerting Integration

### Integration with Prometheus

```bash
# Add Falco exporter for Prometheus metrics
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco-exporter falcosecurity/falco-exporter -n falco
```

### Integration with Grafana

Import Falco dashboard ID: `11914` from Grafana.com for visualization.

### Slack Integration

```yaml
# values.yaml for Slack notifications
falco:
  httpOutput:
    enabled: true
    url: "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
```

---

## Troubleshooting

### Common Issues

**1. Falco not starting:**
```bash
kubectl describe pod -l app.kubernetes.io/name=falco -n falco
kubectl logs -l app.kubernetes.io/name=falco -n falco
```

**2. No alerts appearing:**
```bash
# Check if Falco is processing events
kubectl logs -l app.kubernetes.io/name=falco -n falco | grep "Events processed"

# Verify syscall source
kubectl logs -l app.kubernetes.io/name=falco -n falco | grep -i "syscall"
```

**3. High CPU usage:**
```bash
# Check event rate
kubectl top pod -n falco

# Reduce rule verbosity or increase buffer size
```

### Debug Mode

Enable debug logging:

```bash
helm upgrade falco falcosecurity/falco -n falco --set falco.logLevel=debug
```

---

## Best Practices

### Rule Management
1. **Start with default rules** and gradually customize
2. **Test rules** in non-production environments first
3. **Monitor false positives** and tune accordingly
4. **Regular rule updates** to address new threats

### Performance Optimization
1. **Set appropriate log levels** to balance security and performance
2. **Use rule conditions efficiently** to reduce processing overhead
3. **Monitor resource usage** and scale accordingly
4. **Consider event filtering** for high-traffic environments

### Security Considerations
1. **Secure Falco configuration** with proper RBAC
2. **Protect alert channels** from tampering
3. **Regular security updates** for Falco components
4. **Audit rule changes** and maintain version control

---

## Notes

* Falco **detects and alerts** but does not prevent malicious behavior
* For automated response to violations (4.3), you need to combine with Falcosidekick or other response tools
* Regular monitoring of Falco alerts is essential for effective security posture
* Integration with SIEM systems enhances threat detection capabilities

👉 **Next:** [4.3 Automated Response to Violations with Falcosidekick →](../3-Automated-Remediation/)