# AWS Task 41 — KMS Encryption: Securing Sensitive Data with AWS Key Management Service

## Overview

This project documents the creation of a symmetric AWS KMS key to encrypt and decrypt a sensitive file. The team needed to improve data security by ensuring sensitive files are never stored or transmitted in plaintext. A KMS key was created, used to encrypt a file into binary ciphertext, and then used to decrypt it back — with the decrypted output verified to match the original file exactly.

---

## Architecture

```
SensitiveData.txt (plaintext)
    │
    ▼
┌──────────────────────────────────┐
│         AWS KMS Key              │
│   alias/datacenter-KMS-Key       │
│   Type: Symmetric (AES-256-GCM)  │
│   Usage: ENCRYPT_DECRYPT         │
└────────────┬─────────────────────┘
             │
             ▼ Encrypt API
    EncryptedData.bin (binary ciphertext)
             │
             ▼ Decrypt API
    DecryptedData.txt (plaintext)
             │
             ▼
    diff with SensitiveData.txt → ✅ Match
```

---

## Prerequisites

- AWS CLI configured with appropriate credentials
- IAM permissions for KMS operations:
  - `kms:CreateKey`
  - `kms:CreateAlias`
  - `kms:Encrypt`
  - `kms:Decrypt`
- Python 3 installed (for base64 encode/decode operations)
- A plaintext file at `/root/SensitiveData.txt`
- Region: `us-east-1`

---

## What Was Built

A symmetric KMS key named `datacenter-KMS-Key` with:
- **Key type** — Symmetric (same key used for both encryption and decryption)
- **Key spec** — `SYMMETRIC_DEFAULT` (AES-256-GCM)
- **Key alias** — `alias/datacenter-KMS-Key`
- Encrypted output saved as raw binary to `/root/EncryptedData.bin`
- Decrypted output verified to match the original `SensitiveData.txt` exactly

---

## Encryption & Decryption Flow

```
┌─────────────────────────────────────────────────────────┐
│                   KMS Encrypt Flow                       │
│                                                          │
│  fileb:///root/SensitiveData.txt                         │
│          │                                               │
│          ▼                                               │
│   kms encrypt API  →  CiphertextBlob (base64 string)    │
│          │                                               │
│          ▼                                               │
│   python3 base64.b64decode()  →  raw binary bytes       │
│          │                                               │
│          ▼                                               │
│   /root/EncryptedData.bin  (saved to disk)               │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                   KMS Decrypt Flow                       │
│                                                          │
│  fileb:///root/EncryptedData.bin                         │
│          │                                               │
│          ▼                                               │
│   kms decrypt API  →  Plaintext (base64 string)         │
│          │                                               │
│          ▼                                               │
│   python3 base64.b64decode()  →  original plaintext     │
│          │                                               │
│          ▼                                               │
│   /tmp/DecryptedData.txt  (verified against original)    │
└─────────────────────────────────────────────────────────┘
```

---

## Step-by-Step Implementation

### 1. Verify AWS Credentials

```bash
aws sts get-caller-identity
```

### 2. Create a Symmetric KMS Key

```bash
KEY_ID=$(aws kms create-key \
  --description "datacenter-KMS-Key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --query 'KeyMetadata.KeyId' \
  --output text)

echo "Created KMS Key ID: $KEY_ID"
```

### 3. Create a Key Alias

```bash
aws kms create-alias \
  --alias-name alias/datacenter-KMS-Key \
  --target-key-id $KEY_ID

echo "✅ Alias created: alias/datacenter-KMS-Key"
```

### 4. Encrypt the Sensitive File

```bash
aws kms encrypt \
  --key-id alias/datacenter-KMS-Key \
  --plaintext fileb:///root/SensitiveData.txt \
  --output json > /tmp/encrypt_output.json

cat /tmp/encrypt_output.json | python3 -c "
import json, sys, base64
data = json.load(sys.stdin)
ciphertext = base64.b64decode(data['CiphertextBlob'])
with open('/root/EncryptedData.bin', 'wb') as f:
    f.write(ciphertext)
print('Written', len(ciphertext), 'bytes to /root/EncryptedData.bin')
"
```

### 5. Decrypt the Encrypted File

