# Azure Task 39 — Static Website Hosting on Azure Blob Storage

## Overview

This task demonstrates hosting a static website on Azure using an Azure Storage account's built-in static website feature. The storage account is configured for public access, and an `index.html` file is uploaded to the special `$web` container, making it accessible via a public Azure Storage URL.

---

## Architecture

```
┌─────────────────┐       az storage blob upload
│    Lab Host     │──────────────────────────────────────────┐
│  (azure-client) │                                          │
│  /root/         │                                          ▼
│  index.html     │          ┌──────────────────────────────────────────┐
└─────────────────┘          │       nautiluswebst30759                 │
                             │       Storage Account (LRS)              │
                             │       East US | Public Access Enabled    │
                             │                                          │
                             │   ┌──────────────────────────────────┐   │
                             │   │         $web container           │   │
                             │   │   (auto-created by Azure for     │   │
                             │   │    static website hosting)       │   │
                             │   │                                  │   │
                             │   │   index.html                     │   │
                             │   └──────────────────────────────────┘   │
                             └──────────────────────────────────────────┘
                                          │
                                          │ Public HTTPS
                                          ▼
                    https://nautiluswebst30759.z13.web.core.windows.net/
                             <h1>Welcome to KKE labs!</h1>
```

---

## Prerequisites

- Azure CLI installed and authenticated
- Resource group: `kml_rg_main-89208fe698c54482`
- `index.html` file already present at `/root/index.html` on the lab host
- Region: East US

---

## Step 1 — Login

```bash
az login -u kk_lab_user_main-89208fe698c54482@azurefreekmlprod.onmicrosoft.com -p "gh5n-@a&"
```

---

## Step 2 — Get Resource Group

```bash
az group list --query "[0].name" -o tsv
```

Output: `kml_rg_main-89208fe698c54482`

---

## Step 3 — Create the Storage Account

```bash
az storage account create \
  --name nautiluswebst30759 \
  --resource-group kml_rg_main-89208fe698c54482 \
  --location eastus \
  --sku Standard_LRS \
  --allow-blob-public-access true
```

**Flags explained:**
- `--sku Standard_LRS` — Locally-redundant storage, 3 copies within one datacenter
- `--allow-blob-public-access true` — Required for static website public access; without this the site returns 404 or 403

---

## Step 4 — Enable Static Website Hosting

```bash
az storage blob service-properties update \
  --account-name nautiluswebst30759 \
  --static-website \
  --index-document index.html \
  --auth-mode login
```

This command:
- Enables the static website feature on the storage account
- Sets `index.html` as the default document served at the root URL
- Auto-creates the `$web` container if it doesn't already exist

---

## Step 5 — Upload index.html to the $web Container

```bash
az storage blob upload \
  --account-name nautiluswebst30759 \
  --container-name '$web' \
  --name index.html \
  --file /root/index.html \
  --auth-mode login
```

Upload confirmation output:
```json
{
  "date": "2026-07-08T12:18:26+00:00",
  "etag": "\"0x8DEDCEB03BF64C4\"",
  "lastModified": "2026-07-08T12:18:27+00:00",
  "request_server_encrypted": true,
  "version": "2022-11-02"
}
```

**Note:** The container name `$web` must be wrapped in single quotes in bash to prevent shell variable expansion.

---

## Step 6 — Retrieve the Static Website URL

```bash
az storage account show \
  --name nautiluswebst30759 \
  --resource-group kml_rg_main-89208fe698c54482 \
  --query "primaryEndpoints.web" -o tsv
```

Output:
```
https://nautiluswebst30759.z13.web.core.windows.net/
```

---

## Step 7 — Verify the Website is Accessible

```bash
curl $(az storage account show \
  --name nautiluswebst30759 \
  --resource-group kml_rg_main-89208fe698c54482 \
  --query "primaryEndpoints.web" -o tsv)
```

Output:
```html
<html><body><h1>Welcome to KKE labs!</h1></body></html>
```

✅ Website is publicly accessible via the Azure Storage static website URL.

---

## Key Concepts

**What is the `$web` container?**
Azure automatically creates a special container named `$web` when static website hosting is enabled. Files uploaded here are served via the public static website endpoint. Unlike regular blob URLs, the static website endpoint serves `index.html` at the root path and handles 404 error pages if configured.

**Static website URL vs Blob URL**
Azure provides two different URLs for blob content:

| Type | Format | Use case |
|---|---|---|
| Blob URL | `https://<account>.blob.core.windows.net/$web/index.html` | Direct blob access |
| Static website URL | `https://<account>.z13.web.core.windows.net/` | Website hosting, index doc routing |

Only the static website URL serves `index.html` automatically at the root path.

**Why `--allow-blob-public-access true`?**
Static website hosting requires the storage account to allow public blob access. This is separate from the static website setting itself — both must be enabled for the site to be publicly reachable.

**Why single quotes around `'$web'`?**
In bash, `$web` without quotes would be interpreted as a shell variable (which is empty), resulting in `--container-name ""`. Single quotes prevent variable expansion and pass the literal string `$web` to the CLI.

---

## Outcome

| Task | Result |
|---|---|
| Storage account created (East US, LRS) | ✅ |
| Public blob access enabled | ✅ |
| Static website hosting enabled (index.html) | ✅ |
| index.html uploaded to $web container | ✅ |
| Static website URL retrieved | ✅ `https://nautiluswebst30759.z13.web.core.windows.net/` |
| curl verification → Welcome to KKE labs! | ✅ |
