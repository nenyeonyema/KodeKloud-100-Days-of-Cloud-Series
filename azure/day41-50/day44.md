# Azure Task 44 — VM Log Collection with Azure Event Hubs

## Overview

This task demonstrates integrating an Azure Virtual Machine with Azure Event Hubs for centralized log collection. An Event Hubs namespace and Event Hub are created, a pre-existing Python script on the VM is configured with the connection string, and logs are sent to the Event Hub. This pattern is commonly used in cloud-native observability pipelines where application logs are streamed to a central event ingestion service for processing, storage, or analysis.

---

## Architecture

```
+-------------------+        Python SDK          +---------------------------+
|   datacenter-vm   |  ----------------------->  |   datacenter-namespace    |
|   East US         |  azure-eventhub 5.15.1      |   Event Hubs Namespace    |
|                   |  EventHubProducerClient      |   Standard SKU            |
|  send_logs.py     |  10 log entries per run      |   Auto-inflate enabled    |
|  /home/azureuser/ |                             |                           |
+-------------------+                             |  +---------------------+  |
                                                  |  |   datacenter-hub    |  |
                                                  |  |   4 partitions      |  |
                                                  |  |   7-day retention   |  |
                                                  |  +---------------------+  |
                                                  +---------------------------+
```

---

## Prerequisites

- Azure CLI installed and authenticated
- Resource group: kml_rg_main-991b908afc7543f1
- datacenter-vm already exists in East US
- send_logs.py already present at /home/azureuser/ on the VM
- azure-eventhub Python SDK pre-installed on VM (version 5.15.1)

---

## Step 1 — Login

```bash
az login -u kk_lab_user_main-991b908afc7543f1@azurefreekmlprod.onmicrosoft.com -p "yvf87FyA"
```

---

## Step 2 — Get Resource Group and VM IP

```bash
az group list --query "[0].name" -o tsv

az vm list-ip-addresses \
  --name datacenter-vm \
  --resource-group kml_rg_main-991b908afc7543f1 \
  --query "[].virtualMachine.network.publicIpAddresses[0].ipAddress" -o tsv
```

---

## Step 3 — Create Event Hubs Namespace

```bash
az eventhubs namespace create \
  --name datacenter-namespace \
  --resource-group kml_rg_main-991b908afc7543f1 \
  --location eastus \
  --sku Standard \
  --enable-auto-inflate true \
  --maximum-throughput-units 10
```

**Flags explained:**
- `--sku Standard` — Standard tier supports Auto-inflate, consumer groups, and higher throughput than Basic
- `--enable-auto-inflate true` — Automatically scales throughput units up when traffic increases
- `--maximum-throughput-units 10` — Sets the upper limit for auto-inflate scaling

---

## Step 4 — Create the Event Hub

```bash
az eventhubs eventhub create \
  --name datacenter-hub \
  --namespace-name datacenter-namespace \
  --resource-group kml_rg_main-991b908afc7543f1
```

Event Hub details:
- Name: datacenter-hub
- Partitions: 4
- Retention: 7 days (168 hours)
- Status: Active

---

## Step 5 — Get the Connection String

```bash
az eventhubs namespace authorization-rule keys list \
  --resource-group kml_rg_main-991b908afc7543f1 \
  --namespace-name datacenter-namespace \
  --name RootManageSharedAccessKey \
  --query "primaryConnectionString" -o tsv
```

Output format:
```
Endpoint=sb://datacenter-namespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=<KEY>
```

---

## Step 6 — SSH into the VM and Inspect the Script

```bash
ssh azureuser@<VM_IP>
cat /home/azureuser/send_logs.py
```

Original script content:
```python
from azure.eventhub import EventHubProducerClient, EventData

# Event Hub Configuration
connection_str = "<your_event_hub_connection_string>"
event_hub_name = "datacenter-hub"

# Initialize the producer client
producer = EventHubProducerClient.from_connection_string(
    conn_str=connection_str,
    eventhub_name=event_hub_name
)

# Send logs to the Event Hub
with producer:
    for i in range(10):
        event_data_batch = producer.create_batch()
        event_data_batch.add(EventData(f"Log entry {i + 1}: Sample log message"))
        producer.send_batch(event_data_batch)
        print(f"Log entry {i + 1} sent.")
```

