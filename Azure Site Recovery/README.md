# Azure-Backup-and-Azure-Site-Recovery

  Azure Site Recovery (ASR) – DR Architecture

![Image](https://docs.microsoft.com/en-us/azure/architecture/solution-ideas/media/disaster-recovery-smb-azure-site-recovery.png)

![Image](https://learn.microsoft.com/en-us/azure/site-recovery/media/azure-to-azure-how-to-enable-replication-private-endpoints/architecture.png)

![Image](https://learn.microsoft.com/en-us/azure/site-recovery/media/concepts-azure-to-azure-architecture/failover-v2.png)



### What ASR Solves

* Azure region outage
* Large-scale infrastructure failure
* Regulatory RTO/RPO commitments

### Azure-to-Azure (A2A) Flow

1. Source VM disks replicated to target region
2. Continuous replication (near-real-time)
3. Recovery plans orchestrate:

   * Boot order
   * Network mapping
   * App dependencies
4. Failover → Commit → Re-protect → Failback

---

## 7️⃣ ASR Core Components

| Component                  | Purpose                  |
| -------------------------- | ------------------------ |
| Recovery Services Vault    | Central DR control       |
| Fabric                     | Azure region abstraction |
| Protection Container       | Logical VM grouping      |
| Replication Policy         | RPO & snapshot frequency |
| Replication-Protected Item | The VM                   |
| Recovery Plan              | Orchestrated failover    |

---

## 8️⃣ ASR – Terraform Automation Model

> **Reality check:**
> Terraform **configures replication**, but **failover is operational**
> (triggered via CLI/PowerShell/runbooks).

### Terraform Scope (Correct Usage)

✔ Vault
✔ Replication policy
✔ Protection containers
✔ VM replication

❌ Failover execution (intentional safety)

---

## 9️⃣ Backup + DR Reference Architecture (Recommended)

![Image](https://docs.microsoft.com/en-us/azure/architecture/solution-ideas/media/disaster-recovery-smb-azure-site-recovery.png)

![Image](https://learn.microsoft.com/en-us/azure/architecture/example-scenario/azure-virtual-desktop/images/azure-virtual-desktop-bcdr-pooled-host-pool.png)

![Image](https://learn.microsoft.com/en-us/azure/architecture/data-guide/images/dr-for-azure-data-platform-landing-zone-architecture.png)

### Layered Protection Model

```
Application
 ├─ Azure Site Recovery (Availability)
 │   ├─ Region failover
 │   ├─ Recovery Plans
 │
 └─ Azure Backup (Data Protection)
     ├─ Immutable backups
     ├─ Long-term retention
     ├─ Ransomware recovery
```

---

## 🔐 Security & Compliance Checklist

* ✅ Soft delete enabled
* ✅ Vault immutability locked
* ✅ RBAC separated (Operator ≠ Owner)
* ✅ Key Vault permissions validated (for encrypted VMs)
* ✅ Audit logs enabled
* ✅ Restore tested quarterly

---

## 🧪 Operational Excellence (What Auditors Ask)

| Item                   | Evidence                |
| ---------------------- | ----------------------- |
| Last successful backup | Backup job logs         |
| Restore test           | Screenshot / ticket     |
| DR test                | Test failover report    |
| RPO achieved           | ASR replication health  |
| RTO achieved           | Recovery plan execution |

---

## 📌 Architect Takeaways

* **Backup ≠ DR** – they solve different problems
* Automate **creation**, not **destruction**
* Test restores more often than backups
* Treat RSV as a **security boundary**
* Always document **who presses the failover button**

---

## 4. Disaster Recovery (Azure Site Recovery) – High-level Steps

Here’s a summary for DR using ASR:

1. Create Recovery Services vault (or reuse) in primary region.
2. Configure replication policy (RPO, retention).
3. For each VM, enable replication to target region/zone.
4. Perform **Test failover** to validate.
5. In outage case, perform **Failover**. After recovery, plan **Failback** to primary region.

CLI/ARM/Bicep code for ASR is more involved (fabric containers, replication-protected-items etc) — I can share a full module if you want.

# ✅ CLI / Azure PowerShell Snippets

### 1. Install/Configure CLI extension

```bash
# Install the ASR extension for the Azure CLI
az extension add --name site-recovery
```

(This gives you the `az site-recovery` commands). ([N2W Software][1])

### 2. Create Recovery Services Vault (if not already)

```bash
az group create --name MyRG --location eastus

az recovery-services vault create \
  --resource-group MyRG \
  --name MyRSVault \
  --location eastus
```

Then set vault context for ASR:

```bash
az site-recovery vault set \
  --vault-name MyRSVault \
  --resource-group MyRG
```

(Use the vault in subsequent commands) ([N2W Software][1])

### 3. Enable replication for an Azure VM (Azure to Azure scenario)

```bash
az site-recovery protection-container mapping create \
  --resource-group MyRG \
  --vault-name MyRSVault \
  --fabric-name PrimaryFabric \
  --protection-container PrimaryPC \
  --recovery-fabric RecoveryFabric \
  --recovery-protection-container RecoveryPC \
  --policy-name MyReplicationPolicy

az site-recovery protected-item create \
  --resource-group MyRG \
  --vault-name MyRSVault \
  --fabric-name PrimaryFabric \
  --protection-container PrimaryPC \
  --protection-container-name PrimaryPC \
  --name MyVMName-RPI \
  --policy-id <policyId> \
  --provider-details '{ "azureToAzure": { "osType": "Windows", "vmId": "<sourceVmId>", "recoveryResourceGroupId": "<rgId>", "recoveryAzureNetworkId": "<vnetId>", "recoverySubnetName": "<subnetName>" } }'
```

This is based on the `az site-recovery protected-item create` command. ([Microsoft Learn][2])
(You’ll need to replace placeholders with your specific fabric, containers, VM IDs etc.)

### 4. Failover / Test Failover / Commit / Re-protect

```bash
# Test failover (without committing)
az site-recovery protected-item planned-failover \
  --resource-group MyRG \
  --vault-name MyRSVault \
  --fabric-name PrimaryFabric \
  --protection-container PrimaryPC \
  --name MyVMName-RPI \
  --failover-direction PrimaryToRecovery

# Unplanned failover
az site-recovery protected-item unplanned-failover \
  --resource-group MyRG \
  --vault-name MyRSVault \
  --fabric-name PrimaryFabric \
  --protection-container PrimaryPC \
  --name MyVMName-RPI

# Commit failover
az site-recovery protected-item failover-commit \
  --resource-group MyRG \
  --vault-name MyRSVault \
  --fabric-name PrimaryFabric \
  --protection-container PrimaryPC \
  --name MyVMName-RPI

# Reprotect (after recovery)
az site-recovery protected-item reprotect \
  --resource-group MyRG \
  --vault-name MyRSVault \
  --fabric-name RecoveryFabric \
  --protection-container RecoveryPC \
  --name MyVMName-RPI
```

