# Windows Server Backup & Restore Lab

[![GitHub](https://img.shields.io/badge/GitHub-toannguyenitoz-181717?style=for-the-badge&logo=github)](https://github.com/toannguyenitoz)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Toan%20Nguyen-0A66C2?style=for-the-badge&logo=linkedin)](https://www.linkedin.com/in/toan-nguyen-it-oz/)
[![Windows Server](https://img.shields.io/badge/Windows_Server-Backup_%26_Restore-0078D6?style=for-the-badge&logo=windows&logoColor=white)](https://github.com/toannguyenitoz/windows-server-backup-restore-lab)
[![PowerShell](https://img.shields.io/badge/PowerShell-Automation-5391FE?style=for-the-badge&logo=powershell&logoColor=white)](https://github.com/toannguyenitoz/windows-server-backup-restore-lab)

A hands-on Windows Server lab demonstrating how to configure, perform, verify, and restore backups using **Windows Server Backup (WSB)**.

The lab simulates a real-world scenario where an important file is accidentally deleted from a Windows Server and must be recovered from a backup.

---

## 📚 Table of Contents

- [🎯 Lab Objective](#-lab-objective)
- [🧩 Scenario](#-scenario)
- [🖥️ Lab Environment](#%EF%B8%8F-lab-environment)
- [Step 1 — Install Windows Server Backup](#1-install-windows-server-backup)
- [Step 2 — Create Test Data](#2-create-test-data)
- [Step 3 — Configure the Backup](#3-configure-the-backup)
- [Step 4 — Configure the Backup Schedule](#4-configure-the-backup-schedule)
- [Step 5 — Select Backup Destination](#5-select-backup-destination)
- [Step 6 — Run and Verify the Backup](#6-run-and-verify-the-backup)
- [Step 7 — Simulate Data Loss](#7-simulate-data-loss)
- [Step 8 — Recover the Deleted File](#8-recover-the-deleted-file)
- [Step 9 — Verify the Recovery](#9-verify-the-recovery)
- [🔐 Backup vs Recovery](#-backup-vs-recovery)
- [📊 RPO and RTO](#-rpo-and-rto)
- [🧪 Recovery Test Checklist](#-recovery-test-checklist)
- [🌎 Real-World Considerations](#-real-world-considerations)
- [🛡️ 3-2-1 Backup Principle](#%EF%B8%8F-3-2-1-backup-principle)
- [📚 What I Learned](#-what-i-learned)
- [🚀 Future Improvements](#-future-improvements)

---

## 🎯 Lab Objective

The goal of this lab is to understand the complete backup and recovery lifecycle:

```
Backup → Verify → Simulate Failure → Recover → Verify Again
```

This is not simply a backup configuration exercise.

> [!IMPORTANT]
> A successful backup is only useful if the data can actually be restored when required.

---

## 🧩 Scenario

Imagine an employee accidentally deletes an important file from a Windows file server.

**Example:**

```text
D:\Data\Backup-Test.txt
```

The file is no longer available to the user.

The administrator must determine:

- Is a valid backup available?
- Which backup contains the required file?
- Can the file be restored?
- Was the restored file recovered successfully?
- How long does the recovery process take?

This lab simulates that scenario in a controlled environment.

---

## 🖥️ Lab Environment

| Component | Configuration |
| :--- | :--- |
| **Operating System** | Windows Server 2019 / 2022 |
| **Backup Tool** | Windows Server Backup (WSB) |
| **Backup Source** | `D:\Data` / Full Server |
| **Backup Destination** | Dedicated Volume / Backup Disk |
| **Recovery Type** | Files and Folders |
| **Test Data** | Documents and text files |

> [!TIP]
> Use a dedicated virtual disk or separate backup storage for the lab. Keep backup storage **physically separate** from the production data volume to prevent a single disk failure from affecting both.

---

## 1. Install Windows Server Backup

### GUI Method

Open:

**Server Manager → Manage → Add Roles and Features**

Select:

**Role-based or feature-based installation → Features → Windows Server Backup**

### PowerShell Method

```powershell
# Install Windows Server Backup feature
Install-WindowsFeature -Name Windows-Server-Backup -IncludeManagementTools

# Verify installation
Get-WindowsFeature -Name Windows-Server-Backup
```

After installation, launch the console:

**Server Manager → Tools → Windows Server Backup → Local Backup**

> [!NOTE]
> Windows Server Backup is an integrated Windows Server feature that provides backup and recovery without requiring a third-party platform. For enterprise-scale environments, consider Microsoft Azure Backup Server (MABS) or third-party solutions (Veeam, Acronis).

---

## 2. Create Test Data

Create a test directory and sample files to use in the backup and recovery demonstration.

```powershell
# Create the data directory
New-Item -Path "D:\Data" -ItemType Directory -Force

# Create sample files representing real-world content types
"Backup verification test - $(Get-Date)" | Out-File "D:\Data\Backup-Test.txt"
"Network configuration information"       | Out-File "D:\Data\Network.txt"
"Employee records"                        | Out-File "D:\Data\Employee.txt"
"Server configuration baseline"           | Out-File "D:\Data\Configuration.txt"

# Verify the directory and files
Get-ChildItem "D:\Data" | Format-Table Name, Length, LastWriteTime
```

**Expected result:**

```text
D:\Data
├── Backup-Test.txt
├── Configuration.txt
├── Employee.txt
└── Network.txt
```

---

## 3. Configure the Backup

Open:

**Windows Server Backup → Local Backup → Backup Schedule**

Under **Select Backup Configuration**, choose:

### Option A — Full Server *(Recommended for disaster recovery)*

Protects the entire server including System State, all volumes, and all applications. Enables **Bare Metal Recovery (BMR)** in a full-server failure scenario.

### Option B — Custom *(Volume or folder-specific)*

Use when only specific volumes or folders need to be protected. Reduces backup storage requirements but limits recovery scope.

> [!TIP]
> For most production environments, **Full Server** backup is preferred. For targeted data protection on large servers, combine **Full Server** (weekly) with **Custom/Volume** (daily) backups.

---

## 4. Configure the Backup Schedule

Select the required backup frequency.

Windows Server Backup supports:

- **Once a day** — Suitable for low-change environments
- **Multiple times per day** — Required when data changes frequently and RPO is tight

**Frequency selection decision flow:**

```text
Critical / high-change data
          ↓
Higher backup frequency required
          ↓
Lower potential data loss (lower RPO)
          ↓
Higher storage consumption
```

> The correct frequency is determined by **RPO (Recovery Point Objective)** — see [📊 RPO and RTO](#-rpo-and-rto) below.

---

## 5. Select Backup Destination

Windows Server Backup supports the following destination types:

| Destination | Use Case | Notes |
| :--- | :--- | :--- |
| **Dedicated Backup Disk** | Separate physical/virtual disk | Recommended — isolated from production |
| **Backup Volume** | Specific volume on any accessible disk | Ensure it is not the source volume |
| **Remote Shared Folder** | UNC network path | Only retains the most recent backup |

**Recommended disk layout for this lab:**

```text
C: → Windows Server OS (System Volume)
D: → Application / Data (Source Volume)
E: → Backup Destination (Dedicated Backup Volume)
```

> [!WARNING]
> Never store backups on the same physical disk as the source data. A single disk failure will destroy both the original data and the backup simultaneously.

---

## 6. Run and Verify the Backup

After completing the configuration, start the backup.

### Monitor via GUI

**Windows Server Backup → Local Backup**

View backup status, start time, and completion status.

### Monitor via PowerShell

```powershell
# Check backup status
$status = Get-WBSummary
$status | Select-Object LastSuccessfulBackupTime, LastBackupResultHR, NumberOfVersions

# List all backup versions available
$policy = Get-WBPolicy
Get-WBBackupTarget -Policy $policy

# View backup job details
Get-WBJob -Previous 5 | Format-Table JobType, StartTime, EndTime, JobState, HResult
```

> [!CAUTION]
> **Backup completed successfully ≠ Recovery guaranteed.**
> A backup status of "Success" only confirms that data was written to the backup destination. It does **not** confirm that the data can be successfully restored. Always test recovery separately.

---

## 7. Simulate Data Loss

Simulate a real-world accidental deletion:

```powershell
# Delete the test file
Remove-Item "D:\Data\Backup-Test.txt" -Force

# Confirm the file no longer exists
Test-Path "D:\Data\Backup-Test.txt"
```

**Expected result:**

```text
False
```

```powershell
# Verify the directory shows the missing file
Get-ChildItem "D:\Data"
```

At this point, `Backup-Test.txt` no longer exists on disk.

---

## 8. Recover the Deleted File

### GUI Recovery Wizard

Open:

**Windows Server Backup → Local Backup → Recover**

1. **Select recovery source** — *This server* or *Another location*
2. **Select backup date** — Choose the backup taken before the deletion
3. **Select recovery type** — *Files and folders*
4. **Browse to** `D:\Data` → Select `Backup-Test.txt`
5. **Recovery destination** — Original location or alternate path
6. Complete the recovery wizard

### PowerShell Recovery (Advanced)

```powershell
# Get available backup versions
$backupVersions = Get-WBBackupVolumeBrowsePath -BackupTarget (Get-WBPolicy | Get-WBBackupTarget)

# Start a recovery session to the latest backup
$recoverySource = Get-WBBackupSet | Select-Object -Last 1

# Recover specific file
Start-WBFileRecovery -BackupSet $recoverySource `
                     -SourcePath "D:\Data\Backup-Test.txt" `
                     -TargetPath "D:\Data\Backup-Test.txt" `
                     -Overwrite $true
```

---

## 9. Verify the Recovery

```powershell
# 1. Confirm file exists
Test-Path "D:\Data\Backup-Test.txt"

# 2. Read file content
Get-Content "D:\Data\Backup-Test.txt"

# 3. Check file properties (size, timestamps, attributes)
Get-Item "D:\Data\Backup-Test.txt" | Select-Object Name, Length, CreationTime, LastWriteTime, Attributes

# 4. Verify NTFS permissions are intact
Get-Acl "D:\Data\Backup-Test.txt" | Format-List
```

**Expected verification results:**

| Check | Expected Result |
| :--- | :--- |
| `Test-Path` | `True` |
| File content | Original content readable |
| File size | Matches original |
| Permissions | ACLs intact |

**Complete recovery workflow:**

```text
CREATE DATA
     ↓
BACKUP (scheduled or manual)
     ↓
VERIFY BACKUP (status + test restore)
     ↓
SIMULATE FAILURE (delete file)
     ↓
INITIATE RECOVERY
     ↓
VERIFY RECOVERY (content, size, permissions)
     ↓
DOCUMENT OUTCOME (RTO achieved? Data integrity confirmed?)
```

---

## 🔐 Backup vs Recovery

One of the most important lessons from this lab:

> [!IMPORTANT]
> **Having a backup is not enough.**

A reliable backup strategy requires all three components:

```text
Backup
  +
Verification (automated status check)
  +
Recovery Testing (actual restore test)
  =
Reliable Recovery Capability
```

A backup that has **never been tested** should not be considered a reliable recovery solution — regardless of what the status dashboard shows.

---

## 📊 RPO and RTO

Backup planning should be driven by two business-defined metrics:

### RPO — Recovery Point Objective

**How much data can the business afford to lose?**

```text
RPO = 1 hour
→ Backup strategy must allow recovery to a point
  no more than ~1 hour before the incident.
→ Requires hourly or sub-hourly backup frequency.
```

### RTO — Recovery Time Objective

**How quickly must the service or data be restored?**

```text
RTO = 2 hours
→ The recovery solution must be capable of restoring
  the required service within approximately 2 hours.
→ Drives decisions on backup type, location, and tooling.
```

| Metric | Question | Drives Decision On |
| :--- | :--- | :--- |
| **RPO** | How much data loss is acceptable? | Backup frequency, backup type |
| **RTO** | How fast must recovery complete? | Backup location, recovery method, tooling |

> [!NOTE]
> In practice, RPO and RTO are defined by the **business**, not IT. The Systems Administrator's role is to design and implement a backup strategy that meets those targets — and regularly test that it actually does.

---

## 🧪 Recovery Test Checklist

Use this checklist each time you repeat or extend the lab:

- [ ] Windows Server Backup feature installed and confirmed
- [ ] Test data created and verified (`Get-ChildItem`)
- [ ] Backup destination configured (separate volume)
- [ ] Backup policy created (schedule, source, destination)
- [ ] Backup completed successfully (`Get-WBSummary`)
- [ ] Backup status verified (not just "Success" — check log)
- [ ] Test file deleted and confirmed missing (`Test-Path`)
- [ ] Recovery wizard initiated
- [ ] Correct backup date/version selected
- [ ] Target file identified and selected
- [ ] Recovery completed without errors
- [ ] File content verified (`Get-Content`)
- [ ] File permissions verified (`Get-Acl`)
- [ ] Recovery time recorded (RTO measurement)
- [ ] Outcome documented

---

## 🌎 Real-World Considerations

In a production environment, backup planning extends well beyond configuring Windows Server Backup.

### What needs to be protected?

| Component | Recovery Method | Notes |
| :--- | :--- | :--- |
| **File data** | File/folder recovery | Covered in this lab |
| **System State** | System State Backup | Includes registry, boot files |
| **Active Directory** | AD-aware backup (authoritative restore) | Requires careful handling of tombstone lifetime |
| **Application data** | Application-consistent backup (VSS) | SQL, Exchange, etc. require VSS-aware tools |
| **Full server** | Bare Metal Recovery (BMR) | For complete server failure scenarios |

### Where are backups stored?

Avoid single-location backup storage:

```text
Production Data
     │
     ├── Local Backup (fast restore, single-site risk)
     │
     ├── Secondary On-site Backup (separate rack / NAS)
     │
     └── Off-site / Cloud Backup (disaster recovery, ransomware protection)
```

### How often are backups tested?

| Test Type | Recommended Frequency |
| :--- | :--- |
| Backup status review | Daily (automated monitoring) |
| File/folder recovery test | Monthly |
| Full server recovery test | Quarterly or after major changes |
| Disaster recovery exercise | Annually |

### What happens if the entire server fails?

File-level recovery and **Bare Metal Recovery (BMR)** are fundamentally different procedures with different backup requirements. Ensure the backup strategy explicitly covers both scenarios.

---

## 🛡️ 3-2-1 Backup Principle

A widely adopted backup strategy guideline:

```text
3   copies of data
2   different types of storage media
1   copy stored off-site (or in the cloud)
```

**Example implementation:**

```text
Production Data
      │
      ├── 1. Local Windows Server Backup (on-site, fast restore)
      │
      ├── 2. NAS or secondary server backup (on-site, different media)
      │
      └── 3. Azure Backup / cloud storage (off-site, ransomware protection)
```

> [!NOTE]
> The 3-2-1 principle is a **starting point**, not a rigid rule. The actual architecture should be adapted to the organisation's security, compliance, availability, and budget requirements.

---

## 📚 What I Learned

This lab reinforced several practical Windows Server administration skills:

**Technical skills:**
- Windows Server Backup installation and configuration
- Backup scheduling, storage planning, and destination selection
- File and folder recovery using both GUI and PowerShell
- Backup status verification (`Get-WBSummary`, `Get-WBJob`)
- NTFS permission verification post-recovery
- PowerShell-based automation for backup and verification tasks

**Operational thinking:**
- Understanding RPO and RTO as business-driven metrics
- Distinguishing between backup completion and recovery capability
- Designing layered backup storage (on-site + off-site)
- Structuring regular recovery testing as an operational discipline
- Documenting outcomes for audit and compliance purposes

> [!IMPORTANT]
> The most important lesson from this lab:
>
> **Don't just configure systems. Make sure you can recover them when something goes wrong.**

---

## 🚀 Future Improvements

Planned extensions for this lab:

- [ ] System State Backup and restore
- [ ] Active Directory authoritative restore walkthrough
- [ ] Bare Metal Recovery (BMR) exercise
- [ ] Backup to a network share (UNC path)
- [ ] PowerShell-based backup automation script
- [ ] Scheduled recovery testing procedure
- [ ] Backup monitoring and alerting setup
- [ ] Backup failure simulation and alerting response
- [ ] Disk failure simulation using virtual disk removal
- [ ] Azure Backup integration (cloud off-site copy)
- [ ] VSS (Volume Shadow Copy) deep-dive
- [ ] Disaster Recovery documentation template

---

## 👨‍💻 Author

**Toan Xuan Nguyen**

*IT Support L2 (Work Placement) @ DXC Technology | Systems Administration | Microsoft Technologies*
📍 Adelaide, South Australia

Currently developing hands-on skills in:

**Windows Server • Active Directory • Microsoft 365 • Intune • Azure • Networking • Systems Administration**

- 💼 **LinkedIn:** [Toan Nguyen](https://www.linkedin.com/in/toan-nguyen-it-oz/)
- 🐙 **GitHub:** [@toannguyenitoz](https://github.com/toannguyenitoz)

`#ToanNguyenITOZ` `#WindowsServer` `#BackupAndRestore` `#SystemsAdministrator` `#ITSupport` `#PowerShell` `#DisasterRecovery`

---

## ⭐ Key Takeaway

A Systems Administrator's responsibility does not end when a system is successfully deployed.

The real test comes when something fails.

```
Backup → Verify → Recover → Verify Again
```

**That is the skill this lab is designed to practice.**

---

© 2026 Toan Nguyen. Released under the [MIT License](LICENSE).
