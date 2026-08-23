# Lab 3: Encryption and Key Management

| | |
|---|---|
| **Course** | IKB42603 – Cloud Security |
| **Lab Title** | Encryption and Key Management |
| **Environment** | Ubuntu VirtualBox (`asricloud@asricloud-virtualbox`) |
| **Working Directory** | `~/Documents/cloudLab3` |
| **Simulation Tool** | LocalStack (AWS KMS on `http://localhost:4566`) |

---

> ⚠️ **Confidentiality Notice**
>
> Sensitive values in this report — including KMS Key IDs, ARNs, AWS Account IDs, ciphertext blobs, cryptographic key material, Docker container IDs, Docker image digests, SHA-256 hash outputs, and patient record content — have been **redacted** and replaced with `[REDACTED]` to prevent unintended disclosure.

---

## Table of Contents

1. [Objectives](#objectives)
2. [Prerequisites](#prerequisites)
3. [Lab Overview](#lab-overview)
4. [Session A — File-Level Encryption with OpenSSL](#session-a--file-level-encryption-with-openssl)
   - [Task 1 — Symmetric Encryption (AES-256-CBC)](#task-1--symmetric-encryption-aes-256-cbc)
   - [Task 2 — Asymmetric Encryption (RSA-2048)](#task-2--asymmetric-encryption-rsa-2048)
   - [Task 3 — Digital Signatures](#task-3--digital-signatures)
   - [Task 4 — TLS-Secured NGINX Web Server](#task-4--tls-secured-nginx-web-server)
5. [Session B — Cloud Key Management with AWS KMS](#session-b--cloud-key-management-with-aws-kms)
   - [Task 5 — Create Customer Master Keys (CMKs)](#task-5--create-customer-master-keys-cmks)
   - [Task 6 — Envelope Encryption](#task-6--envelope-encryption)
   - [Task 7 — Key Lifecycle Management](#task-7--key-lifecycle-management)
   - [Task 8 — Data Integrity with SHA-256](#task-8--data-integrity-with-sha-256)
   - [Task 9 — Hash Chaining (Audit Log Integrity)](#task-9--hash-chaining-audit-log-integrity)
6. [Assessment Questions and Answers](#assessment-questions-and-answers)
7. [Cleanup and Teardown](#cleanup-and-teardown)
8. [Conclusion](#conclusion)

---

## Objectives

By the end of this lab, you will be able to:

- Apply **symmetric encryption** (AES-256-CBC) using OpenSSL to protect data at rest.
- Apply **asymmetric encryption** (RSA-2048) using OpenSSL for secure key exchange.
- Create and verify **digital signatures** to prove data authenticity and integrity.
- Configure a **TLS-secured NGINX web server** using Docker to protect data in transit.
- Use **AWS KMS** (via LocalStack) to create and manage Customer Master Keys (CMKs).
- Implement **envelope encryption** — wrapping a data key with a CMK.
- Demonstrate **key lifecycle management** — disabling and scheduling deletion.
- Verify **data integrity** using SHA-256 hashing and hash chaining.

---

## Prerequisites

| Tool | Purpose |
|------|---------|
| Ubuntu VirtualBox | Lab execution environment |
| OpenSSL | Symmetric/asymmetric encryption, TLS certificates |
| Docker | Running NGINX (TLS server) and LocalStack |
| AWS CLI | Communicating with KMS via LocalStack |
| LocalStack | Simulates AWS KMS locally — no real AWS account needed |

---

## Lab Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  SESSION A — File-Level Encryption (OpenSSL)                    │
│  Task 1: AES-256-CBC symmetric encryption                       │
│  Task 2: RSA-2048 asymmetric encryption                         │
│  Task 3: Digital signatures                                     │
│  Task 4: TLS NGINX web server (data in transit)                 │
├─────────────────────────────────────────────────────────────────┤
│  SESSION B — Cloud Key Management (AWS KMS via LocalStack)      │
│  Task 5: Create CMKs for Tenant A and Tenant B                  │
│  Task 6: Envelope encryption with data key wrapping             │
│  Task 7: Key lifecycle — disable, delete, cancel                │
│  Task 8: SHA-256 file integrity verification                    │
│  Task 9: Hash chaining for tamper-evident audit logs            │
└─────────────────────────────────────────────────────────────────┘
```

---

# SESSION A — File-Level Encryption with OpenSSL

---

## Task 1 — Symmetric Encryption (AES-256-CBC)

### 💡 What is Symmetric Encryption?

Symmetric encryption uses the **same key (password)** to both **encrypt** and **decrypt** data. It is fast and ideal for protecting large files at rest.

- **AES** = Advanced Encryption Standard
- **256** = 256-bit key (very strong — industry standard)
- **CBC** = Cipher Block Chaining — each block is scrambled using the result of the previous block, making patterns impossible to detect

---

### Step 1 — Create a sensitive record file

**Command:**
```bash
echo 'Patient: [REDACTED]' > record.txt
ls
```

**Evidence — `lab_3_1.png`:**

![lab_3_1](lab_3_1.png)

**What happened:**
A plain-text file called `record.txt` was created containing a simulated confidential patient record. The `ls` command confirms the file now exists in the working directory.

---

### Step 2 — Encrypt the file with AES-256-CBC

**Command:**
```bash
openssl enc -aes-256-cbc -pbkdf2 -salt -in record.txt -out record.enc
```

**What each flag does:**

| Flag | Meaning |
|------|---------|
| `enc` | Activate OpenSSL encryption mode |
| `-aes-256-cbc` | Use AES algorithm with 256-bit key in CBC mode |
| `-pbkdf2` | Strengthen the password using PBKDF2 key derivation |
| `-salt` | Add a random salt — prevents rainbow-table attacks |
| `-in record.txt` | Source plaintext file |
| `-out record.enc` | Destination encrypted file |

> OpenSSL will prompt: `enter AES-256-CBC encryption password:` and `Verifying - enter AES-256-CBC encryption password:`. Enter the same password both times.

**View the encrypted result:**
```bash
cat record.enc
```

**Evidence — `lab_3_1.png`:**

![lab_3_1](lab_3_1.png)

**What happened:**
The terminal shows `Salted__` followed by garbled binary characters — the original content is completely unreadable. The file is now protected by the password.

```
Salted__[REDACTED — encrypted binary output]
```

---

### Step 3 — Decrypt the file and verify it matches the original

**Command:**
```bash
openssl enc -d -aes-256-cbc -pbkdf2 -in record.enc -out record.dec.txt
diff record.txt record.dec.txt && echo 'MATCH: decryption successful'
```

**Evidence — `lab_3_2.png`:**

![lab_3_2](lab_3_2.png)

**What happened:**
- OpenSSL decrypted `record.enc` back to `record.dec.txt` using the same password.
- The `diff` command compared both files and found **zero differences**.
- Output confirmed:

```
MATCH: decryption successful
```

> ✅ This proves AES-256-CBC encryption and decryption work correctly — the file is perfectly restored.

---

## Task 2 — Asymmetric Encryption (RSA-2048)

### 💡 What is Asymmetric Encryption?

Asymmetric encryption uses a **key pair**:
- **Public key** — can be shared with anyone. Used to **encrypt**.
- **Private key** — must be kept secret. Used to **decrypt**.

RSA-2048 means the key is 2048 bits long. Even knowing the public key, it is computationally infeasible to derive the private key.

---

### Step 4 — Generate an RSA-2048 key pair

**Command:**
```bash
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

**Evidence — `lab_3_3.png`:**

![lab_3_3](lab_3_3.png)

**What happened:**
- `private.pem` — the private key was generated (**keep this secret**)
- `public.pem` — the public key was extracted from the private key (safe to share)
- Terminal confirmed: `writing RSA key`

---

### Step 5 — Encrypt with public key, decrypt with private key

**Encrypt:**
```bash
openssl pkeyutl -encrypt -pubin -inkey public.pem -in record.txt -out record.rsa
```

**Decrypt:**
```bash
openssl pkeyutl -decrypt -inkey private.pem -in record.rsa -out record.rsa.txt
```

**Evidence — `lab_3_3.png`:**

![lab_3_3](lab_3_3.png)

**What happened:**
- The record was encrypted into `record.rsa` using the public key — the file is now unreadable binary.
- It was then decrypted back into `record.rsa.txt` using the private key.
- **Only the holder of `private.pem` can decrypt** — even if `public.pem` is distributed publicly.

---

## Task 3 — Digital Signatures

### 💡 What is a Digital Signature?

A digital signature guarantees two things:
1. **Authentication** — the message genuinely came from the claimed sender (who has the private key).
2. **Integrity** — the message has not been modified since it was signed.

Process: **Sign** with private key → **Verify** with public key.

---

### Step 6 — Sign the record with the private key

**Command:**
```bash
openssl dgst -sha256 -sign private.pem -out record.sig record.txt
```

**Evidence — `lab_3_3.png`:**

![lab_3_3](lab_3_3.png)

**What happened:**
OpenSSL computed a SHA-256 hash of `record.txt`, then encrypted that hash with `private.pem`. The result is stored in `record.sig` — the digital signature file.

---

### Step 7 — Verify the signature with the public key

**Command:**
```bash
openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```

**Evidence — `lab_3_3.png`:**

![lab_3_3](lab_3_3.png)

**Output:**
```
Verified OK
```

**What happened:**
OpenSSL recomputed the SHA-256 hash of `record.txt` and decrypted the signature using `public.pem`. Since both hashes matched, it confirmed:
- The file has **not been tampered** with since signing.
- The signature was created by the holder of `private.pem`.

> ✅ Any modification to `record.txt` after signing would cause verification to **fail**.

---

## Task 4 — TLS-Secured NGINX Web Server

### 💡 What is TLS?

**TLS (Transport Layer Security)** encrypts data **while it travels** across a network. HTTPS = HTTP protected by TLS. Without TLS, anyone on the same network can read transmitted data in plaintext.

---

### Step 8 — Generate a self-signed TLS certificate

**Command:**
```bash
openssl req -x509 -newkey rsa:2048 \
  -keyout key.pem -out cert.pem \
  -days 7 -nodes -subj '/CN=localhost'
```

**What each flag does:**

| Flag | Meaning |
|------|---------|
| `-x509` | Create a self-signed certificate (no Certificate Authority needed) |
| `-newkey rsa:2048` | Generate a fresh 2048-bit RSA key at the same time |
| `-keyout key.pem` | Save the server's private key to `key.pem` |
| `-out cert.pem` | Save the certificate to `cert.pem` |
| `-days 7` | Certificate is valid for 7 days |
| `-nodes` | Do **not** encrypt the private key with a password |
| `-subj '/CN=localhost'` | Set the certificate's domain name to `localhost` |

**Evidence — `lab_3_4.png`:**

![lab_3_4](lab_3_4.png)

**What happened:**
OpenSSL displayed the key generation progress (`.` and `+` characters), then produced:
- `cert.pem` — the TLS certificate
- `key.pem` — the server's private key

---

### Step 9 — First Docker attempt fails (no sudo)

**Command (failed):**
```bash
docker run --rm -d --name tls -p 8443:443 \
  -v $(pwd)/cert.pem:/etc/nginx/cert.pem \
  -v $(pwd)/key.pem:/etc/nginx/key.pem \
  -v $(pwd)/record.txt:/usr/share/nginx/html/record.txt nginx
```

**Evidence — `lab_3_4.png`:**

![lab_3_4](lab_3_4.png)

**Error received:**
```
permission denied while trying to connect to the docker API
at unix:///var/run/docker.sock
```

**Why it failed:**
The Docker daemon socket (`/var/run/docker.sock`) requires root-level access. The user `asricloud` is not in the `docker` group, so the command was rejected.

**Fix:** Add `sudo` to the command.

---

### Step 10 — Pull NGINX image and start container (with sudo)

**Command:**
```bash
sudo docker run --rm -d --name tls -p 8443:443 \
  -v $(pwd)/cert.pem:/etc/nginx/cert.pem \
  -v $(pwd)/key.pem:/etc/nginx/key.pem \
  -v $(pwd)/record.txt:/usr/share/nginx/html/record.txt nginx
```

**Evidence — `lab_3_5.png`:**

![lab_3_5](lab_3_5.png)

**What happened:**
Docker could not find `nginx:latest` locally, so it pulled all image layers from Docker Hub:

```
Unable to find image 'nginx:latest' locally
latest: Pulling from library/nginx
[REDACTED — layer digests]
Digest: [REDACTED]
Status: Downloaded newer image for nginx:latest
[REDACTED — container ID]
```

The container started successfully. However, this run does **not yet** include the SSL config — that comes next.

---

### Step 11 — Write the NGINX SSL configuration

**Command:**
```bash
cat <<EOF > ssl.conf
server {
    listen 443 ssl;
    ssl_certificate /etc/nginx/cert.pem;
    ssl_certificate_key /etc/nginx/key.pem;
    location / {
        root /usr/share/nginx/html;
    }
}
EOF
```

**Evidence — `lab_3_6.png`:**

![lab_3_6](lab_3_6.png)

**What happened:**
The `ssl.conf` file was created. This config tells NGINX to:
- Listen on port 443 (standard HTTPS)
- Use `cert.pem` and `key.pem` for TLS
- Serve files from `/usr/share/nginx/html`

Running `ls` shows all files present:
```
cert.pem  key.pem  private.pem  public.pem
record.dec.txt  record.enc  record.rsa
record.rsa.txt  record.sig  record.txt  ssl.conf
```

---

### Step 12 — Restart NGINX with SSL config mounted

**Command:**
```bash
sudo docker run --rm -d --name tls -p 8443:443 \
  -v $(pwd)/cert.pem:/etc/nginx/cert.pem \
  -v $(pwd)/key.pem:/etc/nginx/key.pem \
  -v $(pwd)/record.txt:/usr/share/nginx/html/record.txt \
  -v $(pwd)/ssl.conf:/etc/nginx/conf.d/ssl.conf \
  nginx
```

**Evidence — `lab_3_6.png`:**

![lab_3_6](lab_3_6.png)

**What happened:**
NGINX started with all four volume mounts. The container ID was returned:
```
[REDACTED — container ID]
```

NGINX is now serving HTTPS on `https://localhost:8443` with TLS fully configured.

---

### Step 13 — Access the HTTPS endpoint and verify

**Command:**
```bash
curl -k https://localhost:8443/record.txt
```

**Evidence — `lab_3_7.png`:**

![lab_3_7](lab_3_7.png)

**What happened:**
The `-k` flag tells curl to accept the self-signed certificate (bypassing the trust warning). The response returned:

```
[REDACTED — patient record content]
```

> ✅ The file was retrieved successfully over an **encrypted HTTPS connection**. Any network traffic between curl and NGINX is TLS-encrypted — a network sniffer would only see ciphertext, not the plaintext content.

---

# SESSION B — Cloud Key Management with AWS KMS

---

## Task 5 — Create Customer Master Keys (CMKs)

### 💡 What is AWS KMS?

**AWS Key Management Service (KMS)** is a managed cloud service for creating, storing, and controlling cryptographic keys. In this lab, **LocalStack** simulates KMS on your local machine — no real AWS account is required.

A **Customer Master Key (CMK)** is the top-level key in KMS. It never directly encrypts large data files — it is used to protect (wrap) smaller **data keys** instead.

```
CMK  →  wraps  →  Data Key  →  encrypts  →  Your Data
```

---

### Step 14 — Set the LocalStack endpoint variable

**Command:**
```bash
EP='--endpoint-url=http://localhost:4566'
```

This variable is prepended to every `aws` command so requests go to LocalStack instead of real AWS.

**Evidence — `lab_3_8.png`:**

![lab_3_8](lab_3_8.png)

---

### Step 15 — Create a CMK for Tenant A

**Command:**
```bash
aws $EP kms create-key --description 'CCSE tenant-A master key'
```

**Evidence — `lab_3_8.png`:**

![lab_3_8](lab_3_8.png)

**Output (sensitive fields redacted):**
```json
{
    "KeyMetadata": {
        "AWSAccountId": "[REDACTED]",
        "KeyId":        "[REDACTED]",
        "Arn":          "[REDACTED]",
        "CreationDate": "2026-08-20T23:57:42.340739+08:00",
        "Enabled":       true,
        "Description":  "CCSE tenant-A master key",
        "KeyUsage":     "ENCRYPT_DECRYPT",
        "KeyState":     "Enabled",
        "Origin":       "AWS_KMS",
        "KeyManager":   "CUSTOMER",
        "KeySpec":      "SYMMETRIC_DEFAULT",
        "EncryptionAlgorithms": ["SYMMETRIC_DEFAULT"],
        "MultiRegion":  false
    }
}
```

**Key fields explained:**

| Field | Meaning |
|-------|---------|
| `KeyId` | Unique ID for this CMK — used in all future operations |
| `KeyState: Enabled` | Key is active and ready to use |
| `KeyUsage: ENCRYPT_DECRYPT` | Can encrypt and decrypt |
| `KeySpec: SYMMETRIC_DEFAULT` | AES-256-GCM symmetric key |
| `KeyManager: CUSTOMER` | You control this key (not AWS) |

**Store the Key ID in a shell variable:**
```bash
KEY_A=[REDACTED]
```

---

### Step 16 — Test-encrypt a value with the CMK

**Command:**
```bash
aws $EP kms encrypt \
  --key-id $KEY_A \
  --plaintext "$(echo -n 'hello' | base64)" \
  --query CiphertextBlob --output text
```

**Evidence — `lab_3_8.png`:**

![lab_3_8](lab_3_8.png)

**Output:**
```
[REDACTED — ciphertext blob]
```

**What happened:**
The word `hello` (base64-encoded as required by the KMS API) was encrypted using `KEY_A`. The returned ciphertext blob is opaque Base64 — it cannot be decoded without calling `kms:Decrypt` with the same key. This confirms the CMK is working.

---

## Task 6 — Envelope Encryption

### 💡 What is Envelope Encryption?

Instead of sending large files to KMS (which has size limits per API call), you use a **two-layer approach**:

```
┌──────────────────────────────────────────────────────────────┐
│  Layer 1: Generate a one-time Data Key via KMS               │
│           KMS returns:                                       │
│           • Plaintext Data Key  → use to encrypt your file  │
│           • Encrypted Data Key  → store permanently          │
│                                                              │
│  Layer 2: Encrypt your file with the plaintext Data Key     │
│           Then DELETE the plaintext key from disk            │
│                                                              │
│  Result stored on disk:                                      │
│   • record.env.enc  — encrypted file                        │
│   • datakey.enc     — KMS-wrapped data key                  │
└──────────────────────────────────────────────────────────────┘
```

To decrypt later: call KMS → unwrap `datakey.enc` → use the returned plaintext key to decrypt `record.env.enc`.

---

### Step 17 — Generate a Data Key from KMS

**Command:**
```bash
aws $EP kms generate-data-key \
  --key-id $KEY_A \
  --key-spec AES_256 \
  --query '[Plaintext,CiphertextBlob]' \
  --output text
```

**Evidence — `lab_3_9.png`:**

![lab_3_9](lab_3_9.png)

**Output — two values returned on one line:**
```
[REDACTED — plaintext data key (base64)]    [REDACTED — encrypted data key (base64)]
```

**Save each value to a file:**
```bash
cat > datakey.b64        # paste the plaintext key (first value)
[REDACTED]

cat > datakey.enc        # paste the encrypted key (second value)
[REDACTED]
```

**Evidence — `lab_3_9.png`:**

![lab_3_9](lab_3_9.png)

---

### Step 18 — Encrypt the record using the plaintext data key

**Commands:**
```bash
base64 -d datakey.b64 > datakey.bin

openssl enc -aes-256-cbc -pbkdf2 \
  -in record.txt \
  -out record.env.enc \
  -pass file:./datakey.bin
```

**Evidence — `lab_3_10.png`:**

![lab_3_10](lab_3_10.png)

**What happened:**
- `base64 -d` decoded the base64 data key into raw binary (`datakey.bin`).
- OpenSSL used `datakey.bin` as the encryption passphrase to encrypt `record.txt` into `record.env.enc`.
- The record is now protected by the data key, which is itself protected by the CMK.

---

### Step 19 — Delete the plaintext data key (critical security step)

**Command:**
```bash
rm datakey.bin datakey.b64
echo 'Only the KMS-wrapped data key (datakey.enc) remains.'
```

**Evidence — `lab_3_10.png`:**

![lab_3_10](lab_3_10.png)

**Output:**
```
Only the KMS-wrapped data key (datakey.enc) remains.
```

**What happened:**

| File | Fate | Reason |
|------|------|--------|
| `datakey.bin` | **Deleted** ✗ | Plaintext key must never stay on disk |
| `datakey.b64` | **Deleted** ✗ | Same key in base64 — equally dangerous |
| `datakey.enc` | **Kept** ✓ | Safe to store — encrypted by CMK |
| `record.env.enc` | **Kept** ✓ | Encrypted data file |

> ✅ Without calling KMS to unwrap `datakey.enc`, it is **impossible** to decrypt `record.env.enc`. This is the security guarantee of envelope encryption.

---

## Task 7 — Key Lifecycle Management

### 💡 Key Lifecycle States

```
Enabled  ──→  PendingDeletion  ──→  Deleted (irreversible)
   │                  ↑
   └──→  Disabled     │
              │       │
              └───────┘ (cancel-key-deletion returns to Disabled)
```

| State | Meaning |
|-------|---------|
| `Enabled` | Key is active — can encrypt and decrypt |
| `Disabled` | Key is inactive — all operations blocked, but recoverable |
| `PendingDeletion` | Scheduled for deletion — no operations allowed |
| `Deleted` | Permanently destroyed — all wrapped data is locked forever |

---

### Step 20 — Create a second CMK for Tenant B

**Command:**
```bash
aws $EP kms create-key --description 'CCSE tenant-B master key'
```

**Evidence — `lab_3_11.png`:**

![lab_3_11](lab_3_11.png)

**Output (sensitive fields redacted):**
```json
{
    "KeyMetadata": {
        "AWSAccountId": "[REDACTED]",
        "KeyId":        "[REDACTED]",
        "Arn":          "[REDACTED]",
        "CreationDate": "2026-08-21T01:34:16.648901+08:00",
        "Enabled":       true,
        "Description":  "CCSE tenant-B master key",
        "KeyUsage":     "ENCRYPT_DECRYPT",
        "KeyState":     "Enabled",
        "KeySpec":      "SYMMETRIC_DEFAULT"
    }
}
```

```bash
KEY_B=[REDACTED]
```

**Why create a second key?**
Each tenant gets a separate CMK. Revoking Tenant A's key has **zero impact** on Tenant B's data — complete cryptographic isolation between tenants.

---

### Step 21 — Schedule Tenant A's key for deletion (7-day window)

**Command:**
```bash
aws $EP kms schedule-key-deletion \
  --key-id $KEY_A \
  --pending-window-in-days 7
```

**Evidence — `lab_3_11.png`:**

![lab_3_11](lab_3_11.png)

**Output (sensitive fields redacted):**
```json
{
    "KeyId":               "[REDACTED]",
    "DeletionDate":        "2026-08-28T01:35:01.627224+08:00",
    "KeyState":            "PendingDeletion",
    "PendingWindowInDays": 7
}
```

**What happened:**
`KEY_A` moved to `PendingDeletion`. It will be permanently deleted after 7 days. During this window, **all cryptographic operations using this key are blocked**.

---

### Step 22 — Attempt operations on a PendingDeletion key (errors expected)

**Evidence — `lab_3_12.png`:**

![lab_3_12](lab_3_12.png)

**Attempt 1 — Disable the key:**
```bash
aws $EP kms disable-key --key-id $KEY_A
```
**Error:**
```
aws: [ERROR]: An error occurred (KMSInvalidStateException)
when calling the DisableKey operation:
[REDACTED KEY ARN] is pending deletion.
```

**Verify key state:**
```bash
aws $EP kms describe-key --key-id $KEY_A \
  --query 'KeyMetadata.KeyState' --output text
```
**Output:**
```
PendingDeletion
```

**Attempt 2 — Schedule deletion again:**
```bash
aws $EP kms schedule-key-deletion --key-id $KEY_A --pending-window-in-days 7
```
**Error:**
```
aws: [ERROR]: An error occurred (KMSInvalidStateException)
when calling the ScheduleKeyDeletion operation:
[REDACTED KEY ARN] is pending deletion.
```

**What happened:**
Both operations failed with `KMSInvalidStateException`. A key in `PendingDeletion` **cannot be used or modified** in any way. KMS enforces strict state machine rules to prevent accidental or malicious misuse during the deletion window.

---

### Step 23 — Cancel deletion, then disable the key

**Evidence — `lab_3_12.png`:**

![lab_3_12](lab_3_12.png)

**Cancel the deletion:**
```bash
aws $EP kms cancel-key-deletion --key-id $KEY_A
```
**Output:**
```json
{
    "KeyId": "[REDACTED]"
}
```

**Disable the key:**
```bash
aws $EP kms disable-key --key-id $KEY_A
```

**Verify the new state:**
```bash
aws $EP kms describe-key --key-id $KEY_A \
  --query 'KeyMetadata.KeyState' --output text
```
**Output:**
```
Disabled
```

**What happened:**
Deletion was cancelled. The key was then manually disabled. A `Disabled` key:
- **Blocks** all encrypt/decrypt operations
- **Can be re-enabled** by an authorized administrator
- Is **not permanently destroyed**

---

### Step 24 — Attempt decryption with a disabled key (error expected)

**Command:**
```bash
aws $EP kms decrypt --ciphertext-blob fileb://datakey.enc 2>&1 | head -3
```

**Evidence — `lab_3_12.png`:**

![lab_3_12](lab_3_12.png)

**Output:**
```
aws: [ERROR]: An error occurred (NotFoundException)
when calling the Decrypt operation:
Invalid keyId '[REDACTED]'
```

**What happened:**
With `KEY_A` disabled, KMS refused to decrypt `datakey.enc`. This means `record.env.enc` is now completely inaccessible — the data is effectively **locked** until the key is re-enabled by an authorized administrator.

> ✅ This is a powerful, instant access revocation tool in cloud security operations.

---

## Task 8 — Data Integrity with SHA-256

### 💡 What is SHA-256?

SHA-256 is a **cryptographic hash function** — it takes any input and produces a fixed 64-character hexadecimal fingerprint. Key properties:

- **Deterministic** — the same input always produces the same hash
- **One-way** — you cannot reverse a hash back to the original data
- **Avalanche effect** — even one character change produces a completely different hash
- **Collision resistant** — it is computationally infeasible to find two different inputs with the same hash

---

### Step 25 — Hash the original file and a tampered copy

**Commands:**
```bash
sha256sum record.txt
cp record.txt tampered.txt
echo 'x' >> tampered.txt
sha256sum record.txt tampered.txt
```

**Evidence — `lab_3_13.png`:**

![lab_3_13](lab_3_13.png)

**Output (hash values redacted):**
```
[REDACTED]  record.txt
[REDACTED]  record.txt      ← same file, same hash ✓
[REDACTED]  tampered.txt    ← 1 character added, completely different hash ✗
```

**What happened:**
`tampered.txt` is identical to `record.txt` except for one appended character (`x`). Despite this tiny change, the SHA-256 hash is **completely different** — the avalanche effect in action. An attacker cannot modify a file without the hash changing and the tampering being detected immediately.

---

## Task 9 — Hash Chaining (Audit Log Integrity)

### 💡 What is Hash Chaining?

Hash chaining links each log entry to all previous entries by **including the previous hash in each new computation**:

```
H1 = SHA256("0"  + "login ok")
H2 = SHA256(H1   + "file read")
H3 = SHA256(H2   + "export data")
```

If an attacker modifies any past entry, its hash changes — which cascades to change **all subsequent hashes**. This makes tampering immediately detectable. This is the same principle used in **blockchain technology**.

---

### Step 26 — Build a 3-entry tamper-evident hash chain

**Command:**
```bash
PREV=0
for line in 'login ok' 'file read' 'export data'; do \
  PREV=$(echo -n "$PREV$line" | sha256sum | cut -d' ' -f1); \
  echo "$line | $PREV"; \
done
```

**Evidence — `lab_3_14.png`:**

![lab_3_14](lab_3_14.png)

**Output (hash values redacted):**
```
login ok    | [REDACTED]
file read   | [REDACTED]
export data | [REDACTED]
```

**How the chain is constructed:**

| Step | Computation | Stored As |
|------|-------------|-----------|
| 1 | `SHA256("0" + "login ok")` | `[HASH-1]` |
| 2 | `SHA256("[HASH-1]" + "file read")` | `[HASH-2]` |
| 3 | `SHA256("[HASH-2]" + "export data")` | `[HASH-3]` |

**What happened:**
Each log entry's hash depends on all entries before it. Deleting or altering `login ok` changes `[HASH-1]`, which changes `[HASH-2]`, which changes `[HASH-3]` — the entire chain becomes invalid. Integrity can be verified at any time by recomputing the chain and comparing the final hash.

---

# Assessment Questions and Answers

---

**Q1. What is the difference between symmetric and asymmetric encryption? When would you use each in a cloud context?**

**Answer:**

**Symmetric encryption** (e.g., AES-256-CBC used in Task 1) uses the **same key** for both encrypting and decrypting. It is computationally fast and ideal for encrypting **large amounts of data at rest** — files, database records, storage volumes. In a cloud context, it is used for S3 server-side encryption (SSE), EBS volume encryption, and RDS at-rest encryption.

**Asymmetric encryption** (e.g., RSA-2048 in Task 2) uses a **public/private key pair** — the public key encrypts, the private key decrypts. It is slower but solves the key distribution problem: you can share the public key freely without compromising security. In a cloud context, it is used in **TLS handshakes** (to securely agree on a session key), **SSH authentication** to cloud VMs, and **digital signatures** for code signing and certificate authorities.

In practice, both are combined in a **hybrid scheme** — asymmetric encryption securely exchanges a symmetric key, and the symmetric key then encrypts the actual data. This is exactly how HTTPS/TLS works — and it gives you the security of asymmetric with the speed of symmetric.

---

**Q2. What is the purpose of the `-pbkdf2` and `-salt` flags in OpenSSL AES encryption?**

**Answer:**

- **`-salt`** adds a random value (the salt) to the password input before key derivation. This means the same password produces a **different encryption key every time**, preventing **rainbow table attacks** (pre-computed tables mapping passwords to keys). It also ensures two users encrypting with the same password produce different ciphertexts.

- **`-pbkdf2`** (Password-Based Key Derivation Function 2) derives the encryption key from the password through **thousands of iterations** of HMAC-SHA256. This makes brute-force or dictionary attacks dramatically slower — an attacker must pay the cost of thousands of iterations for every password guess. For a legitimate user who knows the password, this one-time cost is negligible.

Together, these two flags harden password-based file encryption against offline cracking attacks.

---

**Q3. What is envelope encryption and why is it used in cloud key management?**

**Answer:**

Envelope encryption is a **two-layer encryption scheme**:

1. Your data is encrypted with a short-lived **Data Encryption Key (DEK)** — a randomly generated AES key.
2. The DEK itself is then encrypted (wrapped) by the **Customer Master Key (CMK)** inside KMS.

This approach is used in cloud key management for these reasons:

| Reason | Explanation |
|--------|-------------|
| **Performance** | KMS can only process small payloads per API call. The DEK is tiny; the large data file is handled locally by OpenSSL. |
| **Security** | The plaintext DEK only lives in memory briefly, then is deleted. It is never persisted to disk. |
| **Instant revocation** | Disabling the CMK immediately makes all wrapped DEKs — and therefore all encrypted data — inaccessible, without re-encrypting any files. |
| **Multi-tenancy** | Each tenant has a separate CMK. Revoking one tenant's key has no effect on others. |

In this lab, `datakey.enc` holds the wrapped DEK next to the encrypted data. Decryption requires a KMS API call — only authorized callers with the correct permissions can proceed.

---

**Q4. What happened when you attempted to use KEY_A after it was disabled? What does this demonstrate?**

**Answer:**

After `KEY_A` was disabled, attempting to decrypt `datakey.enc` via KMS returned:

```
aws: [ERROR]: An error occurred (NotFoundException)
when calling the Decrypt operation: Invalid keyId '[REDACTED]'
```

This demonstrates that **disabling a CMK instantly revokes all cryptographic access** to any data whose DEK was wrapped by that key — without deleting the data itself. Key insights:

- **Instant lockout** — decryption fails the moment the key is disabled. No grace period.
- **Reversible** — unlike deletion, disabling can be undone by re-enabling the key.
- **Separation of duties** — the key administrator and the data owner are separate roles.
- **Incident response** — in a breach, a compromised tenant's key can be disabled in seconds, preventing decryption of any stolen ciphertexts.

Additionally, when the key was in `PendingDeletion`, even administrative operations like `disable-key` failed with `KMSInvalidStateException` — showing that KMS enforces a **strict state machine** where only permitted transitions are allowed.

---

**Q5. How does SHA-256 hash chaining protect audit log integrity?**

**Answer:**

Hash chaining creates a **cryptographically linked chain** of log entries. Each hash is computed from the current log entry **concatenated with the previous hash**:

```
H1 = SHA256("0"  + "login ok")
H2 = SHA256(H1   + "file read")
H3 = SHA256(H2   + "export data")
```

If an attacker attempts to modify, delete, or reorder any past log entry, the hash for that entry changes. Because every subsequent hash depends on all previous hashes, **all hashes after the tampered entry become invalid**. The integrity of the full log can be verified at any time by recomputing the chain from scratch and comparing the final hash against a trusted stored reference.

Real-world applications:
- **Blockchain** — each block stores the previous block's hash, making the ledger tamper-evident
- **Certificate Transparency (CT) logs** — public, hash-chained records of all TLS certificates ever issued
- **Compliance audit trails** — PCI-DSS, HIPAA, and ISO 27001 require tamper-evident audit logs

In cloud security, this prevents **insider threats** from silently deleting or modifying log entries to cover their tracks.

---

**Q6. Why was `sudo` required for Docker commands in this lab?**

**Answer:**

The Docker daemon runs as `root` and exposes its management interface through a Unix socket at `/var/run/docker.sock`. By default, this socket is accessible only to the `root` user and members of the `docker` group. The user `asricloud` is not in the `docker` group, so without `sudo` the command fails:

```
permission denied while trying to connect to the docker API
at unix:///var/run/docker.sock
```

Using `sudo` grants temporary root-level access for that specific command.

The preferred production fix is:
```bash
sudo usermod -aG docker $USER
# then log out and back in
```

This allows Docker commands without `sudo`. However, membership in the `docker` group grants **root-equivalent access to the host system** (you can mount any directory into a container), so this should only be granted to trusted administrators.

---

# Cleanup and Teardown

After completing the lab, all resources must be removed to stop running containers and delete all generated files from disk.

---

**Evidence — `lab_3_15.png`:**

![lab_3_15](lab_3_15.png)

---

### Cleanup Step 1 — Stop the TLS NGINX container

**Command:**
```bash
docker stop tls 2>/dev/null
```

The `2>/dev/null` suppresses any error if the container was already stopped. This terminates the NGINX HTTPS server running on port 8443.

---

### Cleanup Step 2 — Remove all generated lab files

**Command:**
```bash
rm -f record.* private.pem public.pem key.pem cert.pem datakey.* tampered.txt
```

**Files removed:**

| File | What it was |
|------|-------------|
| `record.txt` | Original plaintext patient record |
| `record.enc` | AES-256-CBC encrypted record |
| `record.dec.txt` | Decrypted verification copy |
| `record.rsa` | RSA-encrypted record |
| `record.rsa.txt` | RSA-decrypted record |
| `record.sig` | Digital signature file |
| `record.env.enc` | Envelope-encrypted record |
| `private.pem` | RSA private key |
| `public.pem` | RSA public key |
| `key.pem` | TLS server private key |
| `cert.pem` | TLS self-signed certificate |
| `datakey.enc` | KMS-wrapped data key |
| `tampered.txt` | Tampered copy used for hash test |

---

### Cleanup Step 3 — Stop and remove the LocalStack container

**Command:**
```bash
sudo docker stop localstack && sudo docker rm localstack
```

**What happened:**
The LocalStack container (simulating AWS KMS) was stopped and its container filesystem deleted. All KMS keys created during the lab (`KEY_A`, `KEY_B`) are permanently gone.

---

**Final state:** The `~/Documents/cloudLab3` directory is now empty — no sensitive files, keys, certificates, or encrypted data remain on disk.

---

# Conclusion

This lab provided a comprehensive, hands-on exploration of **encryption and key management** — two foundational pillars of cloud security. The following key outcomes were achieved across both sessions.

---

## What Was Accomplished

### Session A — File-Level Encryption

**Task 1 — Symmetric Encryption (AES-256-CBC)**
Successfully demonstrated file-at-rest protection using OpenSSL. The combination of `-pbkdf2` and `-salt` showed how to harden password-based encryption against brute-force and rainbow-table attacks. The `diff` verification confirmed perfect round-trip fidelity.

**Task 2 — Asymmetric Encryption (RSA-2048)**
Generated a fully functional RSA key pair and demonstrated that data encrypted with the public key can only be decrypted by the corresponding private key — solving the key distribution problem inherent in symmetric-only approaches.

**Task 3 — Digital Signatures**
Used OpenSSL to sign a file with a private key and verify the signature with the public key. The `Verified OK` output confirmed both **authenticity** (the signature came from the private key holder) and **integrity** (the file was not modified after signing).

**Task 4 — TLS Web Server**
Set up a Docker-based NGINX HTTPS server with a self-signed certificate on port 8443. Verified that `curl -k` retrieved the file content over an encrypted connection. This demonstrated the full TLS pipeline — certificate generation, server configuration, and encrypted data-in-transit delivery.

---

### Session B — Cloud Key Management

**Task 5 — Customer Master Key Creation**
Used LocalStack to simulate AWS KMS and successfully created CMKs for two separate tenants (Tenant A and Tenant B). The JSON metadata confirmed each key's properties and its `Enabled` state — ready for cryptographic operations.

**Task 6 — Envelope Encryption**
Implemented the full envelope encryption workflow: generated a data key from KMS, used it to encrypt a file with OpenSSL, then deleted the plaintext key from disk. Only the KMS-wrapped `datakey.enc` remained, proving that access to the underlying data is controlled entirely through KMS — not raw key material on disk.

**Task 7 — Key Lifecycle Management**
Explored all major KMS key state transitions: `Enabled` → `PendingDeletion` → (cancelled) → `Disabled`. Demonstrated that:
- A key in `PendingDeletion` blocks **all** operations (including administrative ones).
- A `Disabled` key immediately blocks decryption, effectively locking all data encrypted under it.
- Cancelling deletion and re-enabling is possible — giving administrators a safety net before permanent destruction.

**Task 8 — SHA-256 Hash Integrity**
Proved that SHA-256's avalanche effect makes any file modification — even a single character — immediately detectable through hash comparison.

**Task 9 — Hash Chaining**
Built a tamper-evident audit log using hash chaining. Demonstrated that modifying any past entry invalidates all subsequent entries in the chain — making silent log tampering computationally infeasible.

---

## Key Lessons Learned

| Lesson | Implication |
|--------|-------------|
| Symmetric encryption is fast but requires secure key distribution | Use KMS or a key vault to protect symmetric keys |
| Asymmetric encryption solves key distribution but is slower | Combine both in hybrid schemes (as in TLS) |
| Digital signatures provide non-repudiation and integrity | Critical for code signing, software distribution, and audit trails |
| TLS protects data in transit — not just at rest | Always enforce HTTPS; HTTP exposes all traffic to interception |
| Envelope encryption decouples data protection from key management | Disabling a CMK instantly revokes access to all wrapped data |
| Key lifecycle management is a first-class security control | `Disabled` and `PendingDeletion` states are powerful incident response tools |
| SHA-256 hashing detects any tampering | Always hash files before transfer or storage for integrity checks |
| Hash chaining prevents silent log manipulation | Essential for compliant audit trails in regulated industries |

---

## Real-World Relevance

The techniques practised in this lab directly map to production cloud security controls:

- **AWS S3 SSE-KMS** — uses envelope encryption exactly as shown in Task 6
- **AWS RDS / EBS encryption** — AES-256 at rest, keys managed through KMS
- **HTTPS everywhere** — the TLS setup in Task 4 mirrors any production web application
- **CloudTrail log integrity** — AWS uses hash chaining (similar to Task 9) to detect tampering in audit logs
- **IAM + KMS key policies** — the access revocation demonstrated in Task 7 is the foundation of incident response in multi-tenant SaaS platforms

Mastering these controls builds the practical foundation needed to design, implement, and audit encryption strategies in real cloud environments.

---

> **Report prepared by:** `asricloud`
> **Evidence:** Screenshots `lab_3_1.png` through `lab_3_15.png`
> **Guide:** `IKB42603_Lab3_Encryption_and_Key_Management.pdf`
>
> *All sensitive values — including Key IDs, ARNs, AWS Account IDs, ciphertext blobs, cryptographic key material, Docker container IDs, Docker image digests, SHA-256 hash outputs, and patient record content — have been redacted in this document.*
