# Lab 0 — Environment Setup Report
**Course:** IKB42603 Cloud Security  
**Lab:** Lab 0 — Environment Setup  
**Platform:** Kali Linux  
**Date Completed:** July 30, 2026  

---

## Overview

This report documents the step-by-step setup of the cloud security lab environment on Kali Linux, following the **IKB42603 Lab 0 Environment Setup Cheatsheet**. The environment includes Docker, AWS CLI v2, kind (Kubernetes in Docker), kubectl, OpenSSL, oathtool, and LocalStack — all tools required for subsequent cloud security labs.

---

## Tools Installed

| Tool | Version Verified |
|------|-----------------|
| Docker | 28.5.2+dfsg4 |
| AWS CLI | v2.36.10 |
| kind | v0.23.0 |
| kubectl | v1.33.4 (Kustomize v5.5.0) |
| OpenSSL | 3.6.2 |
| oathtool | 2.6.14 |
| LocalStack | 4.4.0 |

---

## Step-by-Step Setup

---

### Step 1 — Attempt Docker Installation via Convenience Script (Failed)

**Command:**
```bash
curl -fsSL https://get.docker.com | sh
```

**Outcome:** Installation failed.

The official Docker convenience script was run first. The script attempted to add the Docker repository for the detected OS (Debian/Kali), but Kali Linux uses a rolling release that does not have a standard Debian Release file hosted at the Docker repository URL.

**Error received:**
```
E: The repository 'https://download.docker.com/linux/debian kali-rolling Release' does not have a Release file.
```

> **Evidence:** `1_lab0.png` — First attempt running as normal user; the script executed several `sudo -E sh -c` steps before failing.
<img width="1855" height="306" alt="1_lab0" src="https://github.com/user-attachments/assets/763944f0-ff0b-4f98-9b5e-8f6b45d40b91" />

---

### Step 2 — Second Attempt with sudo Password (Still Failed)

**Command:**
```bash
curl -fsSL https://get.docker.com | sh
```

**Outcome:** Same failure, even after entering the sudo password for `kali`.

A second attempt was made to confirm the failure was not a permissions issue. The script prompted for the `[sudo] password for kali:` and proceeded, but hit the same repository error.

**Error received:**
```
E: The repository 'https://download.docker.com/linux/debian kali-rolling Release' does not have a Release file.
```

> **Evidence:** `2_lab0.png` — Shows the sudo password prompt followed by the same repository error.

**Root Cause:** The `get.docker.com` script does not support Kali Linux's `kali-rolling` release directly. The correct method for Kali is to install `docker.io` from the official Kali repository.

---

### Step 3 — Install Docker via Kali's Official Repository

**Command:**
```bash
sudo apt install -y docker.io
```

**Outcome:** Success — Docker and all dependencies installed.

Using the Kali-native package `docker.io` resolved the repository issue. The package manager resolved and installed 17 packages totalling 78.5 MB (334 MB on disk).

**Packages installed:**
- `docker.io` (main package)
- Dependencies: `containerd`, `docker-buildx`, `docker-cli`, `runc`, `tini-static`, `criu`, `libcompel1`, `libintl-perl`, `libintl-xs-perl`, `libmodule-find-perl`, `libproc-processtable-perl`, `libsort-naturally-perl`, `libterm-readkey-perl`, `needrestart`, `python3-protobuf`, `python3-pycriu`

**Summary output:**
```
Upgrading: 0, Installing: 17, Removing: 0, Not Upgrading: 826
Download size: 78.5 MB
Space needed: 334 MB / 62.7 GB available
```

> **Evidence:** `3_lab0.png` — Shows the `sudo apt install -y docker.io` command with the full package list and install summary.

---

### Step 4 — Handle docker.io Package Configuration Dialog

**Outcome:** Selected **Yes** to remove all Docker data.

During installation, a `debconf` package configuration dialog appeared titled **"Configuring docker.io"**. The dialog explained:

> *"The /var/lib/docker directory contains Docker images, containers, and volumes. If you choose this option, all this data will be permanently removed when the docker.io package is purged."*
>
> *"If you are replacing docker.io with another Docker distribution (like docker-ce), or if you want to keep your data, you should choose not to remove it."*

Since this was a fresh setup with no existing Docker data to preserve, **`<Yes>`** was selected.

> **Evidence:** `4_lab0.png` — Shows the blue `debconf` dialog with the `<Yes>` option highlighted.

---

