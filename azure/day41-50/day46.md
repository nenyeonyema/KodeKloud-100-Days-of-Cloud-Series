# Azure Task 46 — VM Log Collection with Event Hubs and Blob Storage Backup

## Overview

This task demonstrates integrating an Azure Virtual Machine with both Azure Event Hubs and Azure Blob Storage for centralized log collection and backup. A Python script running on the VM sends logs simultaneously to an Event Hub (for real-time streaming) and uploads a backup copy to a Blob Storage container (for durable storage). This is a common pattern in cloud-native observability pipelines where logs need both real-time processing and long-term retention.

---

## Architecture

```
+-------------------+
|   Lab Host        |
|   /root/          |
|   send_logs.py    |
+--------+----------+
         |
         | scp
         v
+-------------------+        azure-eventhub SDK        +---------------------------+
|   nautilus-vm     +----------------------------------+   nautilus-namespace       |
|   East US         |                                  |   Event Hubs Namespace    |
|   74.235.76.47    |                                  |   Standard | Auto-inflate |
|                   |                                  |                           |
|  send_logs.py     |                                  |  +---------------------+  |
|  /home/azureuser/ |                                  |  |   nautilus-hub      |  |
+-------------------+                                  |  +---------------------+  |
         |                                             +---------------------------+
         | azure-storage-blob SDK
         v
+---------------------------+
|   nautilusst2280          |
|   Storage Account         |
|   East US | Standard_LRS  |
|                           |
|  +----------------------+ |
|  | nautilus-backup-2932 | |
|  | (public blob access) | |
|  |                      | |
|  |  logs.txt (18 bytes) | |
|  +----------------------+ |
+---------------------------+
```

---

## Prerequisites

- Azure CLI installed and authenticated
- Resource group: kml_rg_main-4bda7e1dd6c84d5f
- send_logs.py already present at /root/ on lab host
- Region: East US

---

## Step 1 — Login

```bash
az login -u kk_lab_user_main-4bda7e1dd6c84d5f@azurefreekmlprod.onmicrosoft.com -p "u^E7dK7L"
az group list --query "[0].name" -o tsv
```

---

## Step 2 — Create Event Hubs Namespace and Event Hub

```bash
az eventhubs namespace create \
  --name nautilus-namespace \
  --resource-group kml_rg_main-4bda7e1dd6c84d5f \
  --location eastus \
  --sku Standard \
  --enable-auto-inflate true \
  --maximum-throughput-units 10

az eventhubs eventhub create \
  --name nautilus-hub \
  --namespace-name nautilus-namespace \
  --resource-group kml_rg_main-4bda7e1dd6c84d5f
```

---

## Step 3 — Create Storage Account and Public Blob Container

```bash
az storage account create \
  --name nautilusst2280 \
  --resource-group kml_rg_main-4bda7e1dd6c84d5f \
  --location eastus \
  --sku Standard_LRS \
  --allow-blob-public-access true

az storage container create \
  --name nautilus-backup-2932 \
  --account-name nautilusst2280 \
  --public-access blob \
  --auth-mode login
```

`--public-access blob` allows anonymous read access to blobs inside the container while keeping container listing private.

---

## Step 4 — Create the VM

The lab subscription policy requires Standard_LRS disk SKU and disk size 128GB or less. The default Ubuntu image uses Premium SSD which is blocked by policy. The fix is to explicitly set `--storage-sku Standard_LRS` and `--os-disk-size-gb 30`:

```bash
az vm create \
  --resource-group kml_rg_main-4bda7e1dd6c84d5f \
  --name nautilus-vm \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard \
  --location eastus \
  --nsg-rule SSH \
  --storage-sku Standard_LRS \
  --os-disk-size-gb 30
```

VM output:
- Public IP: 74.235.76.47
- Private IP: 10.0.0.4
- State: VM running

**Note:** If previous failed VM attempts left orphaned disks, clean them up first:
```bash
az vm delete --resource-group kml_rg_main-4bda7e1dd6c84d5f --name nautilus-vm --yes
az disk list --resource-group kml_rg_main-4bda7e1dd6c84d5f \
  --query "[?contains(name, 'nautilus')].id" -o tsv | xargs -r az disk delete --yes --ids
```

---

## Step 5 — Get Both Connection Strings

```bash
# Event Hub connection string
az eventhubs namespace authorization-rule keys list \
  --resource-group kml_rg_main-4bda7e1dd6c84d5f \
  --namespace-name nautilus-namespace \
  --name RootManageSharedAccessKey \
  --query "primaryConnectionString" -o tsv

# Blob Storage connection string
az storage account show-connection-string \
  --name nautilusst2280 \
  --resource-group kml_rg_main-4bda7e1dd6c84d5f \
  --query "connectionString" -o tsv
```

---

## Step 6 — Copy Script to VM

```bash
scp /root/send_logs.py azureuser@74.235.76.47:/home/azureuser/send_logs.py
```

---

## Step 7 — Update Connection Strings in Script on VM

```bash
ssh azureuser@74.235.76.47 "sed -i 's|<Event Hub Connection String>|Endpoint=sb://nautilus-namespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=<KEY>|g' /home/azureuser/send_logs.py"

ssh azureuser@74.235.76.47 "sed -i 's|<Blob Storage Connection String>|DefaultEndpointsProtocol=https;AccountName=nautilusst2280;AccountKey=<KEY>;BlobEndpoint=https://nautilusst2280.blob.core.windows.net/|g' /home/azureuser/send_logs.py"
```

Verify the replacement:
```bash
ssh azureuser@74.235.76.47 "cat /home/azureuser/send_logs.py"
```

---

## Step 8 — Install Python SDKs on VM

