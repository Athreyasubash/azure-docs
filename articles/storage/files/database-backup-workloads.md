---
title: Use Azure Files as a Database Backup Target
description: Plan a database backup repository on Azure Files, configure SQL Server backups over SMB, and understand SAP HANA support considerations.
author: Athreyasubash
ms.service: azure-file-storage
ms.topic: concept-article
ms.date: 11/17/2026
ms.author: kendownie
# Customer intent: As a database administrator, I want to evaluate Azure Files as a database backup repository so that I can retain familiar backup workflows and validate recovery requirements.
---

# Use Azure Files as a database backup target

<!-- DRAFT: Azure Files and SQL Server owners must validate the SQL Server configuration and examples before publication. SAP-on-Azure owners must approve the SAP support wording. This draft does not establish support for SAP HANA backups to Azure Files. -->

A database backup target stores backup files created by a database engine or a supported backup application. Azure Files can provide an SMB repository for SQL Server backup workflows. For example, [SQL Server log shipping on Azure virtual machines](/azure/azure-sql/virtual-machines/windows/log-shipping-configure) uses an Azure file share to store transaction-log backups.

This article explains how to plan the repository, configure access, and validate recovery. It also distinguishes SAP HANA file-based backup requirements from the supported SAP uses of Azure Files.

## Understand the scope

These scenarios have different requirements:

| Scenario | What the file share contains | Guidance |
|---|---|---|
| Database backups to Azure Files | Database-generated backup files, such as SQL Server data and transaction-log backups | Use this article and the applicable database documentation. |
| Active SQL Server database storage | Live database files used by a SQL Server instance | Follow the separate [SQL Server failover cluster instance guidance](/azure/azure-sql/virtual-machines/windows/failover-cluster-instance-premium-file-share-manually-configure). |
| Protection of an Azure file share | Recovery points for files already stored in Azure Files | See [Azure Files data protection](files-data-protection-overview.md). |
| SAP shared application storage | Supported SAP application directories and shared filesystems | See [Azure Files SSD NFS and SMB for SAP](/azure/sap/workloads/planning-guide-storage-azure-files). |

This article doesn't describe Azure SQL Database automated backups, SQL Server `BACKUP TO URL` to Azure Blob Storage, or a supported HANA-to-Azure-Files deployment procedure.

> [!IMPORTANT]
> Support for a protocol doesn't establish support for every database configuration. Validate the database version, operating system, deployment topology, authentication method, file service, and recovery workflow together.

## Plan the backup architecture

Keep the backup repository separate from the live database data and log locations. Use a dedicated backup share rather than placing backups alongside the production database files.

```text
Database engine or supported backup application
    |
    | Database-consistent backup files
    v
Dedicated backup file share
    |
    +-- Verification and restore testing
    |
    +-- Separately protected retention copy, as required
```

Decide how the design responds to database-host failure, credential compromise, accidental deletion, and regional failure. A different directory or share alone doesn't provide an independent security or failure boundary.

Before deployment, document:

- The recovery point objective (RPO), which defines acceptable data loss.
- The recovery time objective (RTO), which defines acceptable recovery duration.
- Backup frequency, retention, expected growth, and concurrent jobs.
- The identities and network paths used by both backup and recovery servers.
- How required backup chains, catalogs, and encryption keys are retained.
- Which copy remains recoverable if the production administrator or backup writer is compromised.

## Configure SQL Server backups over SMB

The following workflow applies to SQL Server on Windows using a network backup destination. Review [SQL Server backup devices](/sql/relational-databases/backup-restore/backup-devices-sql-server) and the requirements for your SQL Server deployment before using it.

### Prerequisites