---

## Step 7 — Update the Connection String

```bash
sed -i 's|<your_event_hub_connection_string>|Endpoint=sb://datacenter-namespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=<KEY>|' /home/azureuser/send_logs.py
```

Verify the replacement:
```bash
cat /home/azureuser/send_logs.py
```

---

## Step 8 — Verify SDK is Installed

```bash
pip install azure-eventhub 2>/dev/null || pip3 install azure-eventhub
```

Output confirmed:
```
Requirement already satisfied: azure-eventhub in /usr/local/lib/python3.10/dist-packages (5.15.1)
```

The SDK was pre-installed on the VM.

---

## Step 9 — Run the Script Multiple Times

```bash
python3 /home/azureuser/send_logs.py
python3 /home/azureuser/send_logs.py
python3 /home/azureuser/send_logs.py
```

Output per run:
```
Log entry 1 sent.
Log entry 2 sent.
Log entry 3 sent.
Log entry 4 sent.
Log entry 5 sent.
Log entry 6 sent.
Log entry 7 sent.
Log entry 8 sent.
Log entry 9 sent.
Log entry 10 sent.
```

30 log entries sent in total across 3 runs.

---

## Step 10 — Verify in Azure Portal

Navigate to:
```
portal.azure.com -> datacenter-namespace -> datacenter-hub -> Metrics
```

Check the following metrics:
- **Incoming Messages** — should show 30+ messages
- **Incoming Bytes** — confirms data volume received
- **Successful Requests** — confirms no errors during send

---

## Key Concepts

**What is Azure Event Hubs?**
Azure Event Hubs is a fully managed, real-time data ingestion service capable of receiving and processing millions of events per second. It acts as the front door of an event pipeline — producers send data in, and consumers (like Azure Stream Analytics, Azure Functions, or custom apps) read and process it downstream.

**Event Hubs vs Service Bus vs Storage Queue**

| Service | Best for | Ordering | Retention |
|---|---|---|---|
| Event Hubs | High-volume telemetry and log streaming | Partition-level | 1-90 days |
| Service Bus | Enterprise messaging, transactions | Yes (sessions) | Until consumed |
| Storage Queue | Simple async task queuing | Approximate | 7 days (max 30) |

**What is Auto-inflate?**
Auto-inflate automatically increases the number of throughput units when traffic exceeds current capacity, up to the configured maximum. One throughput unit = 1 MB/s ingress and 2 MB/s egress. Without auto-inflate, exceeding throughput limits causes throttling errors.

**What are partitions?**
Partitions are ordered lanes within an Event Hub. Producers send events to partitions and consumers read from them independently. More partitions = higher parallelism for consumers. datacenter-hub was created with 4 partitions (default).

**Why use EventHubProducerClient with batches?**
The script uses `create_batch()` and `send_batch()` instead of sending events individually. Batching is more efficient — it reduces the number of network round trips and maximises throughput by packing multiple events into a single send operation.

**Connection string authentication vs Managed Identity**
This task uses a Shared Access Key connection string (RootManageSharedAccessKey) for simplicity. For production workloads, Managed Identity authentication is preferred — it eliminates hardcoded credentials and uses Azure AD for authentication automatically.

---

## Outcome

| Task | Result |
|---|---|
| Event Hubs namespace datacenter-namespace created (Standard, Auto-inflate) | Done |
| Event Hub datacenter-hub created (4 partitions, 7-day retention) | Done |
| Connection string retrieved from RootManageSharedAccessKey | Done |
| send_logs.py updated with real connection string | Done |
| azure-eventhub SDK confirmed installed (5.15.1) | Done |
| Script executed 3 times, 10 entries per run | Done |
| 30 log entries sent to datacenter-hub | Done |
| Metrics visible in Azure Portal | Done |
