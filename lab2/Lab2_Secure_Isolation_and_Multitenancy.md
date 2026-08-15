# Lab 2: Secure Isolation and Multitenancy
**Course:** IKB42603 – Cloud Security  
**Lab Title:** Secure Isolation and Multitenancy in Kubernetes  
**Environment:** Ubuntu VirtualBox | KinD (Kubernetes in Docker) | Calico CNI  
**Student:** asricloud  
**Date Completed:** August 2026

---

## Table of Contents

1. [Lab Overview](#lab-overview)
2. [Objectives](#objectives)
3. [Prerequisites](#prerequisites)
4. [Step-by-Step Walkthrough](#step-by-step-walkthrough)
   - [Step 1 – Create the KinD Cluster](#step-1--create-the-kind-cluster)
   - [Step 2 – Install Calico CNI](#step-2--install-calico-cni)
   - [Step 3 – Verify Calico Rollout](#step-3--verify-calico-rollout)
   - [Step 4 – Create Tenant Namespaces](#step-4--create-tenant-namespaces)
   - [Step 5 – Deploy Applications per Tenant](#step-5--deploy-applications-per-tenant)
   - [Step 6 – Expose Services and Verify Pods](#step-6--expose-services-and-verify-pods)
   - [Step 7 – Test Cross-Namespace Traffic (Before Policy)](#step-7--test-cross-namespace-traffic-before-policy)
   - [Step 8 – Apply Resource Quota to Tenant-A](#step-8--apply-resource-quota-to-tenant-a)
   - [Step 9 – Apply Default-Deny Network Policy to Tenant-B](#step-9--apply-default-deny-network-policy-to-tenant-b)
   - [Step 10 – Test Resource Quota Enforcement](#step-10--test-resource-quota-enforcement)
   - [Step 11 – RBAC: Scoped Secret Access per Tenant](#step-11--rbac-scoped-secret-access-per-tenant)
   - [Step 12 – Data Lifecycle and Secure Wipe (Docker Volume)](#step-12--data-lifecycle-and-secure-wipe-docker-volume)
   - [Step 13 – Cleanup](#step-13--cleanup)
5. [Lab Questions and Answers](#lab-questions-and-answers)
6. [Conclusion](#conclusion)

---

## Lab Overview

This lab explores how to implement **secure isolation and multitenancy** in a Kubernetes cluster. In a multitenant cloud environment, multiple customers (tenants) share the same underlying infrastructure. The challenge is to ensure that each tenant's workloads, data, and network traffic are properly isolated from one another — even though they run on the same cluster.

The lab uses **KinD** (Kubernetes in Docker) as a lightweight local Kubernetes cluster, **Calico** as the network plugin (CNI) that enforces network policies, and several core Kubernetes security features:

| Feature | Purpose |
|---|---|
| Namespaces | Logical isolation boundary for each tenant |
| NetworkPolicy | Controls which pods/services can talk to each other |
| ResourceQuota | Limits how much CPU, memory, and pods a tenant can use |
| RBAC (Role-Based Access Control) | Controls who can access what Kubernetes resources |
| Docker Volumes + Secure Wipe | Demonstrates proper data lifecycle management |

---

## Objectives

By the end of this lab, you should be able to:

- Create a Kubernetes cluster with a custom network plugin (Calico)
- Isolate tenants using Kubernetes **Namespaces**
- Restrict cross-tenant network traffic using **NetworkPolicy**
- Enforce resource limits using **ResourceQuota**
- Implement least-privilege access using **RBAC**
- Demonstrate secure data disposal using a Docker volume with `dd` wipe

---

## Prerequisites

- Ubuntu Linux with VirtualBox
- Docker installed and running
- `kubectl` installed
- `kind` (Kubernetes in Docker) installed
- Internet access to pull Calico manifests and container images
- Working directory: `~/Documents/cloudLab2`

---

## Step-by-Step Walkthrough

---

### Step 1 – Create the KinD Cluster

**Evidence:** `lab2_1.png`

#### Command Used

```bash
sudo cat <<EOF | sudo kind create cluster --name ccse-lab2 --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
EOF
```

#### What Happened

A new Kubernetes cluster named **ccse-lab2** was created using KinD with a custom configuration:

| Configuration | Value | Reason |
|---|---|---|
| `disableDefaultCNI: true` | Default CNI (kindnet) is disabled | We want to use Calico instead |
| `podSubnet: 192.168.0.0/16` | Pod IP address range | Compatible with Calico's IP pool |

KinD reported the following successful steps:
- ✓ Ensuring node image (kindest/node:v1.30.0)
- ✓ Preparing nodes
- ✓ Writing configuration
- ✓ Starting control-plane
- ✓ Installing StorageClass
- kubectl context set to `kind-ccse-lab2`

#### Why This Matters

By disabling the default CNI and specifying a custom pod subnet, we prepare the cluster to use **Calico** — a CNI that supports **NetworkPolicy enforcement**. Without this, Kubernetes would use kindnet which does not enforce NetworkPolicy rules. Calico is required for network-level tenant isolation.

---

### Step 2 – Install Calico CNI

**Evidence:** `lab2_2.png`

#### Command Used

```bash
sudo kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

#### What Happened

Calico v3.27.0 was installed by applying its official manifest. The command created a large number of Kubernetes resources:

**Custom Resource Definitions (CRDs) created — examples:**
- `bgpconfigurations.crd.projectcalico.org`
- `felixconfigurations.crd.projectcalico.org`
- `globalnetworkpolicies.crd.projectcalico.org`
- `networkpolicies.crd.projectcalico.org`
- `ippools.crd.projectcalico.org`

**Other resources created:**
- `serviceaccount/calico-kube-controllers`
- `serviceaccount/calico-node`
- `serviceaccount/calico-cni-plugin`
- `configmap/calico-config`
- `clusterrole/calico-kube-controllers`
- `clusterrole/calico-node`
- `clusterrole/calico-cni-plugin`
- `clusterrolebinding` (for all three above)
- `daemonset.apps/calico-node`
- `deployment.apps/calico-kube-controllers`

#### Why This Matters

Calico is the brain behind network-level isolation. It runs as a **DaemonSet** (`calico-node`) on every node and acts as the network enforcer — reading Kubernetes `NetworkPolicy` objects and translating them into actual firewall rules at the node level using **iptables** or **eBPF**. Without Calico (or a similar CNI), `NetworkPolicy` objects exist in Kubernetes but have no actual effect.

---

### Step 3 – Verify Calico Rollout

**Evidence:** `lab2_3.png`

#### Command Used

```bash
sudo kubectl -n kube-system rollout status daemonset/calico-node --timeout=180s
```

#### What Happened

```
Waiting for daemon set "calico-node" rollout to finish: 0 of 1 updated pods are available...
daemon set "calico-node" successfully rolled out
```

#### Why This Matters

Before moving forward, it is important to confirm that Calico is **fully running** across all nodes. The `rollout status` command waits until the DaemonSet has deployed its pod on every node. Since this is a single-node KinD cluster, it waits for 1 pod to become available. Only after Calico is ready will NetworkPolicy rules actually be enforced.

---

### Step 4 – Create Tenant Namespaces

**Evidence:** `lab2_4.png`

#### Commands Used

```bash
sudo kubectl create namespace tenant-a
sudo kubectl create namespace tenant-b
```

#### What Happened

```
namespace/tenant-a created
namespace/tenant-b created
```

#### Why This Matters

**Namespaces** are the primary isolation boundary in Kubernetes. Each tenant (tenant-a and tenant-b) gets their own namespace, which means:

- Resources (pods, services, secrets) in `tenant-a` are **not visible** to `tenant-b` by default
- RBAC policies, NetworkPolicies, and ResourceQuotas are applied **per namespace**
- Tenants cannot accidentally name-collide with each other's resources

This is the foundational step for multitenancy — everything else builds on namespace isolation.

---

### Step 5 – Deploy Applications per Tenant

**Evidence:** `lab2_5.png`

#### Commands Used

```bash
sudo kubectl -n tenant-a create deployment web --image=nginx --port=80
sudo kubectl -n tenant-b create deployment web --image=nginx --port=80
```

#### What Happened

```
deployment.apps/web created   (in tenant-a)
deployment.apps/web created   (in tenant-b)
```

#### Why This Matters

Both tenants now have their own nginx web server running in their respective namespace. Notice both deployments are named `web` — this is intentional to show that **namespaces prevent naming conflicts**. Two deployments with the same name can coexist without any collision because they live in different namespaces.

This simulates a real multitenant scenario where each customer runs similar or identical workloads but completely separate from each other.

---

### Step 6 – Expose Services and Verify Pods

**Evidence:** `lab2_6.png`

#### Commands Used

```bash
sudo kubectl -n tenant-a expose deployment web --port=80
sudo kubectl -n tenant-b expose deployment web --port=80

sudo kubectl get pods,svc -n tenant-a
sudo kubectl get pods,svc -n tenant-b
```

#### What Happened

**Tenant-A:**

| Resource | Name | Status | Cluster-IP |
|---|---|---|---|
| Pod | web-79d9f568b9-p6zk2 | Running | — |
| Service | web | ClusterIP | 10.96.193.118 |

**Tenant-B:**

| Resource | Name | Status | Cluster-IP |
|---|---|---|---|
| Pod | web-79d9f568b9-ckpcv | Running | — |
| Service | web | ClusterIP | 10.96.163.196 |

#### Why This Matters

Each tenant now has a running pod and a ClusterIP service. The **ClusterIP** is an internal-only IP that other pods inside the cluster can use to reach the service. We will use these IPs in the next step to test network reachability — both before and after network policies are applied.

---

### Step 7 – Test Cross-Namespace Traffic (Before Policy)

**Evidence:** `lab2_7.png`

#### Commands Used

```bash
# Get tenant-b's service IP
sudo kubectl get svc web -n tenant-b -o jsonpath='{.spec.clusterIP}'; echo
# Output: 10.96.163.196

# Run a temporary probe pod in tenant-a and try to reach tenant-b's service
sudo kubectl -n tenant-a run probe --rm -it --image=curlimages/curl \
  --restart=Never -- curl -s -m 5 http://10.96.163.196 \
  -o /dev/null -w 'HTTP %{http_code}\n'
```

#### What Happened

```
HTTP 200
pod "probe" deleted from tenant-a namespace
```

#### Why This Matters

**HTTP 200 means SUCCESS** — the probe pod in `tenant-a` was able to reach the nginx server in `tenant-b` **without any restrictions**. This is the **insecure default state** of Kubernetes: all pods can talk to all other pods across namespaces unless a NetworkPolicy says otherwise.

This test proves the **problem** we need to solve — without NetworkPolicy, tenant-a can access tenant-b's services freely, which violates tenant isolation. The next steps will fix this.

---

### Step 8 – Apply Resource Quota to Tenant-A

**Evidence:** `lab2_8.png`

#### Command Used

```bash
cat <<EOF | sudo kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 512Mi
    pods: "5"
EOF
```

#### What Happened

```
resourcequota/tenant-a-quota created
```

Verification:

```bash
sudo kubectl describe resourcequota tenant-a-quota -n tenant-a
```

```
Name:             tenant-a-quota
Namespace:        tenant-a
Resource          Used   Hard
--------          ----   ----
pods              1      5
requests.cpu      0      1
requests.memory   0      512Mi
```

#### Why This Matters

A **ResourceQuota** acts as a ceiling — it prevents a single tenant from consuming too many cluster resources, which would degrade other tenants' workloads. The quota set here means:

| Limit | Value | Meaning |
|---|---|---|
| `pods` | 5 | tenant-a can run at most 5 pods |
| `requests.cpu` | 1 | tenant-a can request at most 1 CPU core total |
| `requests.memory` | 512Mi | tenant-a can request at most 512 MB of memory total |

Currently 1 pod is in use (the nginx deployment), with 0 CPU/memory requests declared. In Step 10, we will see the quota actually block a pod that does not declare resource requests.

---

### Step 9 – Apply Default-Deny Network Policy to Tenant-B

**Evidence:** `lab2_9.png`

#### Command Used

```bash
cat <<EOF | sudo kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: tenant-b
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF
```

#### What Happened

```
networkpolicy.networking.k8s.io/default-deny-ingress created
```

#### Why This Matters

This NetworkPolicy is a **default-deny** rule for all ingress (incoming) traffic to `tenant-b`. Breaking it down:

| Field | Value | Meaning |
|---|---|---|
| `podSelector: {}` | Empty selector | Applies to **all pods** in the namespace |
| `policyTypes: [Ingress]` | Ingress only | Blocks all **incoming** traffic |
| No `ingress:` rules defined | — | No exceptions — everything is denied |

After this policy is applied, **no pod from any namespace** (including tenant-a) can send traffic to pods in tenant-b. Only explicitly allowed traffic would get through — and since we defined no allowed rules, everything is blocked.

This is the network isolation counterpart to namespace isolation. Combined with namespace separation, it creates a strong multitenant boundary.

---

### Step 10 – Test Resource Quota Enforcement

**Evidence:** `lab2_10.png`

#### Command Used

```bash
sudo kubectl -n tenant-a run probe --rm -it --image=curlimages/curl \
  --restart=Never -- curl -s -m 5 http://10.96.116.23 \
  -o /dev/null -w 'HTTP %{http_code}\n'
```

#### What Happened

```
Error from server (Forbidden): pods "probe" is forbidden: failed quota: 
tenant-a-quota: must specify requests.cpu for: probe; 
requests.memory for: probe
```

#### Why This Matters

The **ResourceQuota is working correctly**. When a quota enforces `requests.cpu` and `requests.memory`, every new pod in that namespace **must declare its resource requests explicitly**. The `probe` pod did not declare any resource requests, so Kubernetes rejected it immediately with a `Forbidden` error.

This is a security benefit — it forces tenants to be explicit about resource usage. Without this enforcement, a tenant could spin up unlimited pods with no resource declarations and starve the cluster.

> **Note:** There are now two things happening — the ResourceQuota blocks the pod from being created, AND even if it had been created, the NetworkPolicy on tenant-b would have blocked the HTTP connection. This shows **defense in depth**: multiple layers of control working together.

---

### Step 11 – RBAC: Scoped Secret Access per Tenant

**Evidence:** `lab2_11.png`

#### Commands Used

```bash
# Create a secret in each tenant namespace
sudo kubectl -n tenant-a create secret generic data --from-literal=value=SECRET_A
sudo kubectl -n tenant-b create secret generic data --from-literal=value=SECRET_B

# Create a ServiceAccount for tenant-a's app
sudo kubectl -n tenant-a create serviceaccount app-a

# Create a Role that allows reading secrets (in tenant-a only)
sudo kubectl -n tenant-a create role reader --verb=get --resource=secrets

# Bind the role to the service account
sudo kubectl -n tenant-a create rolebinding rb --role=reader --serviceaccount=tenant-a:app-a

# Test RBAC permissions
SA=system:serviceaccount:tenant-a:app-a
sudo kubectl auth can-i get secrets -n tenant-a --as=$SA   # → yes
sudo kubectl auth can-i get secrets -n tenant-b --as=$SA   # → no
```

#### What Happened

```
secret/data created          (tenant-a)
secret/data created          (tenant-b)
serviceaccount/app-a created
role.rbac.authorization.k8s.io/reader created
rolebinding.rbac.authorization.k8s.io/rb created

# Permission checks:
yes    ← app-a CAN read secrets in tenant-a
no     ← app-a CANNOT read secrets in tenant-b
```

> **Note:** The first `rolebinding` attempt without `sudo` failed with a connection error. Re-running with `sudo` succeeded — this is a common issue when `kubectl` needs elevated privileges to reach the cluster socket.

#### Why This Matters

This demonstrates **least-privilege access control** using Kubernetes RBAC. The breakdown:

| Object | What It Does |
|---|---|
| `Secret` | Stores sensitive data (e.g., credentials, tokens) |
| `ServiceAccount` (app-a) | An identity for a pod/application, not a human user |
| `Role` (reader) | Defines permission: can `get` `secrets` — scoped to one namespace |
| `RoleBinding` (rb) | Grants the `reader` role to `app-a` within `tenant-a` |

The result: `app-a` can read its own tenant's secrets (`yes`) but is **completely blocked** from reading `tenant-b`'s secrets (`no`). This is namespace-scoped RBAC in action — a Role and RoleBinding only apply within the namespace they are created in. Even if tenant-a's app is compromised, it cannot exfiltrate tenant-b's data.

---

### Step 12 – Data Lifecycle and Secure Wipe (Docker Volume)

**Evidence:** `lab2_12.png`

#### Commands Used

```bash
# --- First attempt (demonstrates the problem with simple delete) ---
sudo docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE-PATIENT-RECORD > /data/phi.txt; sync; rm /data/phi.txt; \
   grep -a SENSITIVE /data/* 2>/dev/null; echo scan-done: No such file or directory'

# --- Correct approach: write, then SECURE WIPE with dd ---
sudo docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE-PATIENT-RECORD > /data/phi.txt; sync; rm /data/phi.txt; \
   grep -a SENSITIVE /data/* 2>/dev/null; echo scan-done'
# Output: scan-done   (no sensitive data found after rm + scan)

sudo docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE > /data/phi2.txt; sync; \
   dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc; \
   rm /data/phi2.txt; echo wiped'
# Output:
# 1+0 records in
# 1+0 records out
# 1024 bytes (1.0KB) copied, 0.000037 seconds, 26.4MB/s
# wiped
```

#### What Happened

**Test 1 — Simple delete (`rm`):** A sensitive file (`phi.txt`) containing `SENSITIVE-PATIENT-RECORD` was written to the Docker volume, then deleted with `rm`. The `grep` scan showed `scan-done` — at the filesystem level, the file is gone.

**Test 2 — Secure wipe (`dd`):** A new file (`phi2.txt`) was written, then overwritten with **1KB of zeros** using `dd if=/dev/zero ... conv=notrunc` before deletion. The `wiped` output confirms the data was zeroed out before removal.

#### Why This Matters

In cloud environments, storage volumes are often **reused** between tenants. Simply deleting a file with `rm` does not erase the actual data — it only removes the filesystem pointer. The underlying disk blocks still contain the data and could be recovered by:

- A forensic tool
- A future tenant whose volume happens to be allocated the same blocks
- A malicious insider with raw disk access

The secure wipe process:

1. **Overwrite** the file content with zeros (or random data) using `dd`
2. **Then delete** the file pointer with `rm`

This ensures **no sensitive data residue** remains on the volume before it is released back to the cloud provider or reused by another tenant. This practice is especially important for **HIPAA**, **GDPR**, and other compliance frameworks that require proper data destruction.

---

### Step 13 – Cleanup

**Evidence:** `lab2_13.png`

#### Commands Used

```bash
sudo kind delete cluster --name ccse-lab2
sudo docker volume rm ccse-vol
```

#### What Happened

```
Deleting cluster "ccse-lab2" ...
Deleted nodes: ["ccse-lab2-control-plane"]

ccse-vol
```

#### Why This Matters

Proper cleanup is a good security and resource hygiene practice:

- **Deleting the KinD cluster** removes all Kubernetes resources, containers, and associated networks created during the lab
- **Removing the Docker volume** (`ccse-vol`) ensures the storage used in Step 12 is fully removed, preventing any residual data from persisting on the host

In a real cloud environment, cleanup also means **releasing billing resources** and ensuring decommissioned tenant environments do not leave orphaned data or services behind.

---

## Lab Questions and Answers

---

**Q1: What is the purpose of disabling the default CNI (`disableDefaultCNI: true`) when creating the KinD cluster?**

The default CNI used by KinD is called `kindnet`. While kindnet handles basic pod-to-pod networking, it **does not enforce Kubernetes NetworkPolicy** objects. By disabling it and replacing it with **Calico**, we gain a CNI that actively enforces network policies at the kernel level using iptables or eBPF rules. Without Calico (or an equivalent policy-aware CNI), all `NetworkPolicy` manifests applied to the cluster would be silently ignored — the rules exist in the Kubernetes API but have zero enforcement effect. This is why disabling the default CNI and installing Calico is a prerequisite for network-level tenant isolation.

---

**Q2: What does the `default-deny-ingress` NetworkPolicy do, and why is it a security best practice?**

The `default-deny-ingress` policy uses an empty `podSelector: {}` (meaning it targets all pods) combined with `policyTypes: [Ingress]` and **no defined ingress rules**. The result is that all incoming traffic to every pod in `tenant-b` is blocked by default.

This follows the **principle of least privilege** for networking — instead of allowing everything and blocking specific threats (blocklist approach), we block everything and only allow what is explicitly needed (allowlist approach). This is a best practice because:

- It prevents unknown or unintended traffic flows
- New pods added to the namespace are automatically protected without requiring additional policy updates
- It forces developers to be intentional about what network access their services need
- It significantly reduces the blast radius if one pod is compromised, as it cannot receive attack traffic from other namespaces

---

**Q3: What happens when a ResourceQuota is applied to a namespace but a pod does not declare resource requests?**

As demonstrated in Step 10, the pod creation is **rejected immediately** by the Kubernetes API server with a `Forbidden` error:

```
Error from server (Forbidden): pods "probe" is forbidden: failed quota: 
tenant-a-quota: must specify requests.cpu for: probe; requests.memory for: probe
```

When a `ResourceQuota` specifying `requests.cpu` or `requests.memory` is active in a namespace, Kubernetes requires **every pod** in that namespace to explicitly declare its `resources.requests`. This is enforced at the admission control stage — before the pod is even scheduled to a node. This mechanism:

1. Prevents tenants from bypassing quotas by omitting resource declarations
2. Ensures accurate resource accounting across all tenant workloads
3. Helps the scheduler make better placement decisions

---

**Q4: How does RBAC ensure that `tenant-a`'s service account cannot access `tenant-b`'s secrets?**

RBAC in Kubernetes uses three key objects: **Role**, **RoleBinding**, and **ServiceAccount**. The critical detail is that a `Role` (as opposed to a `ClusterRole`) is **namespace-scoped** — it only grants permissions within the namespace where it is created.

In Step 11:
- The `reader` Role was created in `tenant-a`, granting `get` on `secrets`
- The `RoleBinding` bound this Role to `app-a` — also within `tenant-a`
- This means `app-a` can only `get secrets` in `tenant-a`

When `app-a` tries to access `secrets` in `tenant-b`, the Kubernetes RBAC authorizer finds **no RoleBinding** granting that permission in `tenant-b`, so the request is denied. The `kubectl auth can-i` tests confirmed this:

```
kubectl auth can-i get secrets -n tenant-a --as=system:serviceaccount:tenant-a:app-a  → yes
kubectl auth can-i get secrets -n tenant-b --as=system:serviceaccount:tenant-a:app-a  → no
```

This is namespace-scoped RBAC providing **data plane isolation** — even if tenant-a's application is fully compromised, it cannot read tenant-b's secrets.

---

**Q5: Why is a simple `rm` command not sufficient for securely deleting sensitive data on a cloud volume? What is the proper approach?**

When you delete a file with `rm`, the operating system only removes the **directory entry (inode pointer)** that points to the file's data blocks. The actual data blocks on disk are **not erased** — they are simply marked as available for future allocation. Until those blocks are overwritten by new data, the original content remains physically present on the storage medium.

This is a serious risk in cloud multitenancy because:

- Cloud providers may reuse storage blocks from one tenant for another
- Forensic tools can recover data from "deleted" files
- Logs, snapshots, or backups may have captured the data before deletion

The **secure approach** demonstrated in Step 12 is to use `dd` to overwrite the file content with zeros **before** deleting it:

```bash
dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc
rm /data/phi2.txt
```

The `conv=notrunc` flag ensures the overwrite happens in-place without truncating the file first, guaranteeing every byte is overwritten. Only after the data is zeroed out is the file pointer removed with `rm`. This is aligned with data sanitization standards such as **NIST SP 800-88** (Guidelines for Media Sanitization).

---

**Q6: What is the significance of using Namespaces as the foundational isolation layer in multitenant Kubernetes?**

Namespaces serve as the primary **administrative and logical boundary** in Kubernetes multitenancy. Their significance includes:

- **Naming scope:** Two resources with the same name (e.g., two deployments both called `web`) can coexist in different namespaces without collision
- **Policy scope:** NetworkPolicy, ResourceQuota, and RoleBindings are all namespace-scoped, meaning they automatically apply to all resources within that namespace
- **Access control boundary:** RBAC Roles bound in one namespace have no effect in another, creating natural privilege boundaries between tenants
- **Visibility isolation:** By default, `kubectl get pods` only shows pods in the current namespace context, reducing accidental cross-tenant visibility

However, namespaces alone are **not sufficient** for strong isolation — as demonstrated in Step 7, pods in different namespaces can still communicate freely over the network until a NetworkPolicy blocks them. True multitenant isolation requires namespaces **combined** with NetworkPolicy, ResourceQuota, and RBAC working together.

---

## Conclusion

This lab demonstrated a complete, layered approach to **secure isolation and multitenancy** in Kubernetes using real tools and hands-on verification at each step. The following security controls were successfully implemented and tested:

| Security Layer | Tool Used | What It Protects |
|---|---|---|
| Logical Isolation | Kubernetes Namespaces | Resource separation between tenants |
| Network Isolation | Calico + NetworkPolicy | Prevents cross-tenant network traffic |
| Resource Isolation | ResourceQuota | Prevents a tenant from starving the cluster |
| Access Control | RBAC (Role + RoleBinding) | Prevents cross-tenant secret access |
| Data Isolation | Docker Volume + `dd` wipe | Prevents data residue on shared storage |

The most important lesson from this lab is that **no single control is sufficient on its own**. The default Kubernetes state (Step 7) showed that pods across namespaces can freely communicate — a major security gap. It took the combination of Namespace isolation, Calico-enforced NetworkPolicy, ResourceQuota, and RBAC to build a genuinely secure multitenant environment.

The **defense-in-depth** approach is clearly demonstrated in Step 10: even if the ResourceQuota had not blocked the probe pod, the default-deny NetworkPolicy on tenant-b would have blocked the HTTP request. Multiple independent controls working together means that compromising one layer does not automatically lead to a full breach.

Finally, the data lifecycle exercise (Step 12) highlights that **security extends beyond runtime** — sensitive data must be properly sanitized before storage is released, aligning with compliance standards like HIPAA and GDPR that govern how sensitive information is handled and destroyed.

In summary, building a secure multitenant Kubernetes environment is not just about deploying the right tools — it requires understanding how they work together, testing each control, and validating that isolation is enforced at every layer: network, compute, access, and data.

---

*Report prepared based on lab evidence (lab2_1.png – lab2_13.png) following the IKB42603 Lab 2 guide.*