```bash
ssh azureuser@74.235.76.47 "sudo apt-get update -y && sudo apt-get install -y python3-pip && python3 -m pip install azure-eventhub azure-storage-blob"
```

**Note:** `pip3` was not available on the VM — `python3 -m pip` is the correct way to invoke pip when the standalone pip3 binary is not installed.

---

## Step 9 — Run the Script Multiple Times

```bash
ssh azureuser@74.235.76.47 "python3 /home/azureuser/send_logs.py && python3 /home/azureuser/send_logs.py && python3 /home/azureuser/send_logs.py"
```

Output:
```
Log sent to Event Hub and backed up to Blob Storage.
Log sent to Event Hub and backed up to Blob Storage.
Log sent to Event Hub and backed up to Blob Storage.
```

---

## Step 10 — Verify Blob Storage Backup

```bash
az storage blob list \
  --account-name nautilusst2280 \
  --container-name nautilus-backup-2932 \
  --auth-mode login \
  --query "[].{name:name, size:properties.contentLength}" \
  -o table
```

Output:
```
Name      Size
--------  ------
logs.txt  18
```

---

## Step 11 — Verify Event Hub Metrics

```bash
az monitor metrics list \
  --resource /subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-4bda7e1dd6c84d5f/providers/Microsoft.EventHub/namespaces/nautilus-namespace \
  --metric IncomingMessages \
  --interval PT1H \
  --query "value[0].timeseries[0].data[-1].total" \
  -o tsv
```

Note: CLI metrics may show 0.0 due to a delay in metric aggregation. The script output confirming "Log sent to Event Hub" is the authoritative confirmation. Full metrics are visible in the Azure Portal under the Event Hub namespace.

---

## The Python Script (send_logs.py)

```python
import os
from azure.storage.blob import BlobServiceClient
from azure.eventhub import EventHubProducerClient, EventData

# Event Hub Configuration
eventhub_conn_str = "<Event Hub Connection String>"
eventhub_name = "nautilus-hub"
producer = EventHubProducerClient.from_connection_string(
    eventhub_conn_str, eventhub_name=eventhub_name
)

# Blob Storage Configuration
blob_conn_str = "<Blob Storage Connection String>"
blob_service_client = BlobServiceClient.from_connection_string(blob_conn_str)
blob_client = blob_service_client.get_blob_client(
    container="nautilus-backup-2932", blob="logs.txt"
)

# Generate and Send Logs
log_data = "Log entry from VM\n"

# Send to Event Hub
event_data_batch = producer.create_batch()
event_data_batch.add(EventData(log_data))
producer.send_batch(event_data_batch)

# Backup to Blob Storage
blob_client.upload_blob(log_data, blob_type="AppendBlob", overwrite=True)

print("Log sent to Event Hub and backed up to Blob Storage.")
```

---

## Key Concepts

**Why both Event Hubs and Blob Storage?**

| Destination | Purpose | Retention |
|---|---|---|
| Event Hub | Real-time streaming, downstream processing | 1-7 days (configurable) |
| Blob Storage | Durable long-term backup | As long as needed |

Event Hubs is the real-time pipe — data flows through it to consumers like Stream Analytics, Functions, or Databricks. Blob Storage is the permanent archive — logs stay there until explicitly deleted. Using both gives you real-time visibility AND long-term retention.

**Why AppendBlob for log backup?**
The script uses `blob_type="AppendBlob"` which is optimized for append-only workloads like logging. Each write adds to the end of the blob without rewriting existing content — exactly what you want for log files. Block blobs overwrite the whole file; append blobs grow incrementally.

**Why `python3 -m pip` instead of `pip3`?**
On fresh Ubuntu VMs, `pip3` as a standalone binary may not be installed even when Python 3 is present. `python3 -m pip` invokes pip as a Python module directly, which always works as long as pip is installed for that Python version.

**VM disk policy workaround**
The lab subscription policy blocks Premium SSD disks. The default Ubuntu 22.04 image on Standard_B1s uses Premium SSD by default. Explicitly passing `--storage-sku Standard_LRS` and `--os-disk-size-gb 30` forces Standard HDD and satisfies the policy. Previous attempts with `--os-disk-sku` failed because that flag is not recognized in this CLI version.

**Connection string authentication**
Both SDKs use connection string authentication — a simple, portable method that bundles the endpoint, account name, and key in one string. For production, Managed Identity is preferred to avoid storing credentials in scripts.

---

## Lessons Learned

1. Always delete orphaned disks from failed VM deployments before retrying — they block new deployments with the same name.
2. Lab subscription policies may block default disk SKUs — always specify `--storage-sku Standard_LRS` and `--os-disk-size-gb 30` explicitly.
3. `pip3` may not exist on fresh Ubuntu VMs — use `python3 -m pip` instead.
4. Event Hub CLI metrics have a delay — script output is the immediate confirmation of successful sends.
5. Use `AppendBlob` type for log backups to Blob Storage — it is optimized for append-only log workloads.

---

## Outcome

| Task | Result |
|---|---|
| nautilus-namespace created (Standard, Auto-inflate) | Done |
| nautilus-hub created (4 partitions) | Done |
| nautilusst2280 storage account created (Standard_LRS) | Done |
| nautilus-backup-2932 container created (public blob access) | Done |
| nautilus-vm created (East US, B1s, Standard_LRS disk) | Done |
| send_logs.py copied to VM | Done |
| Connection strings updated in script | Done |
| azure-eventhub and azure-storage-blob SDKs installed | Done |
| Script executed 3 times successfully | Done |
| logs.txt verified in nautilus-backup-2932 (18 bytes) | Done |
| Event Hub received incoming messages | Done |
