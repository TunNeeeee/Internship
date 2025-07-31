---
title: " Penetration Testing"
date: 2025-07-15
weight: 3
chapter: false
pre: "5.3"
---

## Main Content

* [Introduction to Penetration Testing](#introduction)
* [Preparing Test Environment](#preparing-test-environment)
* [Testing Pod and Container Security](#testing-pod-and-container-security)
* [Testing RBAC Configuration](#testing-rbac-configuration)
* [Testing Network Policy](#testing-network-policy)
* [Using Kube-Hunter](#using-kube-hunter)
* [Reporting and Remediation](#reporting-and-remediation)

---

## Introduction

**Penetration Testing** is the process of evaluating system security by simulating real-world attacks. For Kubernetes, we need to test:

* Cluster security configuration
* Access control and authorization
* Container image vulnerabilities
* Network configuration and policies
* Compliance with security standards

---

## Preparing Test Environment

### Step 1: Create dedicated namespace for testing

```bash
kubectl create namespace pentest
kubectl config set-context --current --namespace=pentest
```

### Step 2: Create vulnerable pod for testing

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

## Testing Pod and Container Security

### Step 1: Test privilege escalation

```bash
# Access the pod
kubectl exec -it vulnerable-pod -- /bin/bash

# Check root privileges
whoami
id

# Check host filesystem access capability
ls /host
cat /host/etc/passwd
```

### Step 2: Test container breakout

```bash
# Inside pod, check mount points
mount | grep -E "(proc|sys|dev)"

# Check capabilities
capsh --print

# Try to escape container
chroot /host /bin/bash
```

### Step 3: Create pod with better security context

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

## Testing RBAC Configuration

### Step 1: Create ServiceAccount with high privileges

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

### Step 2: Test access permissions

```bash
# Create pod with ServiceAccount
kubectl run test-rbac --image=bitnami/kubectl:latest \
  --serviceaccount=test-sa \
  --namespace=pentest \
  --rm -it --restart=Never \
  -- /bin/bash

# Inside pod, check permissions
kubectl auth can-i --list
kubectl get secrets --all-namespaces
kubectl get nodes
```

### Step 3: Test ServiceAccount token

```bash
# Get token
kubectl get secret $(kubectl get serviceaccount test-sa -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.token}' | base64 -d

# Use token to access API
curl -k -H "Authorization: Bearer <TOKEN>" https://<API_SERVER>/api/v1/namespaces
```

---

## Testing Network Policy

### Step 1: Create network test pods

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

### Step 2: Test connectivity before policy

```bash
# Get pod IP
kubectl get pods -o wide

# Test connectivity
kubectl exec network-test-1 -- ping -c 3 <IP_OF_POD_2>
kubectl exec network-test-1 -- wget -qO- http://<IP_OF_POD_2>:80
```

### Step 3: Apply Network Policy

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

### Step 4: Test connectivity after policy

```bash
# Test connectivity after policy
kubectl exec network-test-1 -- ping -c 3 <IP_OF_POD_2>
# Result: Timeout (blocked)
```

---

## Using Kube-Hunter

### Step 1: Install Kube-Hunter

```bash
# Run Kube-Hunter in container
kubectl run kube-hunter --image=aquasec/kube-hunter:latest \
  --namespace=pentest \
  --rm -it --restart=Never \
  -- --remote <CLUSTER_IP>
```

### Step 2: Run Kube-Hunter from outside

```bash
# On local machine
pip install kube-hunter

# Scan cluster
kube-hunter --remote <CLUSTER_ENDPOINT>
```

### Step 3: Analyze results

Kube-Hunter will report:
* Exposed services
* Insecure configurations
* Potential attack vectors
* Privilege escalation opportunities

---

## Reporting and Remediation

### Summary of discovered vulnerabilities

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

### Remediation recommendations

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
# Delete test resources
kubectl delete namespace pentest
kubectl delete clusterrolebinding test-binding
```

---

## Workshop Summary

🎉 **Congratulations!** You have completed the **"Container Security on AWS EKS"** workshop

**What you've learned:**
* Setting up secure EKS cluster
* Image scanning with Trivy
* Hardening with CIS Benchmark
* Policy management with OPA Gatekeeper
* Monitoring with Falco
* Logging with CloudWatch
* Penetration testing

**Next steps:**
* Apply to production environment
* Set up CI/CD pipeline with security scanning
* Automated monitoring and alerting
* Compliance with security standards

---

✅ **Important Notes:**

* Penetration testing should only be performed in test environments
* Always have backups before testing
* Comply with legal regulations regarding security testing
* Report and remediate discovered vulnerabilities immediately

---

**Thank you for participating in the workshop!** 🚀