---
title: "CIS Benchmark Compliance Check with kube-bench"
date: 2025-07-13
weight: 2
chapter: false
pre: "<b> 3.2 </b>"
---

## Main Content

* [Introduction to CIS Benchmark and kube-bench](#introduction)
* [Deploying kube-bench on EKS](#deployment)
* [Analyzing Check Results and Recommendations](#analyzing-results)

---

## Introduction

**CIS Kubernetes Benchmark** is a standardized security ruleset published by the Center for Internet Security. It includes checks for API Server, Scheduler, Etcd, Kubelet, and more.

**kube-bench** is a tool by Aqua Security that automates the process of checking these security controls on Kubernetes clusters.

---

## Deploying kube-bench on EKS

### Step 1: Create job YAML file

Create a file named `kube-bench-job.yaml` with the following content:

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

### Step 2: Apply the Job to the cluster

```bash
kubectl apply -f kube-bench-job.yaml
```
![CIS-Benchmark-With-KubeBench](/images/3.2/3.2.1.png)

⏳ Wait a few minutes for the Job to complete.

### Step 3: View the check results

```bash
kubectl logs job/kube-bench
```
![CIS-Benchmark-With-KubeBench](/images/3.2/3.2.2.png)
![CIS-Benchmark-With-KubeBench](/images/3.2/3.2.3.png)

---

## Analyzing Results

The output will display check items according to CIS standards, for example:

```
== Summary total ==
4 checks PASS
4 checks FAIL
43 checks WARN
```

### Understanding Result Categories

❌ **[FAIL]** - Critical security issues that must be addressed:
- `--anonymous-auth=false`: disable anonymous access
- `--authorization-mode=Webhook`: configure secure authorization
- `--make-iptables-util-chains=true`: properly handle iptables chains

🔧 **Remediation**: Modify the kubelet.service file on each node according to the guidelines, then run:
```bash
systemctl daemon-reload
systemctl restart kubelet
```

⚠️ **[WARN]** - Security concerns that should be addressed:
- Excessive use of wildcards in RBAC (roles, clusterroles)
- Missing NetworkPolicy for namespaces
- Pod Security Policy not properly configured → allows privileged containers
- IAM / ECR / Secrets not properly configured on AWS

✅ **[PASS]** - Security controls that are properly configured:
- Some kubelet configuration files have correct ownership and permissions (644, root:root)

---

## Additional Notes

### Advanced Usage Options

* kube-bench can run in `host`, `daemonset`, or `job` mode depending on your needs
* Target specific nodes using nodeSelector labels
* Export results to JSON format:

```bash
kubectl logs job/kube-bench -o json > cis-result.json
```

### Integration with CI/CD

You can integrate kube-bench into your automation pipeline:

```yaml
# Example GitHub Actions step
- name: Run CIS Benchmark Check
  run: |
    kubectl apply -f kube-bench-job.yaml
    kubectl wait --for=condition=complete job/kube-bench --timeout=300s
    kubectl logs job/kube-bench > cis-results.txt
```

### Scheduling Regular Checks

Create a CronJob for periodic compliance checks:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: kube-bench-cronjob
spec:
  schedule: "0 2 * * 0"  # Weekly on Sunday at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          # Same spec as above Job
```

---

## Best Practices

✅ **General Best Practices:**
- Always perform regular checks according to CIS Benchmark
- Avoid using default Kubernetes/Kubelet configurations
- Prioritize fixing issues in order: FAIL → WARN → maintain PASS status
- Automate checks by integrating kube-bench into CI/CD or cron jobs

### Remediation Priority Framework

1. **Immediate Action (FAIL)**
   - Critical security misconfigurations
   - Authentication and authorization issues
   - Network security gaps

2. **Planned Remediation (WARN)**
   - RBAC overprivileging
   - Missing security policies
   - Audit logging configuration

3. **Continuous Monitoring (PASS)**
   - Maintain current secure configurations
   - Regular validation of security controls

### EKS-Specific Considerations

- Some CIS controls are managed by AWS and may not be directly configurable
- Focus on node-level and workload-level security controls
- Leverage AWS security services (IAM, VPC, Security Groups) for comprehensive security

🎯 **Conclusion**: Regular CIS Benchmark compliance checking with kube-bench is essential for maintaining a secure Kubernetes environment. Implement automated checking, prioritize remediation based on risk, and maintain continuous security posture improvement.