- An Azure SMB file share with sufficient capacity and measured throughput. Start with the [Azure Files planning guide](storage-files-planning.md).
- Network connectivity and working DNS from every SQL Server host that performs backups or restores. Review [Azure Files networking](storage-files-networking-overview.md).
- A supported SMB authentication configuration for the SQL Server service identity. Review [identity-based authentication](storage-files-active-directory-overview.md).
- Share-level permissions and directory/file permissions for that identity. See [assign share-level permissions](storage-files-identity-assign-share-level-permissions.md) and [configure directory and file-level permissions](storage-files-identity-configure-file-level-permissions.md).
- An existing database and database permissions to perform the backup operation.
- For transaction-log backups, an appropriate recovery model and an initialized backup chain.

The SQL Server Database Engine service performs the file access. Access granted only to the person running SQL Server Management Studio isn't sufficient. A drive mapped in that person's interactive session might not be visible to the service.

### Create a backup location

1. Create a dedicated SMB file share and a directory for the database.
1. Configure access for the Database Engine service identity. Configure any SQL Server Agent proxy or copy-job identities separately if those jobs also access the repository.
1. Grant the required read/write access without granting unnecessary administrative permissions.
1. Validate access by running a database backup through SQL Server, not only by browsing the share interactively.

Use the UNC path to the destination:

```text
\\<storage-account>.file.core.windows.net\<backup-share>\<database-directory>
```

### Create database-native backups

The following examples illustrate the path format. Replace the database name and all path placeholders. Use a unique filename for every backup set, and select compression options supported by your SQL Server version and edition.

```sql
BACKUP DATABASE [DatabaseName]
TO DISK =
    N'\\<storage-account>.file.core.windows.net\<backup-share>\<database-directory>\full_<unique-timestamp>.bak'
WITH COMPRESSION, CHECKSUM;
```

For a database with an appropriate recovery model and an initialized backup chain, a transaction-log backup uses the same repository:

```sql
BACKUP LOG [DatabaseName]
TO DISK =
    N'\\<storage-account>.file.core.windows.net\<backup-share>\<database-directory>\log_<unique-timestamp>.trn'
WITH COMPRESSION, CHECKSUM;
```

Schedule full, differential, and transaction-log backups according to the recovery plan. These examples don't configure scheduling, retention, encryption, or failure notification.

A regular full backup establishes a differential base. If you need an additional full backup without changing an existing differential-backup strategy, review [copy-only backups](/sql/relational-databases/backup-restore/copy-only-backups-sql-server).

Retain every backup set needed for the recovery chain. Don't delete files solely because they exceed an age threshold without checking their dependencies. If backups are encrypted, protect and test access to the required certificates and keys separately from the repository.

### Verify and restore

1. Confirm that each backup job completes successfully and record its duration and output size.
1. Use [RESTORE VERIFYONLY](/sql/t-sql/statements/restore-statements-verifyonly-transact-sql) as an additional verification step.
1. Restore the required full, differential, and transaction-log backups to an isolated SQL Server instance. Follow [restore and recovery guidance](/sql/relational-databases/backup-restore/restore-and-recovery-overview-sql-server).
1. Run database consistency checks and application-level validation on the restored database.
1. Measure the complete recovery time, including provisioning, authentication, backup reads, log replay, database checks, and application validation.

> [!IMPORTANT]
> `RESTORE VERIFYONLY` isn't a substitute for restoring the database. A successful scheduled backup also doesn't prove that a recovery objective is met.

For a failover cluster instance, validate backup and restore access after failover to every eligible node. For log shipping, use the [documented Azure Files configuration](/azure/azure-sql/virtual-machines/windows/log-shipping-configure) for backup, copy, and restore jobs.

## SAP HANA file-based backup considerations

SAP HANA file-based backups illustrate a different database backup workflow: HANA writes data and log backups to configured filesystem destinations, and recovery uses the required backup chain and catalog information.

> [!IMPORTANT]
> The [SAP Azure Files planning guide](/azure/sap/workloads/planning-guide-storage-azure-files) states that SAP DBMS workloads aren't supported on Azure Files SSD volumes, NFS, or SMB. Its supported scenarios don't establish support for a HANA backup repository on Azure Files. Don't interpret this section, NFS connectivity, or support for `/hana/shared` as approval to use Azure Files for HANA backups.

