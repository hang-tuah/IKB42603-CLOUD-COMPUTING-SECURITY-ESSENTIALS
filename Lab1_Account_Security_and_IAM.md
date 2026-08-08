# Lab 1 — Cloud Account Security, Identity & Access Management
**Course:** IKB42603 Cloud Computing Security Essentials
**Lab Duration:** Weeks 1–2 (Session A + Session B)
**Student:** asricloud

---

## Lab Learning Outcomes

By completing this lab, the student is able to:

1. Deploy a local AWS-compatible environment using Docker and LocalStack.
2. Create IAM users, groups, and attach least-privilege policies via the AWS CLI.
3. Generate, list, and rotate/deactivate IAM access keys.
4. Create a Kubernetes cluster and apply namespace-based RBAC controls.
5. Verify that permissions are correctly scoped using `kubectl auth can-i`.

---

## Lab Overview

| Session | Scope | Tasks |
|---------|-------|-------|
| Session A (Week 1) | LocalStack IAM | Tasks 1–4 |
| Session B (Week 2) | Kubernetes RBAC | Tasks 5–7 |

**Tools Required:** Docker, AWS CLI v2, kind, kubectl

---

## Session A — LocalStack IAM

### Environment Setup

Before running any AWS CLI commands, LocalStack must be running as a Docker container.
It acts as a local mock of AWS services so all IAM operations stay on your own machine.

---

### Step 1 — Verify Docker and Start LocalStack

**What we do:** Check the Docker version, then run LocalStack as a background container
and confirm all AWS services are healthy.

**Commands executed:**
```bash
docker --version
sudo docker run -d --name localstack -p 4566:4566 localstack/localstack:3.8.0
curl http://localhost:4566/_localstack/health
```

**Evidence — lab1_1.png:**

![lab1_1](lab1_1.png)

**Result:**
- Docker version **29.7.1** confirmed.
- LocalStack container ID `151202dcf250d1eaae857a9cbc53ccec5879294e5e0a1dba4a965795b9d3ed94` was created.
- Health check returned all services as `"available"`, including `iam`, `sts`, `ec2`, `s3`, and many more.
- LocalStack edition: **community**, version: **3.8.0**.

**Why this matters:** LocalStack simulates the AWS API locally — no real AWS account or
internet access is needed, making it safe to practise IAM operations without incurring costs
or risks.

---

### Step 2 — Configure AWS CLI and Verify Identity

**What we do:** Point the AWS CLI at LocalStack using dummy credentials, then call
`sts get-caller-identity` to confirm the CLI talks to LocalStack successfully.

**Commands executed:**
```bash
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
aws --endpoint-url=http://localhost:4566 sts get-caller-identity
```

**Evidence — lab1_2.png:**

![lab1_2](lab1_2.png)

**Result:**
```json
{
    "UserId": "AKIAIOSFODNN7EXAMPLE",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```

The CLI is operating as the **root** identity inside LocalStack (account `000000000000`).

**Why this matters:** In real AWS, you would never use root credentials for daily work.
This step shows the "before" state — the next tasks create a proper admin identity to
use instead of root.

> **Shortcut used:** The variable `EP='--endpoint-url=http://localhost:4566'` is set
> so that every subsequent command can use `aws $EP ...` instead of typing the full
> endpoint URL each time.

---

### Task 1 — Identity Landscape (Concept Summary)

Before creating resources, it is important to understand the key IAM concepts:

| Identity Concept | What it is | Key Characteristic |
|-----------------|------------|--------------------|
| **Root User** | The account owner identity created when the account is first made | Has unrestricted access; should not be used for daily tasks |
| **IAM User** | A named identity with long-term credentials (password / access keys) | Used for humans or applications that need persistent access |
| **IAM Policy** | A JSON document that defines allowed or denied actions on resources | Attached to users, groups, or roles to grant permissions |
| **IAM Group** | A collection of IAM users that share the same attached policies | Policies on the group are inherited by all members |
| **IAM Role** | A temporary identity assumed by services, EC2 instances, or federated users | No long-term credentials; uses short-lived tokens |

---

### Step 3 — Task 2: Create the Admins Group (Least-Privilege Admin Setup)