```bash
aws kms decrypt \
  --ciphertext-blob fileb:///root/EncryptedData.bin \
  --output json > /tmp/decrypt_output.json

cat /tmp/decrypt_output.json | python3 -c "
import json, sys, base64
data = json.load(sys.stdin)
plaintext = base64.b64decode(data['Plaintext'])
with open('/tmp/DecryptedData.txt', 'wb') as f:
    f.write(plaintext)
print(plaintext.decode())
"
```

### 6. Verify Decrypted Output Matches Original

```bash
echo "=== Original ==="
cat /root/SensitiveData.txt

echo "=== Decrypted ==="
cat /tmp/DecryptedData.txt

if diff /root/SensitiveData.txt /tmp/DecryptedData.txt > /dev/null 2>&1; then
  echo "✅ SUCCESS: Decrypted data matches the original file!"
else
  echo "❌ MISMATCH: Files differ."
fi
```

---

## Verification

**Expected output after diff check:**
```
=== Original ===
This is a sensitive file.

=== Decrypted ===
This is a sensitive file.

✅ SUCCESS: Decrypted data matches the original file!
```

**Confirm KMS key and alias exist:**
```bash
aws kms describe-key --key-id alias/datacenter-KMS-Key \
  --query 'KeyMetadata.{KeyId:KeyId,Description:Description,KeyState:KeyState,KeyUsage:KeyUsage}' \
  --output table

aws kms list-aliases \
  --query "Aliases[?AliasName=='alias/datacenter-KMS-Key']" \
  --output table
```

Both key and alias confirmed ✅

---

## Console Verification Steps

For those who prefer the AWS Console:

1. **KMS → Customer managed keys** — confirm `datacenter-KMS-Key` key exists
2. **Check Key status** — should be `Enabled`
3. **Check Key spec** — should be `SYMMETRIC_DEFAULT`
4. **Check Key usage** — should be `ENCRYPT_DECRYPT`
5. **KMS → Customer managed keys → Aliases tab** — confirm `alias/datacenter-KMS-Key` is listed

---

## Troubleshooting Checklist

| Check | Command | Expected Result |
|---|---|---|
| KMS key exists | `describe-key` | `KeyState: Enabled` |
| Alias created | `list-aliases` | `alias/datacenter-KMS-Key` present |
| Encrypted file exists | `ls -lh /root/EncryptedData.bin` | Non-zero binary file |
| Decryption succeeds | `kms decrypt` | Returns base64 Plaintext |
| Files match | `diff` original vs decrypted | No output — files identical |

---

## Common Mistakes

| Mistake | Consequence |
|---|---|
| Using `file://` instead of `fileb://` for binary input | KMS receives corrupted data — `InvalidCiphertextException` on decrypt |
| Piping through `base64 --decode` in shell instead of Python | Shell pipeline corrupts bytes — decryption fails |
| Saving CiphertextBlob as-is without decoding from base64 | File stored as base64 string — `fileb://` on decrypt fails |
| Not decoding the `Plaintext` field from decrypt response | Output is base64 text not original plaintext — diff fails |
| Forgetting to create alias after creating key | Validation script cannot find key by alias name |

---

## Key Lesson

The AWS CLI returns all KMS binary data (both `CiphertextBlob` and `Plaintext`) as **base64-encoded strings** in its JSON response. This is not the raw data — it must be decoded before use.

Always follow this pattern when handling KMS binary data:

```
Encrypt  → parse JSON → base64.b64decode(CiphertextBlob) → write raw bytes to .bin file
Decrypt  → parse JSON → base64.b64decode(Plaintext)      → write raw bytes to output file
```

Use Python for base64 operations — never rely on shell piping with `base64 --decode`, which can silently corrupt binary data in certain environments and cause `InvalidCiphertextException` errors that are difficult to debug.

---

## Technologies Used

- **AWS KMS** — Fully managed encryption key service
- **Symmetric Encryption** — AES-256-GCM via `SYMMETRIC_DEFAULT` key spec
- **AWS CLI** — Key creation, encryption and decryption operations
- **Python 3** — Reliable base64 encoding and decoding of binary data
- **fileb://** — Binary-safe file input flag for AWS CLI
