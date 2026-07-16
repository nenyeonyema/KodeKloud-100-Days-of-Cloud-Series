# Azure Task 40 — Data Encryption with Azure Key Vault and RSA Key

## Overview

This task demonstrates using Azure Key Vault to secure sensitive data through RSA encryption and decryption. A Key Vault is created with a 4096-bit RSA key, and a sensitive file is encrypted using the RSA-OAEP algorithm via the Azure CLI. The encrypted output is then decrypted and verified to match the original file, validating the end-to-end Key Vault encryption workflow.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      azure-client host                      │
│                                                             │
│  /root/SensitiveData.txt  ──► base64 encode                 │
│         │                          │                        │
│         │                          ▼                        │
│         │              ┌───────────────────────┐           │
│         │              │   az keyvault key      │           │
│         │              │   encrypt (RSA-OAEP)   │           │
│         │              └──────────┬────────────┘           │
│         │                         │                        │
│         │                         ▼                        │
│         │              /root/EncryptedData.bin              │
│         │                         │                        │
│         │              ┌───────────────────────┐           │
│         │              │   az keyvault key      │           │
│         │              │   decrypt (RSA-OAEP)   │           │
│         │              └──────────┬────────────┘           │
│         │                         │                        │
│         │                         ▼ base64 decode          │
│         │              /root/DecryptedData.txt              │
│         │                         │                        │
│         └──────── diff ───────────┘                        │
│                   Files match ✅                            │
└─────────────────────────────────────────────────────────────┘
                          │ encrypt/decrypt API calls
                          ▼
          ┌───────────────────────────────────┐
          │         devops-17164              │
          │         Azure Key Vault           │
          │         East US | Standard        │
          │                                   │
          │   ┌───────────────────────────┐   │
          │   │  devops-key               │   │
          │   │  RSA 4096-bit             │   │
          │   │  Algorithm: RSA-OAEP      │   │
          │   └───────────────────────────┘   │
          └───────────────────────────────────┘
```

---

## Prerequisites

- Azure CLI installed and authenticated
- Resource group: `kml_rg_main-53e5d64f7cc94757`
- `SensitiveData.txt` present at `/root/SensitiveData.txt` on the lab host
- Region: East US

---

## Step 1 — Login

```bash
az login -u kk_lab_user_main-53e5d64f7cc94757@azurefreekmlprod.onmicrosoft.com -p "2Rw@na+="
```

---

## Step 2 — Get Resource Group and Service Principal ID

```bash
az group list --query "[0].name" -o tsv
az account show --query user.name -o tsv
```

Output:
```
kml_rg_main-53e5d64f7cc94757
6d7c53ff-2b0d-4c7f-b5d8-1a23e64d8746
```

The `user.name` field returns the Service Principal object ID in this lab environment — needed for the access policy.

---

## Step 3 — Create the Key Vault

```bash
az keyvault create \
  --name devops-17164 \
  --resource-group kml_rg_main-53e5d64f7cc94757 \
  --location eastus \
  --sku standard \
  --retention-days 7 \
  --enable-rbac-authorization false
```

**Flags explained:**
- `--sku standard` — Standard tier supports software-protected keys (HSM tier costs extra)
- `--retention-days 7` — Soft-deleted vaults are retained for 7 days before permanent deletion
- `--enable-rbac-authorization false` — Uses Vault access policies instead of Azure RBAC for permissions
- `--enable-soft-delete` is omitted — it is enabled by default in modern Azure CLI and the flag is deprecated

---

## Step 4 — Set Access Policy for the Service Principal

```bash
az keyvault set-policy \
  --name devops-17164 \
  --object-id 6d7c53ff-2b0d-4c7f-b5d8-1a23e64d8746 \
  --key-permissions get list encrypt decrypt
```

This grants the lab Service Principal the minimum permissions needed: Get and List to read the key, Encrypt and Decrypt to perform cryptographic operations.

---

## Step 5 — Create the RSA 4096 Key

```bash
az keyvault key create \
  --vault-name devops-17164 \
  --name devops-key \
  --kty RSA \
  --size 4096