**What we do:** Create an IAM group called `Admins`, attach the `AdministratorAccess`
managed policy to it, create the admin user `CloudAdmin_ASRI`, and add the user to the group.

**Commands executed:**
```bash
EP='--endpoint-url=http://localhost:4566'
aws $EP iam create-group --group-name Admins
aws $EP iam attach-group-policy --group-name Admins \
    --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws $EP iam create-user --user-name CloudAdmin_ASRI
aws $EP iam add-user-to-group --group-name Admins --user-name CloudAdmin_ASRI
aws $EP iam get-group --group-name Admins
```

**Evidence — lab1_3.png (group created):**

![lab1_3](lab1_3.png)

**Evidence — lab1_4.png (user created, added to group, group verified):**

![lab1_4](lab1_4.png)

**Key output — `iam create-group`:**
```json
{
    "Group": {
        "Path": "/",
        "GroupName": "Admins",
        "GroupId": "nm06rjr56aa4wvjzxlc1",
        "Arn": "arn:aws:iam::000000000000:group/Admins",
        "CreateDate": "2026-08-05T17:55:32.370000+00:00"
    }
}
```

**Key output — `iam create-user`:**
```json
{
    "User": {
        "Path": "/",
        "UserName": "CloudAdmin_ASRI",
        "UserId": "0vr80rwdokioiye80yer",
        "Arn": "arn:aws:iam::000000000000:user/CloudAdmin_ASRI",
        "CreateDate": "2026-08-05T17:57:21.793000+00:00"
    }
}
```

**Key output — `iam get-group` (verification):**
```json
{
    "Users": [
        {
            "UserName": "CloudAdmin_ASRI",
            "Arn": "arn:aws:iam::000000000000:user/CloudAdmin_ASRI"
        }
    ],
    "Group": {
        "GroupName": "Admins",
        "Arn": "arn:aws:iam::000000000000:group/Admins"
    }
}
```

**Why this matters:** Attaching the policy to the **group** rather than directly to the
user means future admins only need to be added to the group — no policy changes required.
This is the recommended AWS best practice for managing permissions at scale.

---

### Step 4 — Task 3: Create a Scoped Read-Only Analyst User

**What we do:** Create an analyst user `Analyst_ASRI` with only S3 read access,
then verify the correct policy is attached.

**Commands executed:**
```bash
aws $EP iam create-user --user-name Analyst_ASRI
aws $EP iam attach-user-policy --user-name Analyst_ASRI \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws $EP iam list-attached-user-policies --user-name Analyst_ASRI
```

**Evidence — lab1_5.png:**

![lab1_5](lab1_5.png)

**Key output — `iam create-user`:**
```json
{
    "User": {
        "UserName": "Analyst_ASRI",
        "UserId": "n66l364l4d4ir5drpj51",
        "Arn": "arn:aws:iam::000000000000:user/Analyst_ASRI",
        "CreateDate": "2026-08-05T18:11:37.048000+00:00"
    }
}
```

**Key output — `iam list-attached-user-policies`:**
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "AmazonS3ReadOnlyAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
        }
    ]
}
```

**Why this matters:** `Analyst_ASRI` can only read from S3 — it cannot write, delete,
or access any other AWS service. This is the **principle of least privilege** in action:
grant only the permissions needed to do the job, and nothing more.

---

### Step 5 — Task 4: Create, List, and Deactivate Access Keys

**What we do:** Generate an access key for `Analyst_ASRI`, list it to confirm it is active,
then deactivate it to simulate key rotation.

**Commands executed:**
```bash
aws $EP iam create-access-key --user-name Analyst_ASRI
aws $EP iam list-access-keys --user-name Analyst_ASRI
aws $EP iam update-access-key --user-name Analyst_ASRI \
    --access-key-id LKIAQAAAAAAEB2Q7KL5 --status Inactive
