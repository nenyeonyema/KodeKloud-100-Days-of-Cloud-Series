# Azure Task 45 — Private AKS Cluster with Autoscaling

## Overview

This task demonstrates creating a production-ready Azure Kubernetes Service (AKS) cluster with private endpoint access, cluster autoscaling, and monitoring disabled. The cluster is configured with a specific Kubernetes version, node size, and autoscaling boundaries — a common setup for teams preparing a secure Kubernetes environment for application workloads.

---

## Architecture

```
+----------------------------------------------------------+
|              kml_rg_main-2cc939df8e894b8a                |
|              Resource Group | Central US                 |
|                                                          |
|   +------------------------------------------------------+|
|   |                   devops-aks                        ||
|   |              AKS Cluster (Private)                  ||
|   |              Kubernetes 1.33.0                      ||
|   |                                                     ||
|   |   API Server: Private endpoint only                 ||
|   |   privateFqdn: *.privatelink.centralus.azmk8s.io   ||
|   |   Monitoring: Disabled                              ||
|   |   RBAC: Enabled                                     ||
|   |   Identity: System Assigned Managed Identity        ||
|   |                                                     ||
|   |   +-----------------------------------------------+ ||
|   |   |           agentpool (Node Pool)               | ||
|   |   |           VM Size: Standard_D2s_v3            | ||
|   |   |           OS: Ubuntu 22.04                    | ||
|   |   |           Min nodes: 1                        | ||
|   |   |           Max nodes: 2                        | ||
|   |   |           Autoscaling: Enabled                | ||
|   |   |           Current count: 1                    | ||
|   |   +-----------------------------------------------+ ||
|   +------------------------------------------------------+|
+----------------------------------------------------------+

Internet --> BLOCKED (private cluster)
Internal VNet only --> API Server accessible
```

---

## Prerequisites

- Azure CLI installed and authenticated
- Resource group: kml_rg_main-2cc939df8e894b8a
- Region: Central US
- Kubernetes 1.33.0 available in Central US (confirmed)
- Standard_D2s_v3 available in Central US (confirmed)

---

## Step 1 — Login

```bash
az login -u kk_lab_user_main-2cc939df8e894b8a@azurefreekmlprod.onmicrosoft.com -p "6SA8BK-^"
```

---

## Step 2 — Get Resource Group

```bash
az group list --query "[0].name" -o tsv
```

Output: kml_rg_main-2cc939df8e894b8a

---

## Step 3 — Verify Kubernetes Version Availability

```bash
az aks get-versions \
  --location centralus \
  --output table
```

Confirmed: 1.33.0 is available in Central US under KubernetesOfficial support plan.

---

## Step 4 — Verify VM Size Availability

```bash
az vm list-sizes \
  --location centralus \
  --query "[?name=='Standard_D2s_v3'].{name:name, cores:numberOfCores}" \
  -o table
```

Output:
```
Name             Cores
---------------  -------
Standard_D2s_v3  2
```

---

## Step 5 — Create the AKS Cluster

```bash
az aks create \
  --resource-group kml_rg_main-2cc939df8e894b8a \
  --name devops-aks \
  --location centralus \
  --kubernetes-version 1.33.0 \
  --node-vm-size Standard_D2s_v3 \
  --node-count 1 \
  --min-count 1 \
  --max-count 2 \
  --enable-cluster-autoscaler \
  --enable-private-cluster \
  --nodepool-name agentpool \
  --generate-ssh-keys
```

**Flags explained:**
- `--kubernetes-version 1.33.0` — pins the exact Kubernetes version
- `--node-vm-size Standard_D2s_v3` — 2 vCPUs, 8GB RAM per node
- `--node-count 1` — starts with 1 node
- `--min-count 1` / `--max-count 2` — autoscaler boundaries
- `--enable-cluster-autoscaler` — enables automatic node scaling based on workload demand
- `--enable-private-cluster` — API server only accessible via private endpoint, not public internet
- `--nodepool-name agentpool` — names the default node pool agentpool as required
- `--generate-ssh-keys` — generates SSH keys for node access
- monitoring is NOT specified — disabled by default, satisfying the no-monitoring requirement

