# Azure Task 41 — To-Do App with Azure Table Storage

## Overview

This task demonstrates using Azure Table Storage to build the data layer of a simple To-Do application. Azure Table Storage is a NoSQL key-value store that is cost-effective for structured, non-relational data. A storage account and table are created via the Azure CLI, tasks are inserted as entities, and the results are verified by querying the table.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│              xfusiontablest18329                    │
│              Storage Account (LRS)                  │
│              East US                                │
│                                                     │
│   ┌─────────────────────────────────────────────┐   │
│   │            tasks (Table)                    │   │
│   │                                             │   │
│   │  ┌──────────────────────────────────────┐   │   │
│   │  │ PartitionKey: tasks                  │   │   │
│   │  │ RowKey: 1                            │   │   │
│   │  │ description: Learn Table Storage     │   │   │
│   │  │ status: completed                    │   │   │
│   │  └──────────────────────────────────────┘   │   │
│   │                                             │   │
│   │  ┌──────────────────────────────────────┐   │   │
│   │  │ PartitionKey: tasks                  │   │   │
│   │  │ RowKey: 2                            │   │   │
│   │  │ description: Build To-Do App         │   │   │
│   │  │ status: in-progress                  │   │   │
│   │  └──────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

---

## Prerequisites

- Azure CLI installed and authenticated
- Resource group: `kml_rg_main-7fc0153a15564c84`
- Region: East US

---

## Step 1 — Login

```bash
az login -u kk_lab_user_main-7fc0153a15564c84@azurefreekmlprod.onmicrosoft.com -p "Hv@NbTbk"
```

---

## Step 2 — Get Resource Group

```bash
az group list --query "[0].name" -o tsv
```

Output: `kml_rg_main-7fc0153a15564c84`

---

## Step 3 — Create the Storage Account

```bash
az storage account create \
  --name xfusiontablest18329 \
  --resource-group kml_rg_main-7fc0153a15564c84 \
  --location eastus \
  --sku Standard_LRS
```

---

## Step 4 — Get the Storage Account Key

```bash
az storage account keys list \
  --account-name xfusiontablest18329 \
  --resource-group kml_rg_main-7fc0153a15564c84 \
  --query "[0].value" -o tsv
```

This key is used to authenticate all subsequent table and entity operations.

---

## Step 5 — Create the Table

```bash
az storage table create \
  --name tasks \
  --account-name xfusiontablest18329 \
  --account-key <KEY>
```

---

## Step 6 — Insert Task 1

```bash
az storage entity insert \
  --account-name xfusiontablest18329 \
  --account-key <KEY> \
  --table-name tasks \
  --entity PartitionKey=tasks RowKey=1 description="Learn Table Storage" status=completed
```

Confirmation output:
```json
{
  "date": "2026-07-14T11:42:43+00:00",
  "etag": "W/\"datetime'2026-07-14T11%3A42%3A44.5921222Z'\"",
  "version": "2019-02-02"
}
```

---

## Step 7 — Insert Task 2

```bash
az storage entity insert \
  --account-name xfusiontablest18329 \
  --account-key <KEY> \
  --table-name tasks \
  --entity PartitionKey=tasks RowKey=2 description="Build To-Do App" status=in-progress
```

Confirmation output:
```json
{
  "date": "2026-07-14T11:45:09+00:00",
  "etag": "W/\"datetime'2026-07-14T11%3A45%3A10.0297487Z'\"",
  "version": "2019-02-02"
}
```

---

## Step 8 — Verify Both Tasks

```bash
az storage entity query \
  --account-name xfusiontablest18329 \
  --account-key <KEY> \
  --table-name tasks \
  --query "items[].{RowKey:RowKey, description:description, status:status}" \
  -o table
```

Expected output:
```
RowKey    Description           Status
--------  --------------------  -----------
1         Learn Table Storage   completed
2         Build To-Do App       in-progress
```

✅ Both tasks present with correct statuses.

---

## Key Concepts

**What is Azure Table Storage?**
Azure Table Storage is a NoSQL key-value store for structured data. It is schema-less — each entity (row) can have different properties — and scales to very large datasets at low cost. It is well suited for storing structured, non-relational data such as user data, device metadata, or task lists.

**PartitionKey and RowKey**
Every entity in Azure Table Storage is uniquely identified by the combination of `PartitionKey` and `RowKey`:

| Key | Purpose |
|---|---|
| PartitionKey | Groups related entities together; used for load balancing across storage nodes |
| RowKey | Unique identifier within a partition |

In this task all tasks share `PartitionKey=tasks`, meaning they are all in the same partition — appropriate for a small dataset. For large-scale apps, partition design is critical for performance.

**Why account key authentication?**
Account key auth gives full access to all storage services and is straightforward for CLI-based operations. For production workloads, Shared Access Signatures (SAS) with scoped permissions or Azure AD authentication are preferred.

**Table Storage vs Cosmos DB Table API**
Azure Table Storage and Cosmos DB Table API are compatible at the SDK level but differ in scale and features. Table Storage is cheaper and sufficient for simple structured data; Cosmos DB offers global distribution, lower latency guarantees, and automatic indexing at higher cost.

---

## Task Data Reference

| PartitionKey | RowKey | Description | Status |
|---|---|---|---|
| tasks | 1 | Learn Table Storage | completed |
| tasks | 2 | Build To-Do App | in-progress |

---

## Outcome

| Task | Result |
|---|---|
| Storage account xfusiontablest18329 created (East US, LRS) | ✅ |
| Table `tasks` created | ✅ |
| Task 1 inserted (RowKey=1, status=completed) | ✅ |
| Task 2 inserted (RowKey=2, status=in-progress) | ✅ |
| Query verification → both tasks with correct statuses | ✅ |