```

**Evidence — lab1_6.png:**
<img width="1875" height="752" alt="lab1_6" src="https://github.com/user-attachments/assets/3f760d03-1c0a-45f3-b6ce-a28715161ea9" />

![lab1_6](lab1_6.png)

**Key output — `iam create-access-key`:**
```json
{
    "AccessKey": {
        "UserName": "Analyst_ASRI",
        "AccessKeyId": "LKIAQAAAAAAEB2Q7KL5",
        "Status": "Active",
        "SecretAccessKey": "0CdOqxTVxOlRX9G9WAvnx+Ulk3obdouvvWCGocNM",
        "CreateDate": "2026-08-05T18:26:52+00:00"
    }
}
```

**Key output — `iam list-access-keys` (before deactivation):**
```json
{
    "AccessKeyMetadata": [
        {
            "UserName": "Analyst_ASRI",
            "AccessKeyId": "LKIAQAAAAAAEB2Q7KL5",
            "Status": "Active",
            "CreateDate": "2026-08-05T18:26:52+00:00"
        }
    ]
}
```

After running `update-access-key --status Inactive`, the key can no longer be used
to authenticate but is retained for audit purposes.

**Why this matters:** Regular access key rotation is a critical security practice.
If a key is leaked, deactivating it immediately stops the attacker from using it.
AWS recommends rotating keys every 90 days.

---

## Session B — Kubernetes RBAC

Kubernetes uses **Role-Based Access Control (RBAC)** to control what actions service
accounts or users can perform. RBAC in Kubernetes is similar to IAM in AWS — both follow
the principle of least privilege, scoping permissions to only what is required.

---

### Step 6 — Task 5: Create a kind Cluster and Namespaces

#### 6a — Create the Cluster

**What we do:** Create a local Kubernetes cluster named `ccse-lab1` using kind
(Kubernetes IN Docker), then verify it is running.

**Commands executed:**
```bash
sudo kind create cluster --name ccse-lab1
sudo kubectl cluster-info --context kind-ccse-lab1
sudo kubectl get nodes
```

**Evidence — lab1_7.png:**

![lab1_7](lab1_7.png)

**Key output:**
```
Creating cluster "ccse-lab1" ...
 ✓ Ensuring node image (kindest/node:v1.30.0)
 ✓ Preparing nodes
 ✓ Writing configuration
 ✓ Starting control-plane
 ✓ Installing CNI
 ✓ Installing StorageClass
Set kubectl context to "kind-ccse-lab1"

Kubernetes control plane is running at https://127.0.0.1:41425
CoreDNS is running at https://127.0.0.1:41425/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

NAME                       STATUS   ROLES          AGE   VERSION
ccse-lab1-control-plane    Ready    control-plane  88s   v1.30.0
```

**Why this matters:** kind lets you run a real Kubernetes cluster inside Docker containers
on a local machine — perfect for learning RBAC without needing a cloud provider.

#### 6b — Create Namespaces

**What we do:** Create two namespaces — `dev` and `prod` — to simulate environment
separation. This is a common pattern to isolate workloads.

**Commands executed:**
```bash
sudo kubectl create namespace dev
sudo kubectl create namespace prod
sudo kubectl get namespaces
```

**Evidence — lab1_8.png:**

![lab1_8](lab1_8.png)

**Key output:**
```
namespace/dev created
namespace/prod created

NAME              STATUS   AGE
default           Active   4m27s
dev               Active   88s
kube-node-lease   Active   4m27s
kube-public       Active   4m27s
kube-system       Active   4m27s
local-path-storage Active  4m21s
prod              Active   40s
```

**Why this matters:** Namespaces are the primary isolation boundary in Kubernetes.
By separating `dev` and `prod`, we can apply different RBAC rules to each,
ensuring that a developer working in `dev` cannot accidentally affect `prod`.

---

### Step 7 — Task 6: Create ServiceAccount, Role, and RoleBinding

**What we do:** Create a service account `dev-user` in the `dev` namespace,
create a `Role` named `pod-reader` that allows only read operations on pods,
then bind the role to the service account with a `RoleBinding`.

**Commands executed:**
```bash
sudo kubectl create serviceaccount dev-user -n dev
sudo kubectl create role pod-reader -n dev \
    --verb=get,list,watch --resource=pods
sudo kubectl create rolebinding dev-user-binding -n dev \
    --role=pod-reader --serviceaccount=dev:dev-user