### Step 5 — Add User to Docker Group and Verify Installation

**Commands:**
```bash
sudo usermod -aG docker $USER
docker --version
docker run --rm hello-world
sudo docker run --rm hello-world
```

**Outcome:** Docker installed and verified working.

**5a. Add user to docker group:**
```bash
sudo usermod -aG docker $USER
```
This command adds the current user (`kali`) to the `docker` group, allowing Docker commands to be run without `sudo` after a session re-login.

**5b. Verify Docker version:**
```bash
docker --version
```
```
Docker version 28.5.2+dfsg4, build 9cc6dea35e9a963f281434761c656fba4ac43aed
```

**5c. Test without sudo (permission denied — expected):**
```bash
docker run --rm hello-world
```
```
docker: permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock
```
This error is expected — the group change requires a new login session to take effect.

**5d. Test with sudo (success):**
```bash
sudo docker run --rm hello-world
```
```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```
The `hello-world` container was successfully pulled and executed, confirming Docker is fully operational.

> **Evidence:** `5_lab0.png` — Shows all four commands and their outputs, including the successful "Hello from Docker!" message.

---

### Step 6 — Install AWS CLI v2

**Commands:**
```bash
curl 'https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip' -o awscliv2.zip
unzip awscliv2.zip && sudo ./aws/install
```

**Outcome:** AWS CLI v2 downloaded and installed successfully.

The AWS CLI v2 installer was downloaded as a ZIP archive (~69.58 MB) directly from Amazon's servers at an average speed of ~6.61 MB/s. After unzipping, the install script was executed with `sudo`.

**Download progress output:**
```
% Total    % Received  Avg Speed
100 69.58M  100 69.58M  6.61M/s   00:10
```

**Unzip output (partial):**
```
Archive: awscliv2.zip
  creating: aws/
  creating: aws/dist/
  inflating: aws/README.md
  inflating: aws/install
  inflating: aws/THIRD_PARTY_LICENSES
```

> **Evidence:** `6_lab0.png` — Shows the `curl` download progress and `unzip` extraction output.

---

### Step 7 — Verify AWS CLI Installation

**Command:**
```bash
aws --version
```

**Outcome:** AWS CLI v2 confirmed installed.

```
aws-cli/2.36.10 Python/3.14.6 Linux/6.19.14+kali-amd64 exe/x86_64.kali.2026
```

| Component | Version |
|-----------|---------|
| AWS CLI | 2.36.10 |
| Python | 3.14.6 |
| Linux Kernel | 6.19.14+kali-amd64 |
| Architecture | x86_64 |

> **Evidence:** `7_lab0.png` — Shows the `aws --version` command and its output.

---

### Step 8 — Install kind (Kubernetes in Docker)

