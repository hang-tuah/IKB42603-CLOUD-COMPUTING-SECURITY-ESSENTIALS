# QUIZ - W2 Assessment Report

## Student Information
- **Name:** MUHAMMAD ASRI BIN ROSLI [cite: 1]
- **Student ID:** 52215225028 [cite: 1]
- **Class:** B04 [cite: 1]
- **Email:** asripostfix@gmail.com [cite: 1]
- **Total Score:** 28 / 32 (28 correct out of 30 graded questions) [cite: 1]

---

## Executive Summary
This report documents the submission and performance results for **QUIZ - W2** [cite: 1]. The assessment covers foundational cloud computing concepts, AWS identity and access management (IAM), containerization with Docker, and local Kubernetes orchestration tools. 

---

## Detailed Performance Breakdown

| Q# | Question | Selected Answer | Status | Correct Answer / Notes |
| :---: | :--- | :--- | :---: | :--- |
| 1 | Which service model provides virtual machines? | IaaS [cite: 1] | **Correct** [cite: 1] | IaaS (Infrastructure as a Service) |
| 2 | Which account should never have access keys created for routine use? | Root User [cite: 1] | **Correct** [cite: 1] | Root User |
| 3 | Cloud computing refers to: | Buying more physical servers [cite: 1] | **Incorrect** [cite: 1] | Delivering computing resources over the Internet [cite: 1] |
| 4 | Which endpoint is commonly used with LocalStack? | http://localhost:4566 [cite: 1] | **Correct** [cite: 1] | http://localhost:4566 |
| 5 | Access keys are mainly used for: | Programmatic access [cite: 1] | **Correct** [cite: 1] | Programmatic access |
| 6 | If an access key is compromised, what should be done first? | Deactivate or rotate the key [cite: 1] | **Correct** [cite: 1] | Deactivate or rotate the key |
| 7 | The smallest deployable unit in Kubernetes is: | Pod [cite: 1] | **Correct** [cite: 1] | Pod |
| 8 | Which ARN component identifies the AWS account that owns the resource? | Resource ID [cite: 1] | **Incorrect** [cite: 1] | Account ID (Resource ID identifies the specific resource) |
| 9 | Which command lists Kubernetes nodes? | kubectl get nodes [cite: 1] | **Correct** [cite: 1] | kubectl get nodes |
| 10 | Which tool creates a local Kubernetes cluster? | kind [cite: 1] | **Correct** [cite: 1] | kind |
| 11 | For easier permission management, policies should preferably be attached to: | IAM Groups [cite: 1] | **Correct** [cite: 1] | IAM Groups |
| 12 | Which is NOT an essential characteristic of cloud computing? | Manual Provisioning [cite: 1] | **Correct** [cite: 1] | Manual Provisioning |
| 13 | Which AWS CLI command verifies the current identity? | aws sts get-caller-identity [cite: 1] | **Correct** [cite: 1] | aws sts get-caller-identity |
| 14 | Which characteristic allows cloud resources to automatically grow or shrink? | Rapid Elasticity [cite: 1] | **Correct** [cite: 1] | Rapid Elasticity |
| 15 | In the ARN `arn:aws:s3:::my-bucket`, which component represents the AWS service? | s3 [cite: 1] | **Correct** [cite: 1] | s3 |
| 16 | What does ARN stand for? | Amazon Resource Name [cite: 1] | **Correct** [cite: 1] | Amazon Resource Name |
| 17 | LocalStack is used because it: | Simulates AWS services locally [cite: 1] | **Correct** [cite: 1] | Simulates AWS services locally |
| 18 | Which service model requires customers to manage the operating system? | IaaS [cite: 1] | **Correct** [cite: 1] | IaaS |
| 19 | A node is: | A worker machine [cite: 1] | **Correct** [cite: 1] | A worker machine |
| 20 | Docker is mainly used to: | Run containers [cite: 1] | **Correct** [cite: 1] | Run containers |
| 21 | Which AWS-managed policy provides full administrative access? | AdministratorAccess [cite: 1] | **Correct** [cite: 1] | AdministratorAccess |
| 22 | Which deployment model combines private and public cloud? | Hybrid Cloud [cite: 1] | **Correct** [cite: 1] | Hybrid Cloud |
| 23 | Google Docs is an example of: | SaaS [cite: 1] | **Correct** [cite: 1] | SaaS (Software as a Service) |
| 24 | A collection of IAM users is called: | IAM Group [cite: 1] | **Correct** [cite: 1] | IAM Group |
| 25 | Which IAM component contains permissions? | IAM Policy [cite: 1] | **Correct** [cite: 1] | IAM Policy |
| 26 | Which AWS identity has unlimited privileges? | Root User [cite: 1] | **Correct** [cite: 1] | Root User |
| 27 | A Kubernetes cluster consists of: | Multiple nodes [cite: 1] | **Correct** [cite: 1] | Multiple nodes |
| 28 | Which deployment model provides the MOST control? | Private Cloud [cite: 1] | **Correct** [cite: 1] | Private Cloud |
| 29 | Which IAM identity is normally used as a temporary identity? | IAM Role [cite: 1] | **Correct** [cite: 1] | IAM Role |
| 30 | Which security principle gives users only the permissions required to perform their tasks? | Principle of Least Privilege [cite: 1] | **Correct** [cite: 1] | Principle of Least Privilege |

---

## Key Takeaways & Review Points
1. **Cloud Computing Definition:** Cloud computing relies on delivering computing resources over the internet on-demand rather than purchasing and maintaining physical servers locally.
2. **AWS ARNs Structure:** In Amazon Resource Names (ARNs), the component designating the specific account that owns the resource is the **Account ID**, distinguishing it from the resource identifier itself.
