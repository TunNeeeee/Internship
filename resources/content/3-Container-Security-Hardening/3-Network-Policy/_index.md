---
title: "Implementing NetworkPolicy in Kubernetes"
date: 2025-07-13
weight: 3
chapter: false
pre: "<b> 3.3 </b>"
---

## Main Content

* [Introduction to NetworkPolicy](#introduction-to-networkpolicy)
* [Creating namespace and sample Pods](#creating-namespace-and-sample-pods)
* [Creating and applying NetworkPolicy](#creating-and-applying-networkpolicy)
* [Testing the implementation results](#testing-the-implementation-results)

---

## Introduction to NetworkPolicy

**NetworkPolicy** is a Kubernetes feature that helps you control network traffic in and out between Pods.

By default, all Pods in a cluster can communicate with each other. When you apply NetworkPolicy, you can block all traffic and only allow specific connections.

### Key Concepts

- **Default Behavior**: Without NetworkPolicy, all Pod-to-Pod communication is allowed
- **Deny by Default**: When a NetworkPolicy is applied, it follows a deny-by-default model
- **Additive Rules**: Multiple NetworkPolicies can be applied to the same Pod, with rules being additive
- **Namespace Scoped**: NetworkPolicies are applied per namespace

---

## Creating namespace and sample Pods

### Step 1: Create a dedicated namespace

```bash
kubectl create ns secure-ns
```

### Step 2: Create a sample Pod (nginx)

```bash
kubectl run nginx --image=nginx -n secure-ns --expose --port=80
```

📌 Note: `--expose` will automatically create a service to make testing access easier.

### Step 3: Verify the deployment

```bash
kubectl get pods,svc -n secure-ns
```

You should see both the nginx pod and service created.

---

## Creating and applying NetworkPolicy

### Step 4: Create `deny-all.yaml` file

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: secure-ns
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

**Policy Explanation:**
- `podSelector: {}` - applies to all pods in the namespace
- `policyTypes: [Ingress]` - controls incoming traffic
- No `ingress` rules specified - denies all ingress traffic

Apply the policy:

```bash
kubectl apply -f deny-all.yaml
```

### Step 5: Create a test Pod outside the namespace

```bash
kubectl run busybox --image=busybox:1.28 --rm -it --restart=Never -- sh
```

Try to `wget` the nginx service:

```bash
wget -qO- nginx.secure-ns.svc.cluster.local
```

❌ Result: The connection will fail — blocked by NetworkPolicy.

---

## Testing the implementation results

### Step 6: Create a policy to allow same-namespace traffic

You can add NetworkPolicies that allow specific IPs or namespaces, for example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-ns
  namespace: secure-ns
spec:
  podSelector:
    matchLabels:
      run: nginx
  ingress:
  - from:
    - podSelector: {}
```

Apply the new policy:

```bash
kubectl apply -f allow-same-ns.yaml
```

### Step 7: Test from within the same namespace

Create a test pod within the secure-ns namespace:

```bash
kubectl run test-pod --image=busybox:1.28 --rm -it -n secure-ns --restart=Never -- sh
```

Try accessing nginx:

```bash
wget -qO- nginx.secure-ns.svc.cluster.local
```

✅ If successful → you have successfully controlled traffic as intended.

---

## Advanced NetworkPolicy Examples

### Allow specific namespaces

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: secure-ns
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Allow specific external IPs

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-ips
  namespace: secure-ns
spec:
  podSelector: {}
  ingress:
  - from:
    - ipBlock:
        cidr: 10.0.0.0/8
        except:
        - 10.0.1.0/24
```

### Egress policy example

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: secure-ns
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53  # Allow DNS
```

---

## Additional Notes

### Prerequisites and Requirements

* Ensure your cluster has CNI plugin support for NetworkPolicy (Amazon EKS supports this out of the box)
* Popular CNI plugins that support NetworkPolicy:
  - **Calico** - Advanced policy features
  - **Cilium** - eBPF-based networking
  - **Weave Net** - Simple setup
  - **Amazon VPC CNI** - Basic NetworkPolicy support

### Best Practices

1. **Start with Deny-All**: Begin with restrictive policies and gradually add exceptions
2. **Label Management**: Use consistent and meaningful labels for policy selectors
3. **Documentation**: Document your network policies and their intended behavior
4. **Testing**: Always test policies in non-production environments first
5. **Monitoring**: Implement monitoring to detect blocked legitimate traffic

### Troubleshooting NetworkPolicy

```bash
# Check if NetworkPolicy is applied
kubectl get networkpolicy -n secure-ns

# Describe the policy for details
kubectl describe networkpolicy deny-all -n secure-ns

# Check pod labels
kubectl get pods --show-labels -n secure-ns

# Test connectivity between pods
kubectl exec -n secure-ns test-pod -- nc -zv nginx 80
```

### Integration with Zero Trust Architecture

* Implementing NetworkPolicy is a crucial step in Zero Trust model for Kubernetes
* Combine with:
  - **Service Mesh** (Istio, Linkerd) for application-layer security
  - **Pod Security Standards** for runtime security
  - **RBAC** for API access control
  - **Admission Controllers** for policy enforcement

### Monitoring and Observability

Consider implementing tools for NetworkPolicy monitoring:
- **Cilium Hubble** - Network observability
- **Calico Enterprise** - Policy visualization
- **Kubernetes Network Policy Viewer** - Policy visualization tools

🎯 **Conclusion**: NetworkPolicy is essential for implementing microsegmentation in Kubernetes. Start with restrictive policies, test thoroughly, and gradually refine your network security posture to achieve Zero Trust networking.