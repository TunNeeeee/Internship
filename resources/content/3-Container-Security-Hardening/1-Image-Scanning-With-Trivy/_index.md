---
title: "Container Image Vulnerability Scanning with Trivy (via Docker)"
date: 2025-07-13
weight: 1
chapter: false
pre: "<b> 3.1 </b>"
---

### Main Content

- [Introduction to Trivy](#introduction-to-trivy)
- [Installing and Using Trivy with Docker on Windows](#installing-and-using-trivy-with-docker-on-windows)
- [Scanning Docker Images and Analyzing Results](#scanning-docker-images-and-analyzing-results)
- [Notes and Best Practices](#notes-and-best-practices)

---

### Introduction to Trivy

**Trivy** is an open-source tool that supports scanning for:

- Security vulnerabilities in container images (CVE)
- Configuration errors in IaC such as Kubernetes YAML, Terraform, etc.
- Standard output formats: table, JSON, SARIF...

---

### Installing and Using Trivy with Docker on Windows

#### ✅ Prerequisites

- [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/) installed (WSL2 or Hyper-V)
- Ensure Docker is running

---

#### 🐳 Running Trivy Directly from Docker

Sample command (scanning `nginx` image):

```powershell
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock `
    -v $env:USERPROFILE\.trivy-cache:/root/.cache/ `
    aquasec/trivy:latest image nginx
```

💡 Explanation:
- --rm: removes container after execution
- -v /var/run/docker.sock:/var/run/docker.sock: allows Trivy to access images on the host machine
- -v $env:USERPROFILE\.trivy-cache:/root/.cache/: mounts cache directory to save time on subsequent scans
- aquasec/trivy:latest: uses the official image from Docker Hub
- image nginx: specifies the command and image name to scan

🖼 Real-world example with a vulnerable image:
```powershell
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock `
    -v $env:USERPROFILE\.trivy-cache:/root/.cache/ `
    aquasec/trivy:latest image nginxinc/nginx-unprivileged:1.24.0-alpine-slim
```
![Image-Scanning-With-Trivy](/images/3.1/3.1.1.png)

### Scanning Docker Images and Analyzing Results

After running the command, you will receive results in table format:
Total: 27 (UNKNOWN: 2, LOW: 2, MEDIUM: 20, HIGH: 3)

| Library     | Vulnerability   | Severity | Installed  | Fixed       | Title                                                      |
|-------------|------------------|----------|------------|-------------|------------------------------------------------------------|
| libcrypto3  | CVE-2024-6119    | HIGH     | 3.1.4-r6   | 3.1.7-r0    | openssl: DoS in X.509 name checks                         |
| nginx       | CVE-2023-44487   | HIGH     | 1.24.0-r1  | 1.24.0-r7   | HTTP/2 vulnerability leading to DDoS attacks              |
| busybox     | CVE-2023-42366   | MEDIUM   | 1.36.1-r5  | 1.36.1-r6   | Heap-buffer-overflow in busybox awk                       |

⚠️ Despite being a slim and popular image, it still contains many security vulnerabilities (HIGH, MEDIUM). This demonstrates that regular scanning is essential.

When running the scan command, you will receive a detailed table of affected packages, severity levels, and patched versions.

Example analysis:
- CVE-2024-6119 (HIGH): affects openssl causing denial of service (DoS)
- CVE-2023-44487 (HIGH): affects HTTP/2, potentially vulnerable to DDoS
- CVE-2023-42366 (MEDIUM): heap buffer overflow error

✅ How to handle:
- Always prioritize fixing CRITICAL and HIGH vulnerabilities
- If possible, use newer versions of the image (1.24.0-r7, 1.25.x, ...)
- Avoid using :latest, pin specific versions for security control

### Notes and Best Practices

Exporting result files:
```powershell
docker run --rm `
  -v /var/run/docker.sock:/var/run/docker.sock `
  -v "$env:USERPROFILE/.trivy-cache:/root/.cache" `
  -v "$PWD:/root/report" `
  aquasec/trivy:latest image -f json -o /root/report/result.json nginx
``` 

➡️ JSON results will be saved at D:\pj_aws\result.json

#### CI/CD Integration:
You can integrate the above scan command into:
- GitHub Actions
- GitLab CI/CD
- Jenkins, CircleCI...

✅ This helps automatically check security with each build or deployment.

#### Additional Best Practices:

1. **Regular Scanning Schedule**
   - Set up automated daily/weekly scans
   - Monitor new CVE databases for updates

2. **Image Selection Strategy**
   - Use official images from trusted registries
   - Prefer minimal base images (alpine, distroless)
   - Keep base images updated regularly

3. **Vulnerability Management**
   - Create vulnerability remediation workflows
   - Set security thresholds (e.g., no HIGH/CRITICAL in production)
   - Document accepted risks for non-fixable vulnerabilities

4. **Integration with Security Tools**
   - Combine with admission controllers in Kubernetes
   - Use with registry scanning solutions
   - Integrate with security incident response processes

🎯 Conclusion: Trivy is an indispensable tool when deploying containers in production. You should scan images regularly, use official images, keep them updated, and integrate CI/CD to ensure system security.