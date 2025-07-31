---
title: "Overview of Container Security"
date: 2025-07-07
weight: 1
chapter: false
pre: "<b> 1.1 </b>"
---

**Main Contents:**

- [What is a container and why secure it?](#what-is-a-container-and-why-secure-it)
- [Container security layers](#container-security-layers)
- [Final goals of the workshop](#final-goals-of-the-workshop)

---

## What is a container and why secure it?

A container is a method of packaging an application along with its libraries and configurations so it can run anywhere. However, due to its lightweight nature and shared kernel, containers pose several security risks:

- Containers may access host file systems if not properly restricted  
- “Container escape” attacks can compromise the entire node  
- Misconfigurations (e.g., exposed ports, privileged mode) are easily exploited  
- Images from unverified sources may contain malware  

---

## Container security layers

Container security must cover the entire application lifecycle, not just the runtime:

| Stage        | Security Measure                          |
|--------------|--------------------------------------------|
| Build        | Scan images (e.g., Trivy), remove vulnerabilities |
| Deploy       | Apply policy controls (e.g., OPA, limits)  |
| Runtime      | Detect anomalies (e.g., Falco)             |
| Network      | Restrict access (e.g., NetworkPolicy)      |
| Monitoring   | Alerting & incident response               |

---

## Final goals of the workshop

By the end of the workshop, you will be able to:

- Set up an EKS cluster with a containerized application  
- Perform vulnerability scanning and benchmark checks (CIS)  
- Apply NetworkPolicy, OPA, and Falco for container protection  
- Deploy alerting systems and automate incident remediation  
- Understand and apply threat modeling (STRIDE, ATT&CK)

> 🧠 You will take screenshots of each step and include them in the workshop content to guide others in building a comprehensive container security environment.
