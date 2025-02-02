---
description: "Manage accelerated database recovery"
title: "Manage accelerated database recovery | Microsoft Docs"
ms.date: "02/02/2021"
ms.prod: sql
ms.prod_service: backup-restore
ms.technology: backup-restore
ms.topic: conceptual
helpviewer_keywords: 
  - "accelerated database recovery [SQL Server], recovery-only"
  - "database recovery [SQL Server]"
author: mashamsft
ms.author: mathoma
ms.reviewer: kfarlee
monikerRange: ">=sql-server-ver15"
---
# Manage accelerated database recovery

[!INCLUDE[sqlserver](../includes/applies-to-version/sqlserver2019.md)]

## Enabling and controlling ADR

ADR is off by default in [!INCLUDE[sql-server-2019](../includes/sssql19-md.md)], and can be controlled using DDL syntax:
```syntaxsql
ALTER DATABASE [DB] SET ACCELERATED_DATABASE_RECOVERY = {ON | OFF}
[(PERSISTENT_VERSION_STORE_FILEGROUP = { filegroup name }) ];

```

Use this syntax to control whether the feature is on or off, and designate a specific filegroup for the *persistent version store* (PVS) data. If no filegroup is specified, the PVS will be stored in the PRIMARY filegroup.

Note that SQL will take an exclusive lock on the database to change this state.  That means that the ALTER DATABASE command will stall until all active sessions are gone, and that any new sessions will wait behind the ALTER DATABASE command.  If it is important to complete the operation and remove the lock, you can use the termination clause, WITH ROLLBACK [IMMEDIATE | AFTER {number} SECONDS | NO_WAIT] to abort any active sessions in the database.

## Managing the persistent version store Filegroup
The ADR feature is based on having changes versioned, with different versions of a data element kept in the PVS.
There are considerations to locating where the PVS is located and in how to manage the size of the data in the PVS.

### To enable ADR without specifying a filegroup

```sql
ALTER DATABASE [MyDatabase] SET ACCELERATED_DATABASE_RECOVERY = ON;
GO
```

In this case, when the PVS filegroup is not specified, the `PRIMARY` filegroup holds the PVS data.

### To enable ADR and specify that the PVS should be stored in the [VersionStoreFG] filegroup

Before running this script, create the filegroup.

```sql
ALTER DATABASE [MyDatabase] SET ACCELERATED_DATABASE_RECOVERY = ON
(PERSISTENT_VERSION_STORE_FILEGROUP = [VersionStoreFG])
```

### To disable the ADR feature

```sql
ALTER DATABASE [MyDatabase] SET ACCELERATED_DATABASE_RECOVERY = OFF;
GO
```

Even after the ADR feature is disabled, there will be versions stored in the persistent version store that are still needed by the system for logical revert.

### Change the location of the PVS to a different filegroup

You may need to move the location of the PVS to a different filegroup for a variety of reasons. For example, PVS may require more space, or faster storage.

Changing the location of the PVS is a three-step process.

1. Turn the ADR feature off.

   ```sql
   ALTER DATABASE [MyDatabase] SET ACCELERATED_DATABASE_RECOVERY = OFF;
   GO
   ```

2. Wait until all of the versions stored in the PVS can be freed

   In order to be able to turn on ADR with a new location for the persistent version store, you must first make sure that all of the version information has been purged from the previous PVS location. In order to force that cleanup to happen, run the command:

   ```sql
   EXEC sys.sp_persistent_version_cleanup [database name]
   ```

   `sys.sp_persistent_version_cleanup` stored procedure is synchronous, meaning that it will not complete until all version information is cleaned up from the current PVS.  Once it completes, you can verify that the version information is indeed removed by querying the DMV `sys.dm_persistent_version_store_stats` and examining the value of `persistent_version_store_size_kb`.

   ```sql
   SELECT DB_Name(database_id), persistent_version_store_size_kb 
   FROM sys.dm_tran_persistent_version_store_stats where database_id = [MyDatabaseID]
   ```

   When the value of persistent_version_store_size_kb is 0, you can re-enable the ADR feature, configuring the PVS to be located in the new filegroup.

1. Turn on ADR specifying the new location for PVS

   ```sql
   ALTER DATABASE [MyDatabase] SET ACCELERATED_DATABASE_RECOVERY = ON
   (PERSISTENT_VERSION_STORE_FILEGROUP = [VersionStoreFG])
   ```

## Troubleshooting

> [!NOTE]
> This section also applies to Azure SQL Database.

Query `sys.dm_tran_persistent_version_store_stats` to check PVS sizes.

Check `% of DB` size. Also note the difference from typical size.

PVS is considered large if it's significantly larger than baseline or if it is close to 50% of the size of the database. 

1. Retrieve `oldest_active_transaction_id` and check whether this transaction has been active for a really long time by querying `sys.dm_tran_database_transactions` based on the transaction ID.

   Active transactions prevent cleaning up PVS.

2. If the database is part of an availability group, check the `secondary_low_water_mark`. This is the same as the `low_water_mark_for_ghosts` reported by `sys.dm_hadr_database_replica_states`. Query `sys.dm_hadr_database_replica_states` to see whether one of the replicas is holding this value behind, since this will also prevent PVS cleanup.
3. Check `min_transaction_timestamp` (or `online_index_min_transaction_timestamp` if the online PVS is holding up) and based on that check `sys.dm_tran_active_snapshot_database_transactions` for the column `transaction_sequence_num` to find the session that has the old snapshot transaction holding up PVS cleanup.
4. If none of the above applies, then it means that the cleanup is held by aborted transactions. Check the last time the `aborted_version_cleaner_last_start_time`  and `aborted_version_cleaner_last_end_time` to see if the aborted transaction cleanup has completed. The `oldest_aborted_transaction_id` should be moving higher after the aborted transaction cleanup completes.
5. If the aborted transaction hasn’t completed successfully recently, check the error log for messages reporting `VersionCleaner` issues.

Use the sample query below as a troubleshooting aid:

```sql
SELECT pvss.persistent_version_store_size_kb / 1024. / 1024 AS persistent_version_store_size_gb,
       pvss.online_index_version_store_size_kb / 1024. / 1024 AS online_index_version_store_size_gb,
       pvss.current_aborted_transaction_count,
       pvss.aborted_version_cleaner_start_time,
       pvss.aborted_version_cleaner_end_time,
       dt.database_transaction_begin_time AS oldest_transaction_begin_time,
       asdt.session_id AS active_transaction_session_id,
       asdt.elapsed_time_seconds AS active_transaction_elapsed_time_seconds
FROM sys.dm_tran_persistent_version_store_stats AS pvss
LEFT JOIN sys.dm_tran_database_transactions AS dt
ON pvss.oldest_active_transaction_id = dt.transaction_id
   AND
   pvss.database_id = dt.database_id
LEFT JOIN sys.dm_tran_active_snapshot_database_transactions AS asdt
ON pvss.min_transaction_timestamp = asdt.transaction_sequence_num
   OR
   pvss.online_index_min_transaction_timestamp = asdt.transaction_sequence_num
WHERE pvss.database_id = DB_ID();
```
