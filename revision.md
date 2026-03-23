# 📦 **Backup & Recovery in Azure (Revised & Complete)**

## 🔄 **Backup vs Disaster Recovery (DR)**

| **Backup**                               | **Disaster Recovery (DR)**          |
| ---------------------------------------- | ----------------------------------- |
| Focuses on **data protection**           | Focuses on **business continuity**  |
| Point-in-time data copy                  | Near-real-time replication          |
| Item/VM-level restore                    | Region/site failover & failback     |
| **Azure Backup** (RSV / Data Protection) | **Azure Site Recovery (ASR)** (RSV) |

> Tip: Use **Backup** for corruption/ransomware/accidental deletes; use **ASR** for **regional outage** resilience and RTO/RPO commitments.

---

## 🛡️ **Azure Backup (VMs) – What You Get**

* Protect **Azure VMs, Files, SQL in Azure VM, SAP HANA**.
* Centralized control via **Recovery Services Vault (RSV)** or **Backup Center**.
* **Long-term retention** (daily/weekly/monthly/yearly).
* **Security**: encryption at rest, **soft delete**, **immutability (preview/GA by region)**, **MFA delete**, **alerts**, RBAC.

---

## ⚙️ **Prepare (Once Per Subscription/Tenant)**

### ✅ Prereqs

* Owner/Contributor on RG + **Backup Contributor** on the vault
* **Azure CLI ≥ 2.58** (`az upgrade`) and logged in (`az login`)
* Decide **Source Region** and **Target Region** (for ASR)

### ✅ Naming & Vars (customize once)

```bash
# ---------- EDIT THESE ----------
LOCATION_SRC="eastus"
LOCATION_DR="westus2"                           # ASR target
RG="rg-backup-demo"
VAULT="rsv-backup-demo"
VM_NAME="demo-vm-01"                            # existing VM
POLICY_NAME="Daily-7-Weekly-4-Monthly-12-Yearly-5"
STORAGE_ACCOUNT="stbackup$RANDOM"               # for restore-disks
# --------------------------------
```

---

## 🏗️ **Create RG & Recovery Services Vault**

```bash
az group create -n "$RG" -l "$LOCATION_SRC"

# Create RSV
az backup vault create \
  --resource-group "$RG" \
  --name "$VAULT" \
  --location "$LOCATION_SRC"

# Harden the vault (recommended)
# 1) GRS storage
az backup vault backup-properties set \
  --name "$VAULT" --resource-group "$RG" \
  --backup-storage-redundancy GeoRedundant

# 2) Enable soft delete (default is usually On; ensure it’s on)
az backup vault backup-properties show \
  --name "$VAULT" --resource-group "$RG" \
  --query softDeleteFeatureState

# 3) (Optional) Immutable vault (if available in region; prevents tampering)
# az backup vault backup-properties set \
#   --name "$VAULT" --resource-group "$RG" \
#   --immutability-state Locked
```

---

## 📜 **Create a Custom VM Backup Policy**

```bash
# Create a daily policy with weekly/monthly/yearly retention
az backup policy create \
  --resource-group "$RG" \
  --vault-name "$VAULT" \
  --name "$POLICY_NAME" \
  --policy "{\"backupManagementType\":\"AzureIaasVM\",\"schedulePolicy\":{\"scheduleRunFrequency\":\"Daily\",\"scheduleRunTimes\":[\"2025-08-30T20:00:00Z\"]},\"retentionPolicy\":{\"retentionPolicyType\":\"LongTermRetentionPolicy\",\"dailySchedule\":{\"retentionTimes\":[\"2025-08-30T20:00:00Z\"],\"retentionDuration\":{\"count\":7,\"durationType\":\"Days\"}},\"weeklySchedule\":{\"daysOfTheWeek\":[\"Saturday\"],\"retentionDuration\":{\"count\":4,\"durationType\":\"Weeks\"}},\"monthlySchedule\":{\"retentionScheduleFormatType\":\"Weekly\",\"retentionScheduleWeekly\":{\"daysOfTheWeek\":[\"Saturday\"],\"weeksOfTheMonth\":[\"First\"]},\"retentionDuration\":{\"count\":12,\"durationType\":\"Months\"}},\"yearlySchedule\":{\"retentionScheduleFormatType\":\"Weekly\",\"monthsOfYear\":[\"January\"],\"retentionScheduleWeekly\":{\"daysOfTheWeek\":[\"Saturday\"],\"weeksOfTheMonth\":[\"First\"]},\"retentionDuration\":{\"count\":5,\"durationType\":\"Years\"}}}}"
```

> Notes
> • Time is in **UTC**. Adjust `scheduleRunTimes` to your desired window.
> • You can also use the default policy (`DefaultPolicy`) if you don’t need custom retention.

