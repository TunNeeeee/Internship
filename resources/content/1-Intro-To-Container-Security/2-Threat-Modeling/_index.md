---
title: "Threat Modeling"
date: 2025-07-07
weight: 2
chapter: false
pre: "<b> 1.2 </b>"
---

**Main Contents:**

- [What is the STRIDE model?](#what-is-the-stride-model)
- [Identifying container attack surfaces](#identifying-container-attack-surfaces)
- [Drawing a threat model diagram](#drawing-a-threat-model-diagram)
- [Hands-on threat modeling for a containerized app](#hands-on-threat-modeling-for-a-containerized-app)

---

## What is the STRIDE model?

**STRIDE** is a threat modeling framework that helps identify potential threats in a system. It consists of:

| Component | Threat Type                              |
|-----------|-------------------------------------------|
| S         | Spoofing – Identity impersonation         |
| T         | Tampering – Data manipulation             |
| R         | Repudiation – Denial of actions           |
| I         | Information Disclosure – Data leakage     |
| D         | Denial of Service – Service disruption    |
| E         | Elevation of Privilege – Privilege escalation |

> 🧠 STRIDE helps analyze each system component to uncover potential vulnerabilities.

---

## Identifying container attack surfaces

When deploying containerized applications, you must identify exploitable points, such as:

- **Container Image:** may contain CVEs or malware  
- **Registry:** unprotected → vulnerable to unauthorized push/pull  
- **Kubernetes API:** can be attacked via misconfigured RBAC  
- **Pods running as root:** increase escalation risks  
- **Sensitive information:** hardcoded secrets or ENV vars  

---

## Drawing a threat model diagram

You can use the following tools for visualization:

- ✏️ [Draw.io](https://draw.io)  
- 🔐 [Microsoft Threat Modeling Tool](https://www.microsoft.com/en-us/security/blog/2020/11/18/threat-modeling-tool-updates/)

👉 The diagram should include:

- Users (e.g., developers, attackers)  
- Components: registry, CI/CD, cluster, pods, secrets, DB  
- Annotations of potential STRIDE threats at each point  

📸 *Take a screenshot of your threat model diagram to include in workshop materials.*

---

## Hands-on threat modeling for a containerized app

1. Choose a containerized application to be deployed in the next lab (e.g., nginx + flask + mongo)  
2. Analyze each step: from build → deploy → runtime → networking  
3. List threats using STRIDE and provide mitigation actions

| Step        | Threat (STRIDE)       | Mitigation                                 |
|-------------|------------------------|---------------------------------------------|
| Push image  | Tampering              | Sign images, use a private registry         |
| Run pod     | Elevation of Privilege | Avoid root, drop unnecessary capabilities   |
| Connect DB  | Information Disclosure | Use NetworkPolicy, avoid exposing DB        |

---

📘 **Expected outcomes:**

- Understand common container security threats  
- Be able to model threats for your own application  
- Produce a threat model diagram ready to include in the workshop
