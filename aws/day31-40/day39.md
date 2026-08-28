# AWS Task 39 — Static Website Hosting on Amazon S3

## Overview

This project hosts a static website on AWS using an Amazon S3 bucket. The setup demonstrates how to configure an S3 bucket for public static website hosting — disabling the default public access block, applying a bucket policy for public reads, enabling the website hosting feature, and uploading an HTML file that is immediately accessible via the S3 website endpoint URL.

---

## Architecture

```
Browser
    │
    │ HTTP GET
    ▼
S3 Website Endpoint
(datacenter-web-16488.s3-website-us-east-1.amazonaws.com)
    │
    │ serves
    ▼
index.html
(stored in S3 bucket: datacenter-web-16488)
```

---

## Prerequisites

- AWS CLI configured with appropriate credentials
- `index.html` file located at `/root/index.html` on the `aws-client` host
- IAM permissions for S3 in `us-east-1`

---

## Resources Created

| Resource | Name | Details |
|---|---|---|
| S3 Bucket | `datacenter-web-16488` | Public static website hosting enabled |
| Bucket Policy | Public read policy | Allows `s3:GetObject` for all principals |
| Website Config | Index document | `index.html` |
| Uploaded File | `index.html` | Served as the website homepage |

---

## Step-by-Step Setup

### 1. Create S3 Bucket

```bash
aws s3api create-bucket \
  --bucket datacenter-web-16488 \
  --region us-east-1

echo "Bucket created"
```

> **Note:** For `us-east-1` do NOT include `--create-bucket-configuration`. Other regions require `LocationConstraint` but us-east-1 does not — including it causes an error.

### 2. Disable Block Public Access

```bash
aws s3api put-public-access-block \
  --bucket datacenter-web-16488 \
  --public-access-block-configuration \
  "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

echo "Public access unblocked"
```

> **Note:** AWS blocks all public access on new S3 buckets by default. This must be explicitly disabled before a public bucket policy can be applied.

### 3. Enable Static Website Hosting

```bash
aws s3api put-bucket-website \
  --bucket datacenter-web-16488 \
  --website-configuration '{
    "IndexDocument": {"Suffix": "index.html"},
    "ErrorDocument": {"Key": "error.html"}
  }'

echo "Static website hosting enabled"
```

### 4. Apply Public Read Bucket Policy

```bash
cat > /tmp/bucket-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::datacenter-web-16488/*"
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket datacenter-web-16488 \
  --policy file:///tmp/bucket-policy.json

echo "Bucket policy applied"
```

### 5. Upload index.html

```bash
# Verify the file exists
cat /root/index.html

# Upload to S3 with correct content type
aws s3 cp /root/index.html s3://datacenter-web-16488/index.html \
  --content-type "text/html"

echo "index.html uploaded"

# Verify it is in the bucket
aws s3 ls s3://datacenter-web-16488/
```

---

## Verification

```bash
# S3 static website endpoint
WEBSITE_URL="http://datacenter-web-16488.s3-website-us-east-1.amazonaws.com"
echo "Website URL: $WEBSITE_URL"

# Test HTTP response
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" $WEBSITE_URL

# View page content
curl -s $WEBSITE_URL
```

**Expected output:**
```
HTTP Status: 200
<!DOCTYPE html> ... (contents of index.html)
```

Visit in browser:
```
http://datacenter-web-16488.s3-website-us-east-1.amazonaws.com
```

---

## S3 Website Endpoint vs S3 REST Endpoint

| Feature | Website Endpoint | REST Endpoint |
|---|---|---|
| URL format | `bucket.s3-website-region.amazonaws.com` | `bucket.s3.amazonaws.com` |
| Serves index.html automatically | Yes | No |
| Custom error pages | Yes | No |
| Supports HTTPS | No — use CloudFront | Yes |
| Public access required | Yes | Not necessarily |

> For HTTPS support on a static S3 website, place a **CloudFront distribution** in front of the bucket.

---

## How Public Access Works

```
Default S3 bucket (new)
    │
    ├── BlockPublicAcls=true
    ├── BlockPublicPolicy=true      ← Must disable all four
    ├── IgnorePublicAcls=true          before policy applies
    └── RestrictPublicBuckets=true
    │
    ▼
put-public-access-block (all false)
    │
    ▼
Bucket Policy — Principal: "*", Action: s3:GetObject
    │
    ▼
Anyone on the internet can GET objects from the bucket
```

The public access block acts as an **override** — even if a bucket policy grants public access, the block settings silently prevent it. Both must be configured correctly.

---

## Security Highlights

- Only `s3:GetObject` is granted publicly — no write, delete, or list permissions
- The policy is scoped to objects inside the bucket (`/*`) not the bucket itself
- For production workloads, add **CloudFront** in front of S3 for HTTPS, caching, and DDoS protection
- Enable **S3 access logging** to track who is accessing the website

---

## Troubleshooting

| Issue | Cause | Fix |
|---|---|---|
| `403 Forbidden` on website URL | Public access block still enabled | Re-run `put-public-access-block` with all values `false` |
| `404 Not Found` on website URL | `index.html` not uploaded or wrong filename | Verify with `aws s3 ls s3://datacenter-web-16488/` |
| `NoSuchWebsiteConfiguration` | Website hosting not enabled | Re-run `put-bucket-website` command |
| Policy apply fails with `Access Denied` | Block public policy still active | Disable `BlockPublicPolicy` first then apply the policy |
| `--create-bucket-configuration` error | Included for us-east-1 | Remove it — us-east-1 does not accept this parameter |

---

## Technologies Used

- **AWS S3** — Object storage and static website hosting
- **S3 Bucket Policy** — Public read access control
- **S3 Website Endpoint** — HTTP serving of static content
- **HTML** — Static website content
- **AWS CLI** — Full infrastructure provisioning
