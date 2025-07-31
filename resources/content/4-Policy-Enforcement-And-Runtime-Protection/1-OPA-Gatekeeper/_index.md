---
title: "Policy Management with OPA Gatekeeper (Windows)"
date: 2025-07-15
weight: 1
chapter: false
pre: "<b> 4.1 </b>"
---

## Main Content

* [Introduction to OPA Gatekeeper](#introduction)
* [Installing Helm on Windows](#installing-helm-on-windows)
* [Deploying Gatekeeper on EKS](#deploying-gatekeeper-on-eks)
* [Writing and testing policies](#writing-and-testing-policies)

---

## Introduction

**OPA (Open Policy Agent)** is a tool used to enforce policies across multiple systems. Gatekeeper is an extension project that helps apply OPA to Kubernetes.

Gatekeeper helps with:

* Controlling deployments based on rules (constraints)
* Logging policy violations
* Preventing unsafe configurations (e.g., banning images with `:latest` tag)

### Key Benefits

- **Admission Control**: Validates resources before they're created
- **Audit Mode**: Identifies existing violations without blocking
- **Custom Policies**: Write policies using Rego language
- **Mutation**: Automatically modify resources to comply with policies

---

## Installing Helm on Windows

### Method 1: Manual Installation

1. Visit [https://github.com/helm/helm/releases](https://github.com/helm/helm/releases)
2. Download file: `helm-v3.x.x-windows-amd64.zip`
3. Extract → Get the `helm.exe` file
4. Create directory e.g.: `C:\Program Files\helm\` → copy `helm.exe` into it
5. Add the path to `Environment Variables > Path`
6. Open PowerShell/CMD and verify:

```bash
helm version
```

### Method 2: Using Package Managers

**Using Chocolatey:**
```powershell
choco install kubernetes-helm
```

**Using Scoop:**
```powershell
scoop install helm
```

**Using winget:**
```powershell
winget install Helm.Helm
```

✅ If version appears, installation is successful.

---

## Deploying Gatekeeper on EKS

### Step 1: Add Gatekeeper Helm repository

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update
```

### Step 2: Install Gatekeeper

```bash
helm install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system \
  --create-namespace
```

### Step 3: Verify pod status

```bash
kubectl get pods -n gatekeeper-system
```

Expected output should show pods in `Running` state:
```
NAME                                             READY   STATUS    RESTARTS   AGE
gatekeeper-audit-7f9c8b8b8b-xxxxx               1/1     Running   0          2m
gatekeeper-controller-manager-xxxxx-xxxxx       1/1     Running   0          2m
gatekeeper-controller-manager-xxxxx-yyyyy       1/1     Running   0          2m
```

📸 Take a screenshot of the Running status.

### Step 4: Verify Gatekeeper installation

```bash
kubectl get crd | grep gatekeeper
kubectl api-resources | grep gatekeeper
```

---

## Writing and testing policies

### Step 1: Create `template.yaml` file

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
      validation:
        properties:
          allowedTags:
            type: array
            items:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredimagetag
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          endswith(container.image, ":latest")
          msg := "Container image must not use 'latest' tag"
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not contains(container.image, ":")
          msg := "Container image must specify a tag"
        }
```

Apply the template:
```bash
kubectl apply -f template.yaml
```

### Step 2: Create `constraint.yaml` file

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
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
  parameters:
    allowedTags: ["v1.0", "stable", "production"]
```

Apply the constraint:
```bash
kubectl apply -f constraint.yaml
```

### Step 3: Test with a violating pod

Create `test-pod.yaml`:
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

Try to apply:
```bash
kubectl apply -f test-pod.yaml
```

➡️ **Expected Result**: Pod will be rejected due to policy violation.

**Error message should be similar to:**
```
error validating data: ValidationError(Pod.spec.containers[0].image): 
Container image must not use 'latest' tag
```

### Step 4: Test with a compliant pod

Create `compliant-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: compliant-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.21
```

Apply:
```bash
kubectl apply -f compliant-pod.yaml
```

✅ **Expected Result**: Pod should be created successfully.

---

## Advanced Policy Examples

### 1. Require resource limits

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequireresources
spec:
  crd:
    spec:
      names:
        kind: K8sRequireResources
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequireresources
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := "Container must specify memory limits"
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := "Container must specify CPU limits"
        }
```

### 2. Enforce security contexts

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiresecuritycontext
spec:
  crd:
    spec:
      names:
        kind: K8sRequireSecurityContext
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiresecuritycontext
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := "Container must run as non-root user"
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged
          msg := "Container must not run in privileged mode"
        }
```

---

## Monitoring and Troubleshooting

### Check constraint status

```bash
kubectl get constraints
kubectl describe k8srequiredimagetag disallow-latest-tag
```

### View audit violations

```bash
kubectl get constraints -o yaml | grep -A 10 violations
```

### Debug policy issues

```bash
# Check Gatekeeper logs
kubectl logs -n gatekeeper-system deployment/gatekeeper-controller-manager

# Check if templates are valid
kubectl get constrainttemplate k8srequiredimagetag -o yaml
```

---

## Best Practices

### Policy Development
1. **Start with Audit Mode**: Test policies without enforcement first
2. **Gradual Rollout**: Apply policies to non-critical namespaces first
3. **Clear Error Messages**: Write descriptive violation messages
4. **Test Thoroughly**: Validate policies with various resource configurations

### Performance Considerations
1. **Efficient Rego**: Write optimized Rego policies
2. **Resource Limits**: Set appropriate limits for Gatekeeper components
3. **Monitoring**: Monitor Gatekeeper performance and admission latency

### Security
1. **Namespace Exclusions**: Exclude system namespaces when appropriate
2. **RBAC**: Implement proper RBAC for policy management
3. **Policy Reviews**: Regular review and update of policies

---

✅ **Notes**:

* Gatekeeper is highly effective in controlling deployments
* Policies can be extended and customized based on requirements
* Integration with CI/CD pipelines helps catch violations early
* Regular monitoring and updates ensure policy effectiveness

---

Next: [4.2 Runtime Monitoring with Falco →](/4-Policy-Enforcement-And-Runtime-Protection/2-Falco-Setup-And-Rule/)