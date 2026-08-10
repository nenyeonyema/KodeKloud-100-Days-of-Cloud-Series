# AWS Task 4 — Enable S3 Bucket Versioning

## 📌 Task Overview

Data protection and recovery are important parts of cloud infrastructure. If an object is accidentally deleted or overwritten, having previous versions available makes recovery much easier. For this task, S3 bucket versioning was enabled on an existing Amazon S3 bucket using the AWS CLI.

| Requirement | Value |
| :--- | :--- |
| **S3 Bucket** | `devops-s3-15416` |
| **Versioning** | Enabled |
| **AWS Region** | `us-east-1` |
| **Method** | AWS CLI |

---

## 🎯 Objective

Enable versioning on the existing S3 bucket: `devops-s3-15416`.

---

## 🛠️ Prerequisites

Make sure the AWS CLI is installed:
```bash
aws --version
```

Retrieve the temporary lab credentials:
```bash
showcreds
```

Verify the AWS identity:
```bash
aws sts get-caller-identity
```

Set and verify the required region:
```bash
export AWS_DEFAULT_REGION=us-east-1
aws configure get region
```

**Expected output:**
```text
us-east-1
```

---

## 1. Verify the S3 Bucket Exists

Before modifying the bucket, confirm that it exists:
```bash
aws s3api head-bucket \
  --bucket devops-s3-15416
```

*(No output means the bucket exists and is accessible).*

You can also list the bucket:
```bash
aws s3 ls s3://devops-s3-15416
```

---

## 2. Check Current Versioning Status

Before enabling versioning, check its current state:
```bash
aws s3api get-bucket-versioning \
  --bucket devops-s3-15416
```

If versioning has not been enabled, the response may be empty (`{}`) or show:
```json
{
    "Status": "Suspended"
}
```

---

## 3. Enable Versioning

Enable versioning on the S3 bucket:
```bash
aws s3api put-bucket-versioning \
  --bucket devops-s3-15416 \
  --versioning-configuration Status=Enabled
```

*(No output indicates that the request was successfully submitted).*

---

## 4. Verify Versioning Is Enabled

Confirm that versioning is now active:
```bash
aws s3api get-bucket-versioning \
  --bucket devops-s3-15416
```

**Expected output:**
```json
{
    "Status": "Enabled"
}
```

---

## 5. Optional: Test Versioning

To observe versioning in practice, create a test object:
```bash
echo "Version 1" > test.txt
aws s3 cp test.txt s3://devops-s3-15416/
```

Modify and upload the file again:
```bash
echo "Version 2" > test.txt
aws s3 cp test.txt s3://devops-s3-15416/
```

Because versioning is enabled, S3 retains both versions. List all object versions:
```bash
aws s3api list-object-versions \
  --bucket devops-s3-15416
```

### 🧹 Optional Cleanup
If the test object was only created for demonstration purposes, remove local files:
```bash
rm test.txt
```

> **Note:** Deleting a versioned S3 object creates a delete marker while older versions remain available for recovery.

---

## ✅ Final Result

Versioning was successfully enabled on the existing S3 bucket.

| Configuration | Result |
| :--- | :--- |
| **Bucket** | `devops-s3-15416` |
| **Versioning** | `Enabled` |
| **Region** | `us-east-1` |
| **Method** | AWS CLI |

### Verification Check
```bash
aws s3api get-bucket-versioning \
  --bucket devops-s3-15416
```

**Result:**
```json
{
    "Status": "Enabled"
}
```

---

## 🧠 What I Learned

S3 versioning provides an additional layer of protection against accidental deletion, overwriting, or modification of objects:

```text
Object: test.txt
  │
  ├── Version 1
  ├── Version 2
  └── Version 3
```

Without versioning, uploading a new object with the same key overwrites the previous data permanently. With versioning enabled, each version can be retained and recovered independently.

---

## 💰 Cost Considerations

Versioning improves data protection, but every retained version consumes storage space and incurs storage costs:

```text
File v1 → 100 MB
File v2 → 100 MB
File v3 → 100 MB
-----------------
Total   → 300 MB
```

To maintain a balance between **Data Protection & Recovery** and **Cost Control**, versioning should be paired with S3 Lifecycle Rules to transition or expire noncurrent object versions when they are no longer needed.

---

## 💡 Key Takeaway

Prevention and recovery are significantly cheaper than rebuilding lost data. S3 versioning provides a straightforward mechanism to safeguard objects from accidental changes and deletions, while proper lifecycle management ensures that old versions do not result in runaway storage costs.

> *This was Task 4 of 50 AWS tasks in the AWS phase of my KodeKloud 100 Days of Cloud Challenge.*
