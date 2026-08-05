# Lab 0 — Environment Setup Report
**Course:** IKB42603 Cloud Security  
**Lab:** Lab 0 — Environment Setup  
**Platform:** Ubuntu Linux (VirtualBox VM — `asricloud-virtualbox`)  
**User:** `asricloud`

---

## Overview

This lab sets up a complete cloud security development environment on an Ubuntu virtual machine. By the end of this lab, the following tools will be installed and verified:

| Tool | Purpose | Version Installed |
|---|---|---|
| Docker | Container runtime | 29.7.1 |
| AWS CLI | Interact with AWS / LocalStack | 2.36.15 |
| kind | Local Kubernetes clusters via Docker | 0.23.0 |
| kubectl | Kubernetes command-line client | v1.36.3 |
| OpenSSL | Cryptography toolkit | 3.5.5 |
| oathtool | TOTP/HOTP token generator | 2.6.14 |
| LocalStack | Local AWS cloud emulator | 3.8.0 |

---

## Step 1 — Install Docker

Docker is the foundation of this environment. It is used to run LocalStack and Kubernetes nodes.

### 1.1 Run the Official Docker Install Script

```bash
curl -fsSL https://get.docker.com | sh
```

This single command downloads and executes Docker's official installation script. It automatically:
- Updates the `apt` package index
- Installs required dependencies (`ca-certificates`, `curl`)
- Adds the Docker GPG key and repository
- Installs `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-compose-plugin`, and related extras
- Enables and starts the Docker daemon via `systemd`

**Expected output (at the end of the script):**
```
Using systemd to manage Docker service
INFO: Docker daemon enabled and started
```

### 1.2 Verify Docker Installation

```bash
docker version
```

**Expected output:**
```
Client: Docker Engine - Community
 Version:           29.7.1
 API version:       1.55
 Go version:        go1.26.5
 OS/Arch:           linux/amd64

Server: Docker Engine - Community
 Engine:
  Version:          29.7.1
  API version:      1.55 (minimum version 1.40)
containerd:
  Version:          v2.2.6
runc:
  Version:          1.3.6
docker-init:
  Version:          0.19.0
```

> ✅ **Evidence:** `lab0_1.png` — Docker 29.7.1 client and server both running.

---

## Step 2 — Add User to Docker Group & Test Docker

### 2.1 Add Current User to the Docker Group

By default, Docker requires `sudo`. Adding your user to the `docker` group lets you run Docker commands without it.

```bash
sudo usermod -aG docker $USER
```

> **Note:** You may need to log out and back in (or run `newgrp docker`) for this change to take effect.

### 2.2 Confirm Docker Version

```bash
docker --version
```

**Expected output:**
```
Docker version 29.7.1, build e9452d6
```

### 2.3 Run the Hello-World Test Container

```bash
docker run --rm hello-world
```

This pulls the `hello-world` image from Docker Hub and runs it. The `--rm` flag removes the container automatically after it exits.

**Expected output:**
```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

> ✅ **Evidence:** `lab0_2.png` — Hello-world container ran successfully, confirming Docker is fully operational.

---

## Step 3 — Install AWS CLI v2

The AWS CLI is used to interact with both real AWS services and the LocalStack emulator.

### 3.1 Download the AWS CLI Installer

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
```

### 3.2 Unzip and Install

```bash
unzip awscliv2.zip && sudo ./aws/install
```

This extracts the package and runs the installer, placing the `aws` binary in `/usr/local/bin/`.

### 3.3 Verify the Installation

```bash
aws --version
```

**Expected output:**
```
aws-cli/2.36.15 Python/3.14.6 Linux/7.0.0-14-generic exe/x86_64.ubuntu.26
```

> ✅ **Evidence:** `lab0_3.png` — AWS CLI v2.36.15 installed and verified.

---

## Step 4 — Install kind (Kubernetes in Docker)

`kind` lets you run a local Kubernetes cluster entirely inside Docker containers. It is used for cloud-native security experiments.

