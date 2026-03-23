# Azure-Backup-and-Azure-Site-Recovery

# Enterprise-Grade Backup & Disaster Recovery – Architected & Automated

![Image](https://learn.microsoft.com/en-us/azure/backup/media/guidance-best-practices/azure-backup-architecture.png)

![Image](https://learn.microsoft.com/en-us/azure/backup/media/backup-architecture/architecture-on-premises-mars.png)

![Image](https://learn.microsoft.com/en-us/azure/backup/media/backup-azure-vms-introduction/vmbackup-architecture.png)

![Image](https://learn.microsoft.com/en-us/azure/backup/media/backup-overview/azure-backup-overview.png)





### Backup vs Disaster Recovery

| Capability    | Azure Backup            | Azure Site Recovery (ASR) |
| ------------- | ----------------------- | ------------------------- |
| Primary goal  | **Data protection**     | **Business continuity**   |
| Recovery type | Point-in-time restore   | Near-real-time failover   |
| Scope         | Files, disks, VMs, DBs  | Entire VM / application   |
| Typical RPO   | Hours / Days            | Minutes                   |
| Typical RTO   | Minutes–Hours           | Minutes                   |
| Azure service | Recovery Services Vault | Recovery Services Vault   |




## 2️⃣ Azure Backup – Architecture & Flow

![Image](https://learn.microsoft.com/en-us/azure/backup/media/backup-azure-vms-introduction/vmbackup-architecture.png)

![Image](https://learn.microsoft.com/en-us/azure/backup/media/backup-azure-vms/instant-rp-flow.png)

![Image](https://learn.microsoft.com/en-us/azure/backup/media/backup-architecture/architecture-on-premises-mars.png)

### How Azure VM Backup Works

1. **Recovery Services Vault (RSV)** is created in the same region as the VM
2. **Backup policy** defines:

   * Schedule (daily / enhanced multiple times)
   * Retention (daily, weekly, monthly, yearly)
3. Azure takes a **snapshot of managed disks**
4. Only **changed blocks (delta)** are transferred to the vault
5. **Recovery points** are securely stored (encrypted, immutable optional)

### What Gets Backed Up

* OS Disk
* Data Disks
* Application-consistent state (VSS on Windows, scripts on Linux)




## 4️⃣ Azure Backup – Automation Patterns

### Terraform (Production-Ready Pattern)

```hcl
resource "azurerm_recovery_services_vault" "vault" {
  name                = var.vault_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "Standard"
}

resource "azurerm_backup_policy_vm" "policy" {
  name                = "daily-retention-policy"
  resource_group_name = azurerm_resource_group.rg.name
  recovery_vault_name = azurerm_recovery_services_vault.vault.name

  time = "23:00"

  retention_daily {
    count = 30
  }
}

resource "azurerm_backup_protected_vm" "vm" {
  resource_group_name = azurerm_resource_group.rg.name
  recovery_vault_name = azurerm_recovery_services_vault.vault.name
  source_vm_id        = var.vm_id
  backup_policy_id    = azurerm_backup_policy_vm.policy.id
}
```



### 3.1 Azure CLI for VM Backup

```bash
# Sign in
az login

# Set subscription (optional)
az account set --subscription "MySubscriptionId"

# Create resource group (if not exists)
az group create --name MyBackupRG --location eastus

# Create Recovery Services vault
az backup vault create \
  --resource-group MyBackupRG \
  --name MyRecoveryVault \
  --location eastus

# Configure storage redundancy (optional)
az backup vault backup-properties set \
  --resource-group MyBackupRG \
  --name MyRecoveryVault \
  --backup-storage-redundancy GeoRedundant

# Enable backup for VM
az backup protection enable-for-vm \
  --resource-group MyBackupRG \
  --vault-name MyRecoveryVault \
  --vm MyVMName \
  --policy-name DefaultPolicy

# Trigger an on-demand backup
az backup protection backup-now \
  --resource-group MyBackupRG \
  --vault-name MyRecoveryVault \
  --item-name MyVMName \
  --backup-management-type AzureIaasVM \
  --workload-type VM
```

These commands derive from official documentation. ([Microsoft Learn][7])

### 3.2 CLI for Restore (VM Disks)

```bash
# List recovery points
az backup recoverypoint list \
  --resource-group MyBackupRG \
  --vault-name MyRecoveryVault \
  --backup-management-type AzureIaasVM \
  --container-name IaasVMContainer;iaasvmcontainerv2;MyVMName \
  --item-name MyVMName \
  --query [0].name -o tsv

# Restore disks from recovery point
az backup restore restore-disks \
  --resource-group MyBackupRG \
  --vault-name MyRecoveryVault \
  --container-name IaasVMContainer;iaasvmcontainerv2;MyVMName \
  --item-name MyVMName \
  --recovery-point-id <RecoveryPointID> \
  --storage-account MyStorageAccount
```

From Microsoft docs. ([Microsoft Learn][9])