**Note:** `--disable-addons monitoring` is not a valid flag in this CLI version. Omitting monitoring flags is sufficient to keep it disabled.

---

## Step 6 — Verify the Cluster

```bash
az aks show \
  --resource-group kml_rg_main-2cc939df8e894b8a \
  --name devops-aks \
  --query "{name:name, version:kubernetesVersion, state:powerState.code, privateCluster:apiServerAccessProfile.enablePrivateCluster, location:location, monitoring:addonProfiles, nodePool:agentPoolProfiles[0].{name:name, vmSize:vmSize, minCount:minCount, maxCount:maxCount, autoscaler:enableAutoScaling}}" \
  -o json
```

Output:
```json
{
  "location": "centralus",
  "monitoring": null,
  "name": "devops-aks",
  "nodePool": {
    "autoscaler": true,
    "maxCount": 2,
    "minCount": 1,
    "name": "agentpool",
    "vmSize": "Standard_D2s_v3"
  },
  "privateCluster": true,
  "state": "Running",
  "version": "1.33.0"
}
```

---

## Requirements Verification

| Requirement | Expected | Actual | Status |
|---|---|---|---|
| Cluster name | devops-aks | devops-aks | Done |
| Kubernetes version | 1.33.0 | 1.33.0 | Done |
| Private endpoint | true | true | Done |
| Region | Central US | centralus | Done |
| Node pool name | agentpool | agentpool | Done |
| Node size | Standard_D2s_v3 | Standard_D2s_v3 | Done |
| Min node count | 1 | 1 | Done |
| Max node count | 2 | 2 | Done |
| Autoscaling | enabled | true | Done |
| Monitoring | disabled | null | Done |
| Power state | Running | Running | Done |

---

## Key Concepts

**What is AKS?**
Azure Kubernetes Service is a managed Kubernetes offering where Azure handles the control plane (API server, etcd, scheduler) for free. You only pay for the worker nodes (agent pool VMs). This reduces operational overhead compared to self-managed Kubernetes.

**Why private cluster?**
With `--enable-private-cluster`, the Kubernetes API server is only reachable via a private endpoint within the VNet — not from the public internet. This is a security best practice for production clusters, preventing external attackers from even reaching the API server.

**What is cluster autoscaling?**
The cluster autoscaler automatically adds nodes when pods cannot be scheduled due to insufficient resources, and removes nodes when they are underutilised. With min=1 and max=2, the cluster will always have at least 1 node and will scale up to 2 when workload demands it.

**AKS vs self-managed Kubernetes**

| Aspect | AKS | Self-managed |
|---|---|---|
| Control plane | Managed by Azure (free) | You manage and pay for it |
| Upgrades | One command | Manual, complex |
| Node scaling | Built-in autoscaler | Must configure manually |
| HA control plane | Automatic | Must design yourself |
| Cost | Pay for nodes only | Pay for everything |

**Standard_D2s_v3 specs**

| Spec | Value |
|---|---|
| vCPUs | 2 |
| Memory | 8 GB |
| Temp storage | 16 GB |
| Max data disks | 4 |
| Premium storage | Supported |

This is a lightweight but capable node size suitable for dev/test Kubernetes workloads and small production deployments.

**Why check versions before creating?**
AKS only supports specific Kubernetes versions per region, and the available patch versions (1.33.0, 1.33.1, etc.) differ from the minor version list. Always run `az aks get-versions` first to confirm the exact patch version is available before creating the cluster.

---

## Outcome

| Task | Result |
|---|---|
| Kubernetes 1.33.0 availability confirmed | Done |
| Standard_D2s_v3 availability confirmed | Done |
| devops-aks cluster created (Central US) | Done |
| Private endpoint enabled | Done |
| agentpool configured (D2s_v3, min=1, max=2) | Done |
| Cluster autoscaler enabled | Done |
| Monitoring disabled (addonProfiles: null) | Done |
| Cluster state: Running | Done |
| provisioningState: Succeeded | Done |