---

## 🧷 **Enable Backup for a VM**

```bash
# Protect the VM under the policy
az backup protection enable-for-vm \
  --resource-group "$RG" \
  --vault-name "$VAULT" \
  --vm "$VM_NAME" \
  --policy-name "$POLICY_NAME"
```

---

## ▶️ **Run an On-Demand Backup (Now) – Correct Way**

The `backup-now` call needs the **container** and **item** names. We’ll query and pass them programmatically:

```bash
# Get container (IaaS VM container)
CONTAINER=$(az backup container list \
  --resource-group "$RG" \
  --vault-name "$VAULT" \
  --backup-management-type AzureIaasVM \
  --query "[?contains(properties.friendlyName, '$VM_NAME')].name" -o tsv)

# Get item (protected VM item)
ITEM=$(az backup item list \
  --resource-group "$RG" \
  --vault-name "$VAULT" \
  --container-name "$CONTAINER" \
  --workload-type VM \
  --query "[?properties.friendlyName=='$VM_NAME'].name" -o tsv)

# Trigger backup now with 30-day retention for this ad-hoc point
az backup protection backup-now \
  --resource-group "$RG" \
  --vault-name "$VAULT" \
  --container-name "$CONTAINER" \
  --item-name "$ITEM" \
  --retain-until 30-09-2025
```

---

## ♻️ **Restore VM (Portal & CLI)**

### ✅ Portal (simple)

1. **Recovery Services Vault** → **Backup items** → **Azure Virtual Machine** → *Select VM*
2. **Restore VM** → choose **Restore point** → **Create new VM** or **Restore disks**
3. Review → **Restore**

### ✅ CLI – List Recovery Points & Restore Disks

```bash
# List recovery points
az backup recoverypoint list \
  --resource-group "$RG" \
  --vault-name "$VAULT" \
  --container-name "$CONTAINER" \
  --item-name "$ITEM" \
  -o table

# Choose a RecoveryPointId from above
RP_ID="<COPY_FROM_TABLE>"

# Create a storage account to land restored disks (if not already)
az storage account create -n "$STORAGE_ACCOUNT" -g "$RG" -l "$LOCATION_SRC" --sku Standard_LRS

# Restore disks (you can build a VM from the restored VHDs)
az backup restore restore-disks \
  --resource-group "$RG" \
  --vault-name "$VAULT" \
  --container-name "$CONTAINER" \
  --item-name "$ITEM" \
  --rp-name "$RP_ID" \
  --storage-account "$STORAGE_ACCOUNT" \
  --target-resource-group "$RG"
```

> Tip: After **restore-disks**, turn the restored VHD(s) into a **managed disk**, then **create VM** from that disk.

---

## 🏢 **Azure Site Recovery (ASR) – Overview & Steps**

**Use case:** Replicate Azure VMs from **\$LOCATION\_SRC → \$LOCATION\_DR** with orchestrated failover/failback.

### ✅ ASR Concepts (Azure→Azure A2A)

* **Fabric**: Azure (auto)
* **Protection container**: logical group per region
* **Replication policy**: RPO/Snapshot frequency/Retention
* **Protection container mapping**: binds source container + policy to target container
* **Replication-protected item (RPI)**: your protected VM

### ✅ ASR Setup – Recommended (Portal First)

1. Vault → **Site Recovery** → **Enable replication** (Azure to Azure)
2. Choose **Source RG/Region**, **Target Region**, **Target RG**, **Target VNet**, **Cache storage**
3. Create **Replication policy** (e.g., 5-min RPO, 24hr snapshots)
4. **Enable replication** for VM(s)
5. **Test failover** to validation network (no impact)
6. **Create Recovery Plan** for multi-VM app orchestration
7. **Failover** (disaster) → **Commit** → later **Failback**

### 📌 Minimal ASR CLI (A2A) – Pragmatic Example

> ASR CLI is verbose; below is a lean, working shape. For multi-VM orchestration, prefer **Recovery Plans** in Portal.