```

**Evidence — lab1_9.png:**

![lab1_9](lab1_9.png)

**Key output:**
```
serviceaccount/dev-user created
role.rbac.authorization.k8s.io/pod-reader created
rolebinding.rbac.authorization.k8s.io/dev-user-binding created
```

**Concept breakdown:**

| Resource | Purpose |
|----------|---------|
| **ServiceAccount** | An identity for processes running inside a pod (similar to an IAM User but for workloads) |
| **Role** | Defines a set of permissions (verbs: get, list, watch) on resources (pods) within a specific namespace |
| **RoleBinding** | Links (binds) a Role to a subject (the `dev-user` ServiceAccount) within the same namespace |

**Why this matters:** The `dev-user` service account now has the minimum permissions
needed to read pod information in the `dev` namespace only. It cannot write, delete,
or access any other namespace.

---

### Step 8 — Task 7: Verify Permissions with `kubectl auth can-i`

**What we do:** Test whether the `dev-user` service account can perform three
specific actions, and confirm that RBAC is correctly scoped.

**Commands executed:**
```bash
SA=system:serviceaccount:dev:dev-user
sudo kubectl auth can-i list pods   -n dev  --as=$SA
sudo kubectl auth can-i delete pods -n dev  --as=$SA
sudo kubectl auth can-i list pods   -n prod --as=$SA
```

**Evidence — lab1_10.png:**

![lab1_10](lab1_10.png)

**Results:**

| Action | Namespace | Result | Reason |
|--------|-----------|--------|--------|
| `list pods` | `dev` | **yes** | The `pod-reader` Role grants `get,list,watch` on pods in `dev` |
| `delete pods` | `dev` | **no** | `delete` is not in the Role — only `get,list,watch` are allowed |
| `list pods` | `prod` | **no** | The RoleBinding is scoped to `dev` only — `prod` is unreachable |

**Why this matters:** This confirms the RBAC configuration works exactly as intended.
The service account is constrained to read-only access in a single namespace.
No privilege escalation is possible.

---

### Step 9 — Verify RoleBinding Configuration (YAML)

**What we do:** Inspect the full YAML definition of the `dev-user-binding` to confirm
all settings are correct before submitting as a deliverable.

**Command executed:**
```bash
sudo kubectl get rolebinding dev-user-binding -n dev -o yaml
```

**Evidence — lab1_11.png:**

![lab1_11](lab1_11.png)

**Full YAML output:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: "2026-08-05T18:40:26Z"
  name: dev-user-binding
  namespace: dev
  resourceVersion: "967"
  uid: 029f41e2-34d5-4a1f-94f3-1b02b2a1348f
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: dev
```

**Key points visible in the YAML:**
- `namespace: dev` — the binding only applies in the `dev` namespace.
- `roleRef.name: pod-reader` — the bound role.
- `subjects[0].name: dev-user` — the service account holding the permission.

This YAML output serves as the **verification deliverable** required by the lab guide.

---

## Short-Answer Questions

### Q1. Why is attaching policies to groups better than attaching them directly to users?

Attaching policies to **groups** instead of individual users makes permission management
much easier and less error-prone. When a new employee joins, you simply add them to the
appropriate group and they immediately inherit all the correct permissions — no need to
manually attach every policy to each new user.

When permissions need to change (e.g., an analyst role gains a new tool), you update
the group policy once and every member automatically gets the updated permission.
If you attached policies directly to users, you would have to update every single user
individually, which is time-consuming and prone to mistakes.

In this lab, `CloudAdmin_ASRI` inherited `AdministratorAccess` purely by being a member
of the `Admins` group. A second admin added later would automatically get the same access
without any extra steps.

---

### Q2. What is the difference between an IAM User and an IAM Role?

| | IAM User | IAM Role |
|--|----------|----------|
| **Credentials** | Long-term (password + access keys) | Short-term (temporary tokens, no permanent keys) |
| **Who uses it** | A human or a long-running application | AWS services (EC2, Lambda), federated users, or cross-account access |
| **How it is used** | Log in once and keep using it | "Assumed" temporarily — expires automatically |
| **Key risk** | Keys can be leaked and used indefinitely | Tokens expire, limiting the window of exposure |

In simple terms: an IAM **User** is like a permanent badge you carry every day.
An IAM **Role** is like a visitor pass — valid only for a limited time and purpose,
then automatically revoked.