For a documented Azure NFS storage design, see [SAP HANA configuration with Azure NetApp Files](/azure/sap/workloads/hana-vm-operations-netapp), which includes backup and log-backup volumes. Azure NetApp Files is a different service; its HANA guidance doesn't establish Azure Files support.

### Plan a HANA file-based backup workflow

For a storage configuration approved for the HANA deployment:

1. Identify the system database and tenant databases covered by the recovery plan.
1. Define separate destinations for data backups and log backups, and identify the catalog and recovery metadata needed to locate and use them.
1. Validate filesystem permissions and access from every participating HANA host, using the documented mount configuration for the selected storage service.
1. Configure and monitor backups through supported HANA administration tools.
1. Manage retention using the database's backup-chain and catalog requirements rather than independent age-based file deletion.
1. Restore into an isolated recovery environment and verify the intended recovery point, required credentials and keys, and application behavior.

These are planning considerations, not instructions to mount Azure Files as a HANA backup destination. Confirm the configuration-specific support position with the SAP-on-Azure and Azure Files product teams before implementing that design.

HANA Backint protection is a separate architecture. For example, [Azure Backup for SAP HANA](/azure/backup/tutorial-backup-sap-hana-db) integrates with Backint; it isn't a copy job for a filesystem backup directory.

## Size for both backup and recovery

Full backups often generate sustained writes. Log backups can generate frequent smaller writes. Verification, copy jobs, and restores generate reads. Measure representative concurrency rather than sizing only for one database.

Estimate repository capacity from retained full backups, differential or incremental backups where applicable, required log chains, and operational headroom. Account separately for share snapshots or secondary copies. Don't assume backup compression or deduplication ratios without measurements.

Choose the media tier and billing model based on:

- Supported protocol and configuration.
- Measured backup and restore throughput.
- Concurrent database and administrative operations.
- Required capacity, IOPS, and throughput.
- Provisioning, transactions, networking, retention, and secondary-protection costs.

Review [Azure Files scalability targets](storage-files-scale-targets.md), [performance](understand-performance.md), and [billing](understanding-billing.md). Don't assume SSD is required for every backup repository or that HDD meets every recovery objective.

## Protect the repository

Use network restrictions and least-privilege access. Where appropriate, use a [private endpoint](storage-files-networking-endpoints.md), and validate DNS from the recovery environment.

Protect completed backup artifacts using features supported for the specific share. Review [Azure Files data protection](files-data-protection-overview.md) and the current [Azure Backup support matrix](/azure/backup/azure-file-share-support-matrix) before choosing a protection method.

Share snapshots and redundancy don't automatically provide application consistency, immutability, or an isolated backup. A snapshot of incomplete backup files might not contain a usable recovery chain. Coordinate repository protection with backup completion, and test recovery from the protected copy.

Don't treat synchronization alone as an independent backup: deletion or corruption can propagate. If an independently protected or immutable copy is required, validate that the selected service, policy, permissions, and retention controls actually meet that requirement.

## Monitor and test

Alert on failed or missing jobs, unexpected backup age, capacity pressure, permission changes, and throughput problems. Monitor backup duration, size, and the most recent usable recovery point.

Repeat restore exercises after changes to identities, network controls, database versions, failover configuration, or repository protection. Retain the results as evidence that the recovery design works.

## Next steps

- [Plan an Azure Files deployment](storage-files-planning.md).
- [Configure SQL Server log shipping with Azure Files](/azure/azure-sql/virtual-machines/windows/log-shipping-configure).
- [Review SAP storage guidance](/azure/sap/workloads/planning-guide-storage-azure-files).
- [Protect data stored in Azure Files](files-data-protection-overview.md).
