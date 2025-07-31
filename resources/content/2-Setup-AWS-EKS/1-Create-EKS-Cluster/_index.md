---
title: "Creating EKS Cluster on AWS"
date: 2025-07-07
weight: 1
chapter: false
pre: "<b> 2.1 </b>"
---

**Main Content:**

- [Step 1: Install tools on Windows](#step-1-install-tools-on-windows)
- [Step 2: Create cluster configuration file](#step-2-create-cluster-configuration-file)
- [Step 3: Create cluster using command](#step-3-create-cluster-using-command)
- [Step 4: Verify connection with kubectl](#step-4-verify-connection-with-kubectl)
- [Notes and Important Points](#notes-and-important-points)

---

## Step 1: Install tools on Windows

1. **AWS CLI**
   Visit: [https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-windows.html](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-windows.html) → Download the `.msi` file and install it like any regular software. Then open **CMD** and verify:
   ```powershell
   aws --version
   ```
   If the version appears, the installation was successful.

   ⚙️ Configure AWS account:
   - Create IAM USER
   - Create Access keys/Secret Access Key
   - Open PowerShell
   ```powershell
   aws configure
   ```
   - Then enter the following information:
     - AWS Access Key ID
     - AWS Secret Access Key
     - Region: ap-southeast-1
     - Output format: json

2. **Install kubectl**
   Visit the official link: 👉 https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/
   Or use choco if you have Chocolatey installed:
   ```powershell
   choco install kubernetes-cli
   ```
   Verify the installation:
   ```powershell
   kubectl version --client
   ```

3. **Install eksctl**
   Visit: 👉 https://eksctl.io/introduction/#installation
   Download the .exe file → Rename it to eksctl.exe and:
   - Copy to a directory like C:\Program Files\eksctl
   - Add that directory to Environment Variables > PATH

   Or use choco if you have Chocolatey installed:
   ```powershell
   choco install eksctl
   ```
   - Verify the installation:
   ```powershell
   eksctl version
   ```

## Step 2: Create cluster configuration file

Open Notepad or VS Code, create a file named eks-cluster.yaml with the following content:

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

Save it in the same directory where you'll open the terminal later.

## Step 3: Create cluster using command

Open PowerShell or terminal in VS Code, navigate to the directory containing the eks-cluster.yaml file, then run:

```bash
eksctl create cluster -f eks-cluster.yaml
```

You might encounter this error: Error: cannot find EC2 key pair "~/.ssh/id_rsa.pub"

💡 Cause: In the eks-cluster.yaml configuration file, you have enabled SSH for the node group:

```yaml
ssh:
  allow: true
```

When allow: true, eksctl will try to find the ~/.ssh/id_rsa.pub file to use as the SSH key pair – but you haven't created this key on your Windows machine yet.

You need to manually create the SSH key pair. Open PowerShell and run:

```powershell
ssh-keygen
```

Press Enter repeatedly to accept the default path (C:\Users\<username>\.ssh\id_rsa)

Then verify the file:

```powershell
type $env:USERPROFILE\.ssh\id_rsa.pub
```

Run the cluster creation command again:

```bash
eksctl create cluster -f eks-cluster.yaml
```

📌 eksctl will automatically use the id_rsa.pub file and create the corresponding key pair in AWS EC2.

⏳ The creation process may take 10–15 minutes. eksctl will create: VPC, IAM role, Security Group, Control Plane, EC2 Nodes...

## Step 4: Verify connection with kubectl

After the cluster is created, verify the connection:

```bash
kubectl get nodes
```

✅ You should see 2 worker nodes running – EKS is ready to use!

📸 Take a screenshot of the terminal results to use in your workshop report.

## Notes and Important Points

- eksctl will automatically create VPC and node groups if they don't exist
- It's recommended to use region ap-southeast-1 (Singapore) if you're in Vietnam to reduce latency
- The cluster creation process is fully automated through eksctl
- Make sure your AWS credentials have sufficient permissions to create EKS resources
- Keep your SSH keys secure as they provide access to your worker nodes