---

### Q3. Explain least privilege using the Analyst account, and how it reduces blast radius if compromised.

**Least privilege** means giving an identity only the permissions it needs to do its job —
nothing more.

`Analyst_ASRI` was given only `AmazonS3ReadOnlyAccess`. This means the account can:
- Read and list S3 bucket contents.

It **cannot**:
- Write or delete S3 objects.
- Access EC2, IAM, RDS, or any other AWS service.
- Create or modify any resources.

**Blast radius** refers to the maximum damage an attacker can do if they compromise
an identity. If `Analyst_ASRI`'s credentials were stolen:
- The attacker can only **read** S3 data — they cannot destroy anything, escalate
  privileges, or pivot to other services.
- The damage is contained to read-only S3 access.

Compare this to a scenario where the analyst had `AdministratorAccess` — in that case
a compromise would allow the attacker to create new admin users, delete all resources,
or exfiltrate data from every service. Least privilege directly limits this blast radius.

---

### Q4. In Kubernetes, what is the difference between a Role and a RoleBinding?

A **Role** and a **RoleBinding** are two separate, complementary objects in Kubernetes RBAC:

| Object | What it does |
|--------|-------------|
| **Role** | Defines *what* is allowed — a set of permissions (verbs like `get`, `list`, `watch`) on specific resources (like `pods`) within a namespace |
| **RoleBinding** | Defines *who* gets the permissions — it links (binds) a Role to a subject (a ServiceAccount, User, or Group) |

**Analogy:** A Role is like a job description ("can read pod status"). A RoleBinding is like
a contract that says "this specific employee (dev-user) holds this job description."

In this lab:
- The `pod-reader` Role grants `get`, `list`, `watch` on `pods` in the `dev` namespace.
- The `dev-user-binding` RoleBinding connects `pod-reader` to the `dev-user` ServiceAccount.

Without the RoleBinding, the Role exists but nobody is assigned to it.
Without the Role, there are no permissions to bind.
Both must exist for access to work.

---

### Q5. Why did the developer service account fail to access `prod`, and which security principle does that demonstrate?

The `dev-user` service account failed to list pods in the `prod` namespace because
its `RoleBinding` (`dev-user-binding`) was created **in the `dev` namespace** and
references a `Role` (not a `ClusterRole`).

In Kubernetes, a `Role` and its `RoleBinding` are **namespace-scoped**. Permissions
granted in `dev` do not carry over to `prod` — each namespace is an independent
boundary. There is no `RoleBinding` in the `prod` namespace that refers to `dev-user`,
so access is denied by default.

This demonstrates the **Principle of Least Privilege** and specifically **namespace
isolation as a security boundary**:
- Access is **deny-by-default** — unless a RoleBinding explicitly grants access to
  a namespace, the request is rejected.
- A compromised workload in `dev` cannot directly access or damage resources in `prod`.
- Namespace separation mirrors environment separation (dev vs. production) and is a
  fundamental defence-in-depth strategy in Kubernetes security.

---

## Security Best-Practices Checklist

| # | Check | Status |
|---|-------|--------|
| 1 | Root user is not used for daily tasks — a dedicated admin identity (`CloudAdmin_ASRI`) was created | ✅ Done |
| 2 | Permissions are granted via groups/roles, not directly to individual users (`Admins` group used) | ✅ Done |
| 3 | At least one least-privilege (read-only) identity was created and tested (`Analyst_ASRI` with S3 ReadOnly) | ✅ Done |
| 4 | Access keys were listed and a rotation (deactivate) was demonstrated (key `LKIAQAAAAAAEB2Q7KL5` set to Inactive) | ✅ Done |
| 5 | Kubernetes RBAC blocks an unauthorised action (`delete` in `dev` = no; `list` in `prod` = no) | ✅ Done |

---

## Cleanup & Teardown

After completing the lab, all resources are removed to free up system resources.

**Evidence — lab1_12.png:**

![lab1_12](lab1_12.png)

**Commands executed:**
```bash
# Remove the Kubernetes cluster
sudo kind delete cluster --name ccse-lab1

# Stop and remove LocalStack (first attempt failed without sudo)
sudo docker stop localstack && sudo docker rm localstack
```

