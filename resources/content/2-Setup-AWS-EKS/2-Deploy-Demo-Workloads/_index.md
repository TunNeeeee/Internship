---

title: "Deploy Sample Workload on EKS"
date: 2025-07-07
weight: 2
chapter: false
pre: "<b> 2.2 </b>"

---

### Table of Contents

* [Create a separate namespace for the demo](#create-a-separate-namespace-for-the-demo)
* [Deploy nginx and backend applications](#deploy-nginx-and-backend-applications)
* [Verify connectivity and prepare for upcoming labs](#verify-connectivity-and-prepare-for-upcoming-labs)

---

### Create a separate namespace for the demo

Namespaces help isolate resources between different applications. First, create a namespace named `demo`:

```bash
kubectl create namespace demo
```
📸 Output:

![Create Namespace Demo](/images/2.2/2.2.1.png)

Then verify the namespace:

```bash
kubectl get ns
```

📸 Output:

![kubectl get ns](/images/2.2/2.2.2.png)

---

### Deploy nginx and backend applications
The sample workload consists of two Pods:

nginx (used as frontend)

http-echo (used as backend)

Create a file named `demo-app.yaml` with the following content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: demo
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: backend
  namespace: demo
spec:
  containers:
    - name: backend
      image: hashicorp/http-echo
      args:
        - "-text=Hello from backend"
      ports:
        - containerPort: 5678
```

▶️ Apply the configuration:

```bash
kubectl apply -f demo-app.yaml
```

📸 Output after applying:

![kubectl apply demo-app](/images/2.2/2.2.3.png)

---

### Verify connectivity and prepare for upcoming labs
Run the following command to check the status of the Pods:

```bash
kubectl get pods -n demo
```

📸 Output:

![kubectl get pods -n demo](/images/2.2/2.2.4.png)

✅ You should see both nginx and backend Pods in Running state.

> This sample workload will be used throughout the upcoming labs such as: NetworkPolicy, Falco, OPA, etc.

---

### Optional: Expose nginx externally

To access the nginx pod from outside (e.g., via browser or curl), you can expose it using a LoadBalancer service:

```bash
kubectl expose pod nginx --port=80 --type=LoadBalancer -n demo
```

Notes:

* This command will create a Service with a public IP (if running on a cloud platform like EKS).
* You can retrieve the access URL using: kubectl get svc -n demo.

---

🎉 **You have successfully deployed a sample workload – ready for the next security-focused labs!**