**Commands:**
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
kind --version
```

**Outcome:** kind v0.23.0 installed successfully.

The `kind` binary was downloaded directly from the official kind releases page (6.23 MB at 3.79 MB/s), made executable, and moved to `/usr/local/bin/`.

**Download progress:**
```
100  6.23M  100  6.23M  0  0  3.79M  0  00:01  00:01  --:--:--  0
```

**Version verification:**
```
kind version 0.23.0
```

**Note — kubectl typo encountered:**  
When attempting to install `kubectl`, a typo was made initially:
```bash
sudo apt instaall kubectl -y   # typo — double 'a'
# Error: Invalid operation instaall
```
The correct command was then run:
```bash
sudo apt install kubectl -y    # correct
```

> **Evidence:** `8_lab0.png` — Shows the full kind installation flow and the kubectl typo error followed by the corrected command.

---

### Step 9 — Verify kind and kubectl Versions

**Commands:**
```bash
kind --version
kubectl version --client
```

**Outcome:** Both tools confirmed installed and working.

```
kind version 0.23.0
```

```
Client Version: v1.33.4
Kustomize Version: v5.5.0
```

| Tool | Version |
|------|---------|
| kind | 0.23.0 |
| kubectl | v1.33.4 |
| Kustomize | v5.5.0 |

> **Evidence:** `9_lab0.png` — Shows both version check commands and their clean outputs.

---

### Step 10 — Verify OpenSSL and oathtool

**Commands:**
```bash
openssl version
oathtool --version
```

**Outcome:** Both tools confirmed available (pre-installed on Kali Linux).

**OpenSSL:**
```
OpenSSL 3.6.2 7 Apr 2026 (Library: OpenSSL 3.6.2 7 Apr 2026)
```

**oathtool (OATH Toolkit):**
```
oathtool (OATH Toolkit) 2.6.14
Copyright (C) 2009-2026 Simon Josefsson.
License GPLv3+: GNU GPL version 3 or later
```

These tools are used in later labs for cryptographic operations and TOTP/HOTP token generation.

> **Evidence:** `10_lab0.png` — Shows both `openssl version` and `oathtool --version` outputs.

---

### Step 11 — Deploy LocalStack and Verify Health

**Command:**
```bash
docker run -d --name localstack -p 4566:4566 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  localstack/localstack:4.4.0
```

**Outcome:** LocalStack 4.4.0 pulled from Docker Hub and started successfully.

Since the image was not available locally, Docker pulled all layers from `localstack/localstack:4.4.0`. After the container started, the health endpoint was checked.

**Pull confirmation:**
```
Status: Downloaded newer image for localstack/localstack:4.4.0
Digest: sha256:b52c16663c70b7234f217cb993a339b46686e30a1a5d9279cb5feeb2202f837c
```

**Health check:**
```bash
curl http://127.0.0.1:4566/_localstack/health
```

**Response (all services available):**
```json
{"services": {"acm": "available", "apigateway": "available",
"cloudformation": "available", "cloudwatch": "available",
"config": "available", "dynamodb": "available",
"dynamodbstreams": "available", "ec2": "available",
"events": "available", "firehose": "available",
"iam": "available", "kinesis": "available", "kms": "available",
"lambda": "available", "logs": "available",
"opensearch": "available", "redshift": "available",
"resourcegroupstaggingapi": "available", "route53": "available",
"route53resolver": "available", "s3": "available",
"s3control": "available", "scheduler": "available",
"secretsmanager": "available", "ses": "available",
"sqs": "available", "ssm": "available",
"stepfunctions": "available", "sts": "available",
"support": "available", "swf": "available",
"transcribe": "available"},
"edition": "community", "version": "4.4"}
```

> **Evidence:** `11_lab0.png` — Shows the full Docker pull output and the health endpoint JSON response confirming all AWS services are available.

---

### Step 12 — Test LocalStack Stop and Start

**Commands:**
```bash
docker stop localstack
docker start localstack
```

**Outcome:** LocalStack container stopped and restarted successfully.

This step verified that the LocalStack container can be managed (stopped and restarted) without needing to recreate it each time.

```
localstack    ← output of docker stop
localstack    ← output of docker start
```

> **Evidence:** `12_lab0.png` — Shows both `docker stop` and `docker start` commands returning `localstack` as confirmation.

---

### Step 13 — Create and Test a kind Kubernetes Cluster

**Commands:**
```bash
kind create cluster --name ccse
kubectl cluster-info --context kind-ccse
kubectl get nodes
kind delete cluster --name ccse
```

**Outcome:** Kubernetes cluster created, verified, and deleted successfully.

**Cluster creation output:**
```
Creating cluster "ccse" ...
 ✓ Ensuring node image (kindest/node:v1.30.0) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-ccse"
You can now use your cluster with:
kubectl cluster-info --context kind-ccse
```

**Cluster info:**
```
Kubernetes control plane is running at https://127.0.0.1:43101
CoreDNS is running at https://127.0.0.1:43101/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

**Node status:**
```
NAME                 STATUS   ROLES           AGE   VERSION
ccse-control-plane   Ready    control-plane   14m   v1.30.0
```

**Cluster deletion:**
```
Deleting cluster "ccse" ...
Deleted nodes: ["ccse-control-plane"]
```

> **Evidence:** `13_lab0.png` — Shows the complete cluster lifecycle: create, info, node listing, and deletion.

---

### Step 14 — Configure AWS CLI for LocalStack

**Commands:**
```bash
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
EP='--endpoint-url=http://localhost:4566'
aws $EP sts get-caller-identity
```

**Outcome:** AWS CLI successfully communicates with LocalStack using dummy credentials.

Dummy credentials (`test`/`test`) and the `us-east-1` region were configured to allow the AWS CLI to authenticate against LocalStack. The `EP` variable was set as a shorthand for the LocalStack endpoint URL.

**Note:** A typo was made initially using `ws` instead of `aws`:
```bash
ws $EP sts get-caller-identity
# ws: command not found
```
The correct command was run immediately after.

**STS response from LocalStack:**
```json
{
    "UserId": "AKIAIOSFODNN7EXAMPLE",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```