**Output:**
```
Deleting cluster "ccse-lab1" ...
Deleted nodes: ["ccse-lab1-control-plane"]

localstack
localstack
```

> **Note:** The first attempt `docker stop localstack && docker rm localstack` failed
> with `permission denied while trying to connect to the Docker API`. This was fixed by
> adding `sudo` to both commands.

**All resources successfully removed.** The system is clean and ready for the next lab.

---

## Expansion Ideas (Advanced Students)

The following sections cover four advanced topics that extend beyond the core lab requirements.

---

### Expansion 1 — Infrastructure as Code with Terraform

**Objective:** Recreate the IAM group, user, and policy attachment from Session A using
a Terraform script pointed at LocalStack. This demonstrates how real-world cloud teams
manage IAM at scale through code rather than manual CLI commands.

#### Step 1 — Write the Terraform Configuration

Navigate to the project directory and create `main.tf`:

**Commands executed:**
```bash
cd ~/Documents/cloudProjectL1
nano main.tf
cat main.tf
```

**Evidence — lab1_13.png (Terraform provider and resources 1–2):**

![lab1_13](lab1_13.png)

**Evidence — lab1_14.png (Terraform resources 3–4):**

![lab1_14](lab1_14.png)

**Full `main.tf` content:**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Point the AWS Provider to LocalStack
provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"
  secret_key                  = "test"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true

  endpoints {
    iam = "http://localhost:4566"
  }
}

# 1. Create the IAM Group
resource "aws_iam_group" "admins" {
  name = "Admins"
}

# 2. Attach AdministratorAccess policy to the group
resource "aws_iam_group_policy_attachment" "admin_attach" {
  group      = aws_iam_group.admins.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
}

# 3. Create your Admin User
resource "aws_iam_user" "cloud_admin" {
  name = "CloudAdmin_ASRI"
}

# 4. Add User to the Admins Group
resource "aws_iam_user_group_membership" "admin_membership" {
  user   = aws_iam_user.cloud_admin.name
  groups = [
    aws_iam_group.admins.name
  ]
}
```

#### Step 2 — Verify LocalStack is Still Running

Before running Terraform, confirm LocalStack is healthy:

**Evidence — lab1_15.png:**

![lab1_15](lab1_15.png)

```bash
curl http://localhost:4566/_localstack/health
```

All services returned `"available"` — LocalStack is ready to accept Terraform requests.

#### Step 3 — Initialise Terraform

**Command executed:**
```bash
terraform init
```

**Evidence — lab1_16.png:**

![lab1_16](lab1_16.png)

**Key output:**
```
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.100.0...
- Installed hashicorp/aws v5.100.0 (signed by HashiCorp)

Terraform has been successfully initialized!
```

Terraform downloaded the AWS provider plugin (v5.100.0) and created a lock file.

#### Step 4 — Run Terraform Plan

**Command executed:**
```bash
terraform plan
```

**Evidence — lab1_17.png (resources: admins group, admin_attach, cloud_admin user):**

![lab1_17](lab1_17.png)

**Evidence — lab1_18.png (resource: admin_membership, plan summary):**

![lab1_18](lab1_18.png)

**Plan summary:**
```
Plan: 4 to add, 0 to change, 0 to destroy.
```

The four resources to be created:
1. `aws_iam_group.admins` — the "Admins" group
2. `aws_iam_group_policy_attachment.admin_attach` — attaches `AdministratorAccess`
3. `aws_iam_user.cloud_admin` — the `CloudAdmin_ASRI` user
4. `aws_iam_user_group_membership.admin_membership` — places `CloudAdmin_ASRI` in `Admins`

**Why Terraform matters:** Instead of running 4 separate `aws iam` CLI commands, a
single `terraform apply` would create all resources consistently and repeatably.
The configuration can be version-controlled, reviewed, and reused across environments —
this is **Infrastructure as Code (IaC)**.

---

### Expansion 2 — Custom MFA Enforcement Policy

**Objective:** Write a custom IAM policy that **denies all actions** unless the user has
authenticated with Multi-Factor Authentication (MFA). Attach it to `Analyst_ASRI` to add
an extra authentication layer.

#### Step 1 — Create the MFA Policy

**Commands executed:**
```bash
EP='--endpoint-url=http://localhost:4566'
aws $EP iam create-policy --policy-name EnforceMFAPolicy \
    --policy-document file://enforce-mfa.json