```

**Why 4096-bit RSA?**
4096-bit RSA provides stronger security than the default 2048-bit, making it more resistant to brute-force attacks. It is recommended for encrypting sensitive data with a long security lifetime.

---

## Step 6 — Encrypt the Sensitive File

```bash
# Base64 encode the plaintext file
BASE64_DATA=$(base64 /root/SensitiveData.txt)

# Get the key version ID
KEY_ID=$(az keyvault key show \
  --vault-name devops-17164 \
  --name devops-key \
  --query "key.kid" -o tsv)

# Encrypt using RSA-OAEP and save binary output
az keyvault key encrypt \
  --vault-name devops-17164 \
  --name devops-key \
  --algorithm RSA-OAEP \
  --value "$BASE64_DATA" \
  --data-type base64 \
  --query result -o tsv | base64 -d > /root/EncryptedData.bin
```

**Why Base64 encode before encrypting?**
The Key Vault encrypt API expects the plaintext value as a Base64-encoded string. The raw file content must be Base64 encoded first so it can be passed safely as a JSON string value in the API call.

**Why RSA-OAEP?**
RSA-OAEP (Optimal Asymmetric Encryption Padding) is the recommended padding scheme for RSA encryption. It is more secure than the older PKCS#1 v1.5 padding and is the standard for modern RSA encryption.

---

## Step 7 — Decrypt and Verify

```bash
# Base64 encode the encrypted binary for the API
ENCRYPTED_B64=$(base64 /root/EncryptedData.bin)

# Decrypt
az keyvault key decrypt \
  --vault-name devops-17164 \
  --name devops-key \
  --algorithm RSA-OAEP \
  --value "$ENCRYPTED_B64" \
  --data-type base64 \
  --query result -o tsv | base64 -d > /root/DecryptedData.txt

# Verify files match
diff /root/SensitiveData.txt /root/DecryptedData.txt && echo "Files match ✅" || echo "Files differ ❌"
[O```

Output:
```
Files match ✅
```

Decrypted content:
```
This is a sensitive file.
```

---

## Key Concepts

**Vault Access Policy vs Azure RBAC**
Azure Key Vault supports two permission models:

| Model | How it works |
|---|---|
| Vault access policy | Permissions set directly on the vault per identity |
| Azure RBAC | Permissions managed via Azure role assignments |

This task uses Vault access policies (`--enable-rbac-authorization false`) as required by the task specification.

**Encrypt/Decrypt flow summary**

```
SensitiveData.txt
      │
      ▼
 base64 encode          (required by Key Vault API)
      │
      ▼
 az keyvault key encrypt --algorithm RSA-OAEP
      │
      ▼
 API returns base64-encoded ciphertext
      │
      ▼
 base64 -d              (convert back to binary)
      │
      ▼
 EncryptedData.bin
```

**Why `base64 -d` after encryption and before decryption?**
The Key Vault API returns and accepts Base64-encoded values. After encryption, decoding with `base64 -d` converts the ciphertext to its raw binary form for storage. Before decryption, the binary file is re-encoded with `base64` so it can be passed back to the API.

---

## Files Summary
[I
| File | Location | Description |
|---|---|---|
| SensitiveData.txt | /root/ | Original plaintext file |
| EncryptedData.bin | /root/ | RSA-OAEP encrypted binary |
| DecryptedData.txt | /root/ | Decrypted output (matches original) |

---

## Outcome

| Task | Result |
|---|---|
| Key Vault devops-17164 created (East US, Standard, 7-day retention) | ✅ |
| Vault access policy model configured | ✅ |
| Access policy set (Get, List, Encrypt, Decrypt) | ✅ |
| RSA 4096 key devops-key created | ✅ |
| SensitiveData.txt encrypted → EncryptedData.bin (RSA-OAEP) | ✅ |
| EncryptedData.bin decrypted → DecryptedData.txt | ✅ |
| Diff check → Files match | ✅ |
