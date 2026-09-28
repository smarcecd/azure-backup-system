# Azure Automated Backup System

**Azure Blob Storage · Azure Monitor · Logic Apps · Log Analytics · Terraform**

![Terraform](https://img.shields.io/badge/Terraform-%3E%3D1.3.0-844FBA?logo=terraform&logoColor=white)
![Azure](https://img.shields.io/badge/Azure-East%20US-0078D4?logo=microsoftazure&logoColor=white)
![Status](https://img.shields.io/badge/Status-Lab%20Ready-brightgreen)

A Terraform-deployed backup platform on Azure: a geo-redundant storage account with versioning, soft delete, and tiered lifecycle rules, backed by diagnostic logging, a daily Logic App confirmation email, and a Monitor alert that fires when backup writes stop.

Watch me building this lab here: 🎥 **[Add your video link here]**

---

## 🔗 Lab Overview

| Component | Details |
|---|---|
| **Cloud Provider** | Microsoft Azure |
| **Region** | East US |
| **Infrastructure as Code** | Terraform v1.3+ |
| **Resource Group** | `rg-backup-[yourname]` |
| **Primary Backup Store** | Storage Account `stbackup[yourname]` (RA-GRS) |
| **Containers** | `documents`, `database-exports`, `application-files` |
| **Data Protection** | Blob versioning, blob soft delete (30 days), TLS 1.2 minimum |
| **Cost Optimization** | Lifecycle policy: Cool at 30 days, Archive at 90 days, delete at 365 days |
| **Logging** | Log Analytics Workspace + Storage Diagnostic Settings |
| **Automation** | Logic App, daily backup confirmation email (8:00 AM) |
| **Alerting** | Monitor Alert Rule `alert-no-backup-writes` |
| **Difficulty** | Beginner to Intermediate |

---

## 🎯 Purpose of This Lab

Backups only matter if they are protected, monitored, and verified. This lab shows how to build a small but realistic backup solution on Azure where the whole environment is reproducible from code.

By the end of this lab you will have:

- Provisioned a **geo-redundant storage account** as a central backup target using Terraform
- Enabled **blob versioning and soft delete** so overwritten or deleted files can be recovered
- Applied a **lifecycle management policy** that moves aging data to cheaper tiers automatically
- Sent storage logs to **Log Analytics** for visibility and auditing
- Built a **Logic App** that emails a daily backup status confirmation
- Configured a **Monitor alert** that notifies you if backup writes stop
- Validated the setup by uploading a test file and confirming that versions are kept

---

## ✅ Prerequisites

- [ ] [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) installed and authenticated (`az login`)
- [ ] [Terraform](https://developer.hashicorp.com/terraform/downloads) **v1.3+** installed
- [ ] Active Azure subscription with **Contributor** rights, plus permission to create role assignments (Owner or User Access Administrator) for Step 6 (see [Troubleshooting](#-troubleshooting) if you hit `AuthorizationFailed`)
- [ ] An **Office 365 Outlook** account to authorize the Logic App email action
- [ ] Git for Windows/macOS

> If you've already completed a previous lab in this series, Terraform and the Azure CLI should already be installed.

---

## 📁 Project Structure

```text
rg-backup-[yourname]
├── Storage Account (primary backup store)
│   ├── Container: documents
│   ├── Container: database-exports
│   ├── Container: application-files
│   ├── Blob Versioning (enabled)
│   └── Lifecycle Policy (30d cool → 90d archive)
├── Log Analytics Workspace
├── Storage Diagnostic Settings     → logs to Log Analytics
├── Logic App Workflow              → daily backup confirmation email
└── Monitor Alert Rule              → fires if storage writes stop
```

---

## 🚀 Deployment Guide

### 📥 Step 1 — Clone This Repository

```powershell
git clone https://github.com/<your-github-username>/azure-backup-system.git
```

Move into the newly created folder:

```powershell
cd azure-backup-system
```

### 🌐 Step 2 — Log In to Azure

```powershell
az login
```

### ⚙️ Step 3 — Configure Variables

Update your information in **terraform.tfvars** (copy the example variables file first if your copy of the repo includes one) and fill in your own values:

```hcl
yourname     = "yourname"
location     = "East US"
alert_email  = "your.email@example.com"
```

### 🏗️ Step 4 — Deploy Infrastructure

```powershell
terraform init
terraform plan
terraform apply
```

The storage account connection string is marked `sensitive = true`, so Terraform will not print it in plain text in your terminal. To view it:

```powershell
terraform output -raw storage_account_connection_string
```

### 🔧 Step 5 — Configure the Logic App (Portal)

Terraform creates the Logic App shell, but the workflow itself is built in the portal. You will create a daily schedule, have it count the files in the `documents` container, and email you the result as a backup confirmation.

1. Open `rg-backup-[yourname]` → Development Tools → **Logic app designer**
2. **Add a trigger** → Search and select **Recurrence**
   - Interval: `1`
   - Frequency: `Day`
   - Time zone: your time zone (mine is EST)
   - Start time: `2026-09-27T08:00:00`
   - At these hours: `8`
3. Click the **+** → Select **Add New Interaction** → Search for **Azure Blob Storage** → Click **See more** → Select **List blobs (V2)** → Change the **Authentication Type** to **Logic Apps Managed Identity**
   - Connection name: `new_conn_bk`
   - Click **Create**
   - Under **Parameters**:
     - Storage account name or blob endpoint: type your storage account name
     - Folder: `documents`
4. Click the **+** → Select **Add New Interaction** → Search for **Office 365 Outlook** → Select **Send an email (V2)** → Sign in to your Outlook account when prompted, then fill in the email:
   - **To:** your alert email
   - **Subject:** `Daily Backup Confirmation — @{formatDateTime(utcNow(), 'yyyy-MM-dd')}`
   - **Body:** `Backup system status: Active. Files in documents container: @{length(body('Lists_blobs_(V2)')?['value'])}. All backup containers are protected and healthy.`
5. Click **Save**

### 🗂️ Step 6 — Upload a Test File and Verify Versioning

#### 6.1 — Enable Azure Storage Permissions for Your User Account

Get your user's object ID:

```powershell
az ad signed-in-user show --query id -o tsv
```

Copy the output. That is your Azure AD user ID.

Assign RBAC on the storage account:

```powershell
az role assignment create `
  --role "Storage Blob Data Contributor" `
  --assignee <your-user-id> `
  --scope "/subscriptions/<your-subscription-id>/resourceGroups/rg-backup-yourname/providers/Microsoft.Storage/storageAccounts/stbackupyourname"
```

Replace `<your-user-id>` with the ID from the previous step, `<your-subscription-id>` with your subscription ID, and `yourname` in the resource group and storage account names with your own value.

 ⚠️ Once this role is applied, Azure may take 30–60 seconds to propagate. ⚠️

#### 6.2 — Create and Upload a Test File

Change `YourStorageAccountName` and paste into PowerShell:

```powershell
"Backup test file created $(Get-Date)" | Out-File -FilePath "$env:TEMP\backup_test.txt" -Encoding utf8

az storage blob upload `
  --account-name YourStorageAccountName `
  --container-name documents `
  --name test/backup_test.txt `
  --file "$env:TEMP\backup_test.txt" `
  --auth-mode login
```

#### 6.3 — Overwrite the File to Create a Second Version

```powershell
"Updated content — second version $(Get-Date)" | Out-File -FilePath "$env:TEMP\backup_test.txt" -Encoding utf8

az storage blob upload `
  --account-name YourStorageAccountName `
  --container-name documents `
  --name test/backup_test.txt `
  --file "$env:TEMP\backup_test.txt" `
  --auth-mode login `
  --overwrite
```

#### 6.4 — List the Versions to Confirm Both Exist

```powershell
az storage blob list `
  --account-name YourStorageAccountName `
  --container-name documents `
  --include v `
  --auth-mode login `
  --output table
```

You should see two entries for `test/backup_test.txt`: the current version and the previous one.

### 📝 Step 7 — Verification Checklist

**✔ Resource group deployed**

Portal → Resource Groups → `rg-backup-[yourname]`

<img width="719" height="335" alt="Screenshot 2026-09-28 134801" src="https://github.com/user-attachments/assets/2a79a049-317b-47dc-b60f-83c68d681372" />



**✔ Storage Account and Versioning**

Home → Storage accounts → `stbackupyourname` → Overview → Properties tab

<img width="817" height="404" alt="Screenshot 2026-09-28 135154" src="https://github.com/user-attachments/assets/a940bfc8-4d3f-4373-819d-600bbaa32132" />


| Setting | Expected Value | Why It Matters |
|---|---|---|
| Replication | Read-Access Geo-Redundant Storage (RA-GRS) | Data replicated across two Azure regions |
| Versioning | Enabled | Every file version is kept for recovery |
| Blob soft delete | Enabled (30 days) | Deleted files are recoverable for 30 days |
| Minimum TLS version | Version 1.2 | Secure connections enforced |



**✔ Lifecycle Policy**

Home → Storage accounts → `stbackupyourname` → Data management → Lifecycle management

<img width="869" height="340" alt="Screenshot 2026-09-28 140743" src="https://github.com/user-attachments/assets/7f5e6098-a293-47e5-9df6-30d465c6696a" /> 


You should see the `backup-lifecycle` rule configured with:

- Move to Cool tier after 30 days
- Move to Archive tier after 90 days
- Delete after 365 days
- Delete old versions after 30 days

 <img width="472" height="356" alt="image" src="https://github.com/user-attachments/assets/030b4f53-573e-43bd-b97f-baf9f8783f86" />



**✔ Storage Containers**

Home → Storage accounts → `stbackupyourname` → Data storage → Containers

<img width="947" height="254" alt="Screenshot 2026-09-28 140958" src="https://github.com/user-attachments/assets/3a04996a-6679-430f-8305-8834a345f6ae" />


You should see four containers: `$logs` (auto-created by Azure for diagnostics), `application-files`, `database-exports`, and `documents`, all with **Private** access.



**✔ Logic App is live**

Home → Logic Apps → Status: **Enabled**

<img width="941" height="274" alt="Screenshot 2026-09-28 141058" src="https://github.com/user-attachments/assets/569a1b28-5211-4e47-9782-d8998f530feb" />

- Run history shows a successful test

<img width="845" height="232" alt="Screenshot 2026-09-28 141152" src="https://github.com/user-attachments/assets/3e6a82bc-ea9e-4536-afe2-d4752be3a907" />
  
- Test email received in your inbox

<img width="676" height="367" alt="Screenshot 2026-09-28 141308" src="https://github.com/user-attachments/assets/23ced7bc-afd7-4382-9997-bc87632f1259" />



**✔ Monitor Alert**

Home → Monitor → Alerts → Alert rules

<img width="941" height="326" alt="Screenshot 2026-09-28 141504" src="https://github.com/user-attachments/assets/7a7ec990-c571-4482-885a-1f7d2b654f68" />

You should see `alert-no-backup-writes` configured to fire when zero write transactions occur in a 24-hour window. If no files have been uploaded yet, the alert may already show as **Fired**. This is expected behavior and confirms the alert is working correctly.

<img width="943" height="257" alt="Screenshot 2026-09-28 141404" src="https://github.com/user-attachments/assets/4d99ff33-eacc-48dc-a2d2-6bd9abbcc09b" />


---


## 📘 What You Learn

| Skill | Why It Matters |
|---|---|
| Deploying Azure infrastructure with Terraform | Makes the backup environment repeatable, versioned, and easy to tear down |
| Blob versioning and soft delete | Protects against accidental overwrites and deletions, the most common causes of data loss |
| Storage lifecycle management | Cuts storage costs by automatically tiering and expiring aging data |
| Geo-redundant storage (RA-GRS) | Keeps backups available even during a regional outage |
| Diagnostic settings and Log Analytics | Provides an audit trail and visibility into storage activity |
| Logic Apps with Managed Identity | Automates workflows without storing credentials |
| Azure Monitor alert rules | Detects silent backup failures before you need to restore |
| Azure RBAC for data-plane access | Explains why control-plane rights alone don't grant access to blob data |


---


## 🔧 Troubleshooting

| Error | Cause | Resolution |
|---|---|---|
| `AuthorizationFailed` during `terraform apply` | Your account lacks the required rights on the subscription or resource group | Confirm you have Contributor (and Owner or User Access Administrator if Terraform creates role assignments) on the subscription |
| `AuthorizationPermissionMismatch` or `403` on `az storage blob upload` | Your user has no data-plane role on the storage account | Complete Step 6.1 and assign **Storage Blob Data Contributor**. Role assignments can take a few minutes to propagate, so wait and retry |
| `StorageAccountAlreadyTaken` | Storage account names are globally unique | Change `yourname` in `terraform.tfvars` to something more unique (3-24 characters, lowercase letters and numbers only) |
| `terraform output` shows `<sensitive>` | The connection string is marked `sensitive = true` | Use `terraform output -raw storage_account_connection_string` |
| Logic App **List blobs (V2)** returns `403` | The Logic App's managed identity has no access to the storage account | Enable the system-assigned identity on the Logic App and grant it a blob data role (for example, **Storage Blob Data Reader**) on the storage account |
| Logic App email action fails with an authentication error | The Office 365 Outlook connection expired or was not authorized | Open the **Send an email (V2)** action, re-sign in to your Outlook account, and save |
| `--include v` returns only one blob | Versioning is not enabled or the overwrite did not run | Verify Versioning is **Enabled** on the storage account, then re-run the upload with `--overwrite` |
| `alert-no-backup-writes` shows **Fired** right after deployment | No write transactions have happened in the 24-hour window yet | Expected behavior. Upload a file and the alert will resolve once writes are detected |


---


## 🏁 Final Notes

This lab is intentionally standalone. No other lab in the series depends on it, so it's safe to tear down as soon as you're done validating it.

```bash
# Full teardown
terraform destroy -auto-approve
```