```bash
# Ensure target RG and a VNet exist in DR region
RG_DR="${RG}-dr"
VNET_DR="vnet-dr"
SUBNET_DR="subnet-dr"

az group create -n "$RG_DR" -l "$LOCATION_DR"
az network vnet create -g "$RG_DR" -n "$VNET_DR" -l "$LOCATION_DR" --address-prefixes 10.40.0.0/16 \
  --subnet-name "$SUBNET_DR" --subnet-prefix 10.40.1.0/24

# Get default ASR fabric/container (Azure)
FABRIC_NAME="Azure"
SRC_CONTAINER=$(az site-recovery protection-container list \
  --resource-group "$RG" --vault-name "$VAULT" \
  --query "[?contains(name, 'default')].name" -o tsv)

# Create target container in DR region (Azure auto-manages one per region)
# For A2A, you typically reference the target container via its name:
TGT_CONTAINER=$(az site-recovery protection-container list \
  --resource-group "$RG" --vault-name "$VAULT" \
  --query "[?contains(name, 'default')].name" -o tsv)

# Create a replication policy
ASR_POLICY="A2A-5min-24hr"
az site-recovery policy create \
  --resource-group "$RG" --vault-name "$VAULT" \
  --name "$ASR_POLICY" \
  --provider-type A2A \
  --recovery-point-retention-in-hours 24 \
  --application-consistent-snapshot-frequency-in-hours 1

# Map source container to target container using the policy
MAPPING_NAME="map-src-to-dr"
az site-recovery protection-container-mapping create \
  --resource-group "$RG" --vault-name "$VAULT" \
  --name "$MAPPING_NAME" \
  --policy-name "$ASR_POLICY" \
  --source-protection-container-name "$SRC_CONTAINER" \
  --target-protection-container-name "$TGT_CONTAINER"

# Discover protectable item (the VM)
PROTECTABLE_ID=$(az site-recovery protectable-item list \
  --resource-group "$RG" --vault-name "$VAULT" \
  --fabric-name "$FABRIC_NAME" \
  --protection-container-name "$SRC_CONTAINER" \
  --query "[?properties.friendlyName=='$VM_NAME'].name" -o tsv)

# Protect (enable replication). For A2A you must pass target details
RPI_NAME="$VM_NAME-rpi"
az site-recovery replication-protected-item create \
  --resource-group "$RG" --vault-name "$VAULT" \
  --fabric-name "$FABRIC_NAME" \
  --protection-container-name "$SRC_CONTAINER" \
  --name "$RPI_NAME" \
  --protectable-item-name "$PROTECTABLE_ID" \
  --policy-name "$ASR_POLICY" \
  --protectable-item-type "VirtualMachine" \
  --auto-protect true \
  --recovery-azure-resource-group-id "$(az group show -n "$RG_DR" --query id -o tsv)" \
  --recovery-azure-network-id "$(az network vnet show -g "$RG_DR" -n "$VNET_DR" --query id -o tsv)" \
  --recovery-azure-subnet-name "$SUBNET_DR"
```

> Wait for **Protection State = Protected** (initial seeding completes).

### ▶️ Test Failover & Commit (Non-disruptive)

```bash
# Create a simple recovery plan with one VM (optional but recommended)
RP="rp-$VM_NAME"
az site-recovery recovery-plan create \
  --resource-group "$RG" --vault-name "$VAULT" \
  --name "$RP" --primary-fabric-id "$FABRIC_NAME" \
  --recovery-fabric-id "$FABRIC_NAME" \
  --group-type Boot \
  --replication-protected-items "$RPI_NAME"

# Test failover (uses test network; VM comes up in DR as test)
az site-recovery recovery-plan test-failover \
  --resource-group "$RG" --vault-name "$VAULT" \
  --name "$RP" --failover-direction PrimaryToRecovery

# After validation, clean up the test
az site-recovery recovery-plan test-failover-cleanup \
  --resource-group "$RG" --vault-name "$VAULT" \
  --name "$RP" --comments "Validated"
```

### 🚨 Actual Failover & Commit

```bash
# Initiate failover (planned/unplanned). Here: Primary -> DR
az site-recovery recovery-plan failover \
  --resource-group "$RG" --vault-name "$VAULT" \
  --name "$RP" --failover-direction PrimaryToRecovery

# Commit (makes DR the new primary)
az site-recovery recovery-plan failover-commit \
  --resource-group "$RG" --vault-name "$VAULT" \
  --name "$RP"
```

> Later you can perform **re-protect** (reverse replication) and **failback** when the original region is ready.

---

## 🔔 **Ops Hygiene & Cost/Security Gotchas**

* **Backup Center** for fleet view, jobs, alerts, reports.
* Enable **Azure Monitor alerts** for backup failures & ASR health.
* Keep **GRS** for vaults; consider **Cross-Region Restore** capability (where available).
* **Soft delete + Immutability** to resist ransomware.
* Tag protected items with `costCenter`, `owner`, `recoveryTier`.
* Periodically **Test Failover** and document RTO/RPO evidence.

---

## ✅ Quick Validation Checklist

* Vault has **GRS**, **Soft Delete = Enabled**, **(Optional) Immutable = Locked**
* Policy schedules are in **UTC** and match backup window
* At least one **Successful Backup Job** present
* You can **list recovery points** for the VM
* **Test restore** (disks) succeeds into a staging RG/SA
* ASR **Protection State = Protected**
* **Test Failover** succeeded and was cleaned up

---