This confirms the AWS CLI is properly connected to the local mock AWS environment.

> **Evidence:** `14_lab0.png` — Shows all four `aws configure` commands, the `EP` variable assignment, the `ws` typo error, and the successful `aws $EP sts get-caller-identity` response.

---

### Step 15 — Environment Teardown and Cleanup Verification

**Commands:**
```bash
docker start localstack 2>/dev/null || docker run -d --name localstack -p 4566:4566 localstack/localstack
EP='--endpoint-url=http://localhost:4566'
docker ps
kind get clusters
docker rm -f localstack
kind delete clusters --all
```

**Outcome:** Cleanup commands verified; container and cluster management confirmed working.

A conflict error was triggered when attempting to start the already-running container using `docker run`:
```
docker: Error response from daemon: Conflict. The container name "/localstack" is already in use by container "cfbca5c299bdee2294bbdb12ef6fba44529c30ddea386fb90d61e6f2364117fb".
```

**Docker ps output — LocalStack running healthy:**
```
CONTAINER ID  IMAGE                      COMMAND                  CREATED         STATUS               PORTS
2be80d12c632  localstack/localstack:4.4.0  "docker-entrypoint.sh"  57 minutes ago  Up 57 minutes (healthy)  0.0.0.0:4510-4559->4510-4559/tcp, 0.0.0.0:4566->4566/tcp, 5678/tcp
```

```bash
kind get clusters
# No kind clusters found.

docker rm -f localstack
# localstack

kind delete clusters --all
```

> **Evidence:** `15_lab0.png` — Shows the conflict error, `docker ps` healthy output, `kind get clusters` (no clusters), and the force-removal commands.

---

### Step 16 — Docker System Prune (Disk Cleanup)

**Command:**
```bash
docker system prune -f
```

**Outcome:** Unused Docker resources removed; 974.2 MB reclaimed.

This command cleaned up the stopped `kind` network and unused images (the `kindest/node` image from the cluster test in Step 13).

**Prune output:**
```
Deleted Networks:
kind

Deleted Images:
untagged: kindest/node@sha256:047357ac0cfea04663786a612ba1eaba9702bef25227a794b52890dd8bcd692e
deleted: sha256:9319cf209ac58c5f091c9cb183fdd8784e753cfb5b1b3cb6692b26abd8d4efac
deleted: sha256:3d6d117551c9bfd7d3cdf6a6d17b15c8925c5bd389c60fa2e3c484f2b94c82cd
deleted: sha256:9ebea5aa64b29e11213cbde5050502f61c2384ead1a33519cfabc8a6f4063d20

Total reclaimed space: 974.2MB
```

**Final docker ps — LocalStack still running after prune:**
```
CONTAINER ID  IMAGE                      STATUS               PORTS
2be80d12c632  localstack/localstack:4.4.0  Up 58 minutes (healthy)  0.0.0.0:4510-4559->4510-4559/tcp, 0.0.0.0:4566->4566/tcp
```

The `docker system prune -f` only removes stopped/unused containers and images — the running LocalStack container was unaffected.

> **Evidence:** `16_lab0.png` — Shows the full prune output with deleted images and reclaimed space, followed by `docker ps` confirming LocalStack is still healthy.

---

## Summary

All required tools for the IKB42603 Cloud Security lab environment were successfully installed and verified on Kali Linux:

| Step | Task | Status |
|------|------|--------|
| 1–2 | Attempt Docker install via `get.docker.com` script | Failed (kali-rolling unsupported) |
| 3–4 | Install Docker via `sudo apt install docker.io` | Success |
| 5 | Add user to docker group, verify with hello-world | Success |
| 6–7 | Install and verify AWS CLI v2 (v2.36.10) | Success |
| 8–9 | Install kind (v0.23.0) and kubectl (v1.33.4) | Success |
| 10 | Verify OpenSSL (3.6.2) and oathtool (2.6.14) | Success |
| 11–12 | Deploy LocalStack 4.4.0 and verify all services | Success |
| 13 | Create/verify/delete a kind Kubernetes cluster | Success |
| 14 | Configure AWS CLI and test against LocalStack | Success |
| 15–16 | Teardown and disk cleanup | Success |

The environment is fully set up and ready for subsequent IKB42603 Cloud Security lab exercises.

---

*Report prepared by: kali@kali*  
*Evidence images: `1_lab0.png` through `16_lab0.png`*  
*Reference: IKB42603_Lab0_Environment_Setup_Cheatsheet.pdf*