### 4.1 Download the kind Binary

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
```

### 4.2 Make it Executable and Move to PATH

```bash
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
```

### 4.3 Verify kind Installation

```bash
kind --version
```

**Expected output:**
```
kind version 0.23.0
```

> ✅ **Evidence:** `lab0_4.png` — kind v0.23.0 downloaded and verified.

> ⚠️ At this point, running `kubectl version --client` returns `Command 'kubectl' not found`. This is expected — kubectl is installed in the next step.

---

## Step 5 — Install kubectl

`kubectl` is the command-line tool for managing Kubernetes clusters.

### 5.1 Install via Snap

```bash
sudo snap install kubectl --classic
```

**Expected output:**
```
kubectl 1.36.3 from Canonical installed
```

### 5.2 Verify Both Tools Are Working

```bash
kind --version
kubectl version --client
```

**Expected output:**
```
kind version 0.23.0
Client Version: v1.36.3
Kustomize Version: v5.8.1
```

> ✅ **Evidence:** `lab0_5.png` — Both kind 0.23.0 and kubectl v1.36.3 confirmed working together.

---

## Step 6 — Verify OpenSSL & Install oathtool

These tools support cryptographic operations used in later cloud security labs.

### 6.1 Check OpenSSL (Pre-installed)

OpenSSL is already present on Ubuntu. Confirm the version:

```bash
openssl version
```

**Expected output:**
```
OpenSSL 3.5.5 27 Jan 2026 (Library: OpenSSL 3.5.5 27 Jan 2026)
```

### 6.2 Install oathtool

`oathtool` generates one-time passwords (TOTP/HOTP) — useful for MFA simulation in security labs.

```bash
sudo apt install -y oathtool
```

**Expected output:**
```
oathtool is already the newest version (2.6.14-1).
oathtool set to manually installed.
```

### 6.3 Verify oathtool

```bash
oathtool --version
```

**Expected output:**
```
oathtool (OATH Toolkit) 2.6.14
Copyright (C) 2009-2026 Simon Josefsson.
```

> ✅ **Evidence:** `lab0_6.png` — OpenSSL 3.5.5 and oathtool 2.6.14 both confirmed.

---

## Step 7 — Run LocalStack (AWS Cloud Emulator)

LocalStack provides a local simulation of AWS services. It runs as a Docker container on port `4566`.

### 7.1 Start the LocalStack Container

```bash
docker run -d --name localstack -p 4566:4566 localstack/localstack:3.8.0
```

Flag breakdown:
- `-d` — runs the container in the background (detached)
- `--name localstack` — assigns a friendly name for easy management
- `-p 4566:4566` — maps port 4566 on your machine to port 4566 in the container
- `localstack/localstack:3.8.0` — uses the pinned stable version 3.8.0

Docker will pull the image layers from Docker Hub on first run.

**Expected output (after all layers pull):**
```
Status: Downloaded newer image for localstack/localstack:3.8.0
cf407ddd70af...
```

### 7.2 Check LocalStack Health

Wait about 30 seconds for LocalStack to initialise, then run:

```bash
curl http://localhost:4566/_localstack/health
```

**Expected output (truncated):**
```json
{"services": {"acm": "available", "apigateway": "available", "cloudformation": "available",
"dynamodb": "available", "ec2": "available", "iam": "available", "kms": "available",
"lambda": "available", "s3": "available", "secretsmanager": "available", "sns": "available",
"sqs": "available", "sts": "available", ...}, "edition": "community", "version": "3.8.0"}
```

All services showing `"available"` confirms LocalStack is running correctly.

> ✅ **Evidence:** `lab0_7.png` — LocalStack 3.8.0 pulled and all AWS services available.

### 7.3 Basic LocalStack Container Management

```bash
docker stop localstack     # Stop the container (preserves data)
docker start localstack    # Start it again
docker rm -f localstack    # Force remove the container completely
```

> ✅ **Evidence:** `lab0_8.png` — stop, start, and remove commands demonstrated.

---

## Step 8 — Configure AWS CLI for LocalStack

To use the AWS CLI against LocalStack instead of real AWS, configure dummy credentials and set an endpoint variable.

### 8.1 Set Dummy AWS Credentials

LocalStack does not validate credentials, but the AWS CLI requires them to be set.

```bash
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
```

### 8.2 Set the LocalStack Endpoint Variable

```bash
EP='--endpoint-url=http://localhost:4566'
```

This variable saves you from typing the endpoint URL every time.

### 8.3 Test the Connection with STS

```bash
aws $EP sts get-caller-identity
```

> ⚠️ **Important:** LocalStack must be running for this to work. If the container is stopped, you will see:
> ```
> aws: [ERROR]: Could not connect to the endpoint URL: "http://localhost:4566/"
> ```

Start LocalStack first if needed:
```bash
docker start localstack
```

**Expected output when LocalStack is running:**
```json
{
    "UserId": "AKIAIOSFODNN7EXAMPLE",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```

> ✅ **Evidence:** `lab0_10.png` — STS call fails when LocalStack is stopped, succeeds after `docker start localstack`.

---

## Step 9 — Create a Kubernetes Cluster with kind

This step creates a local Kubernetes cluster named `ccse` for use in later labs.

### 9.1 Create the Cluster

```bash
kind create cluster --name ccse
```

kind will pull the node image and set up the cluster automatically. This takes about 1–2 minutes.

**Expected output:**
```
Creating cluster "ccse" ...
 ✓ Ensuring node image (kindest/node:v1.30.0)
 ✓ Preparing nodes
 ✓ Writing configuration
 ✓ Starting control-plane
 ✓ Installing CNI
 ✓ Installing StorageClass
Set kubectl context to "kind-ccse"
You can now use your cluster with:

kubectl cluster-info --context kind-ccse

Have a nice day! 👋
```

### 9.2 Verify the Cluster is Running

```bash
kubectl cluster-info --context kind-ccse
```

**Expected output:**
```
Kubernetes control plane is running at https://127.0.0.1:36375
CoreDNS is running at https://127.0.0.1:36375/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

```bash
kubectl get nodes
```

**Expected output:**
```
NAME                 STATUS   ROLES           AGE   VERSION
ccse-control-plane   Ready    control-plane   78s   v1.30.0
```

The node status `Ready` confirms the cluster is fully operational.

### 9.3 Delete the Cluster (Clean Up)

When you no longer need the cluster:

```bash
kind delete cluster --name ccse
```

> ✅ **Evidence:** `lab0_9.png` — ccse cluster created, verified with cluster-info and get nodes, then deleted.

---

## Step 10 — Running Both LocalStack and kind Together

In later labs, you will need both LocalStack and a kind cluster running simultaneously. This step shows how to start them together reliably.

### 10.1 Start LocalStack (with Fallback)

Use this command — it starts the existing container if it exists, or creates a new one if it does not:

```bash
docker start localstack 2>/dev/null || docker run -d --name localstack -p 4566:4566 localstack/localstack:3.8.0
```

### 10.2 Set the Endpoint Variable

```bash
EP='--endpoint-url=http://localhost:4566'
```

### 10.3 Verify Both Are Running

```bash
docker ps
```

**Expected output (both containers visible):**
```
CONTAINER ID   IMAGE                      COMMAND                  STATUS                  PORTS                    NAMES
503ccbd9c702   kindest/node:v1.30.0       "/usr/local/bin/entr…"   Up 10 minutes           127.0.0.1:36375->6443    ccse-control-plane
cf407ddd70af   localstack/localstack:3.8.0  "docker-entrypoint.sh"  Up 5 minutes (healthy)  0.0.0.0:4566->4566/tcp   localstack
```

```bash
kind get clusters
```

**Expected output:**
```
ccse
```

> ✅ **Evidence:** `lab0_11.png` — Both the LocalStack and ccse kind node containers running simultaneously, confirmed via `docker ps` and `kind get clusters`.

---

## Summary — All Tools Verified

| # | Tool | Install Method | Verified Command | Result |
|---|---|---|---|---|
| 1 | Docker 29.7.1 | `curl \| sh` script | `docker version` | ✅ |
| 2 | Docker (no sudo) | `usermod -aG docker` | `docker run --rm hello-world` | ✅ |
| 3 | AWS CLI 2.36.15 | Download zip + install | `aws --version` | ✅ |
| 4 | kind 0.23.0 | `curl` binary download | `kind --version` | ✅ |
| 5 | kubectl v1.36.3 | `snap install kubectl` | `kubectl version --client` | ✅ |
| 6 | OpenSSL 3.5.5 | Pre-installed | `openssl version` | ✅ |
| 7 | oathtool 2.6.14 | `apt install oathtool` | `oathtool --version` | ✅ |
| 8 | LocalStack 3.8.0 | `docker run` | `curl localhost:4566/health` | ✅ |
| 9 | kind cluster (ccse) | `kind create cluster` | `kubectl get nodes` | ✅ |
| 10 | AWS CLI + LocalStack | `aws configure` | `aws $EP sts get-caller-identity` | ✅ |

---

## Common Issues & Fixes

| Problem | Cause | Fix |
|---|---|---|
| `permission denied` running docker | User not in docker group | Run `sudo usermod -aG docker $USER` then log out/in |
| `kubectl` not found | Not yet installed | Run `sudo snap install kubectl --classic` |
| `Could not connect to endpoint URL` | LocalStack not running | Run `docker start localstack` |
| `kind create cluster` fails | Docker not running | Run `sudo systemctl start docker` |
| `aws` command not found | AWS CLI not in PATH | Re-run `sudo ./aws/install` or restart terminal |

---

*Report prepared based on IKB42603 Lab 0 Environment Setup Cheatsheet and lab evidence screenshots (lab0_1.png – lab0_11.png).*