```

**Evidence — lab1_19.png:**

![lab1_19](lab1_19.png)

**Key output:**
```json
{
    "Policy": {
        "PolicyName": "EnforceMFAPolicy",
        "PolicyId": "ADLDHI8FLG9TFJ59DGPES",
        "Arn": "arn:aws:iam::000000000000:policy/EnforceMFAPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "CreateDate": "2026-08-06T08:02:57.616000+00:00"
    }
}
```

#### Step 2 — Attach the Policy to the Analyst User

**Commands executed:**
```bash
aws $EP iam attach-user-policy --user-name Analyst_ASRI \
    --policy-arn arn:aws:iam::000000000000:policy/EnforceMFAPolicy
aws $EP iam list-attached-user-policies --user-name Analyst_ASRI
```

**Evidence — lab1_20.png:**

![lab1_20](lab1_20.png)

**Key output:**
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "EnforceMFAPolicy",
            "PolicyArn": "arn:aws:iam::000000000000:policy/EnforceMFAPolicy"
        }
    ]
}
```

> **Note:** In this verification, `EnforceMFAPolicy` is the only attached policy shown
> (the earlier `AmazonS3ReadOnlyAccess` was replaced, or both would appear together in
> a real cumulative scenario).

**How the MFA policy works (the `enforce-mfa.json` logic):**
The JSON policy document uses an IAM condition key `aws:MultiFactorAuthPresent`.
If that condition is **not** true (meaning the user did not authenticate with MFA),
then a `Deny` effect blocks all actions. This forces the Analyst to use MFA or
be denied every API call, regardless of what other policies allow.

**Why this matters:** Even if `Analyst_ASRI`'s password or access key is compromised,
the attacker cannot perform any actions without also having the MFA device.
This is a foundational defence against credential theft.

---

### Expansion 3 — ClusterRole and ClusterRoleBinding

**Objective:** Create a `ClusterRole` (cluster-wide permissions, not namespace-scoped)
and a `ClusterRoleBinding` to give `dev-user` the ability to list pods **across all
namespaces**, including `prod`. This demonstrates the difference between namespaced and
cluster-wide RBAC.

#### Step 1 — Create the ClusterRole

**Command executed:**
```bash
sudo kubectl create clusterrole cluster-pod-reader \
    --verb=get,list,watch --resource=pods
```

**Evidence — lab1_21.png:**

![lab1_21](lab1_21.png)

**Output:**
```
clusterrole.rbac.authorization.k8s.io/cluster-pod-reader created
```

#### Step 2 — Create the ClusterRoleBinding

**Command executed (first attempt failed without sudo, second succeeded):**
```bash
sudo kubectl create clusterrolebinding global-dev-user-binding \
    --clusterrole=cluster-pod-reader --serviceaccount=dev:dev-user
```

**Output:**
```
clusterrolebinding.rbac.authorization.k8s.io/global-dev-user-binding created
```

#### Step 3 — Verify Access to `prod` Now Works

**Commands executed:**
```bash
SA=system:serviceaccount:dev:dev-user
sudo kubectl auth can-i list pods -n prod --as=$SA
```

**Evidence — lab1_22.png:**

![lab1_22](lab1_22.png)

**Output:**
```
yes
```

**Before vs After comparison:**

| Scope | Before (RoleBinding only) | After (ClusterRoleBinding added) |
|-------|--------------------------|----------------------------------|
| `list pods` in `dev` | yes | yes |
| `list pods` in `prod` | **no** | **yes** |

**Key insight — Role vs ClusterRole:**

| | Role | ClusterRole |
|--|------|-------------|
| **Scope** | Single namespace | All namespaces (cluster-wide) |
| **Use case** | Limit access to one environment (e.g., `dev`) | Grant access across environments (e.g., a monitoring agent) |
| **Security risk** | Lower — contained blast radius | Higher — a compromise affects the whole cluster |

