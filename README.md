# 📘 SQL Server AAG Playbook

A practical, production-tested guide for managing SQL Server Always On Availability Groups (AAG) — from initial setup to daily operations and troubleshooting.

> All server names, IPs, and database names in this guide use dummy data.

---

## 📁 Contents

| Section | Description |
|---|---|
| [01 - Overview](#01---aag-overview) | What is AAG and when to use it |
| [02 - Monitoring](#02---daily-monitoring) | Daily health check queries |
| [03 - Troubleshooting](#03---troubleshooting) | Common issues and solutions |
| [04 - Credential Rotation](#04---credential-rotation) | SOP for rotating service account passwords |
| [05 - New Node Setup](#05---adding-a-new-node) | Steps to add a new VM to existing AAG |

---

## 01 - AAG Overview

Always On Availability Groups (AAG) is a SQL Server high availability solution that provides:

- **Automatic failover** between primary and secondary replicas
- **Data redundancy** via synchronous or asynchronous replication
- **Readable secondaries** for reporting workloads
- **Multi-subnet support** via Distributed AG (DAG)

### Typical 3-Node Setup

Primary (Sync)     ──►  Secondary-1 (Sync, Auto Failover)
──►  Secondary-2 (Async, Manual Failover)

---

## 02 - Daily Monitoring

### Check AAG Health Status

```sql
SELECT
    ag.name AS ag_name,
    ar.replica_server_name,
    ars.role_desc,
    ars.synchronization_health_desc,
    drs.synchronization_state_desc,
    drs.log_send_queue_size,
    drs.redo_queue_size
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
JOIN sys.dm_hadr_database_replica_states drs ON ar.replica_id = drs.replica_id
ORDER BY ag.name, ars.role_desc
```

### Check Primary Replica per AG

```sql
SELECT
    ag.name AS ag_name,
    ar.replica_server_name,
    ars.role_desc
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
WHERE ars.role_desc = 'PRIMARY'
```

### Check Database Sync State

```sql
SELECT
    db.name AS database_name,
    ar.replica_server_name,
    drs.synchronization_state_desc,
    drs.synchronization_health_desc,
    drs.log_send_queue_size,
    drs.redo_queue_size
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
JOIN sys.databases db ON drs.database_id = db.database_id
ORDER BY db.name
```

---

## 03 - Troubleshooting

### Issue: Replica showing SYNCHRONIZING (stuck)

**Symptoms:** Secondary replica stays in SYNCHRONIZING state and never reaches SYNCHRONIZED.

**Check log send & redo queue:**
```sql
SELECT
    ar.replica_server_name,
    drs.log_send_queue_size,
    drs.log_send_rate,
    drs.redo_queue_size,
    drs.redo_rate
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
```

**Common causes:**
- High disk latency on secondary
- Network bandwidth insufficient
- Large transaction log backups blocking redo thread

---

### Issue: AAG Listener not responding

**Check listener status:**
```sql
SELECT
    dns_name,
    port,
    state_desc,
    ip_configuration_string_from_cluster
FROM sys.availability_group_listeners
```

---

## 04 - Credential Rotation

SOP for rotating Windows domain service account passwords used by SQL Server integrations.

### Scenario
A domain account (e.g. `DOMAIN\svc_sqlservice`) is used as a credential for SQL Server integrations. When the password changes in Active Directory, the new password must be updated in all dependent systems.

### Step 1 — Identify where the credential is used

```sql
-- Check SQL Server login
SELECT name, type_desc, is_disabled
FROM sys.server_principals
WHERE name LIKE '%svc_sqlservice%'

-- Check SQL Agent Proxy
SELECT p.name AS proxy_name, c.credential_identity
FROM msdb.dbo.sysproxies p
JOIN sys.credentials c ON p.credential_id = c.credential_id
WHERE c.credential_identity LIKE '%svc_sqlservice%'

-- Check Windows Credential
SELECT name, credential_identity
FROM sys.credentials
WHERE credential_identity LIKE '%svc_sqlservice%'
```

### Step 2 — Update password in application config table

```sql
-- Verify current value
SELECT ConfigID, StringValue AS CurrentPassword
FROM AppDB.dbo.SysConfig
WHERE ConfigID = 1

-- Update on PRIMARY replica only (auto-syncs to secondary)
BEGIN TRAN
UPDATE AppDB.dbo.SysConfig
SET StringValue = 'NewPassword123'
WHERE ConfigID = 1

-- Verify before commit
SELECT ConfigID, StringValue FROM AppDB.dbo.SysConfig
WHERE ConfigID = 1
COMMIT TRAN
```

> ⚠️ **Note:** If the database is NOT part of the Distributed AG, run this update separately on each AG's primary replica.

### Step 3 — Sanity Check

```sql
-- Trigger test job to validate connectivity
EXEC msdb.dbo.sp_start_job @job_name = 'Integration_Test_Job'

-- Check job result
SELECT TOP 1
    j.name,
    jh.run_status,
    jh.run_date,
    jh.run_time,
    jh.message
FROM msdb.dbo.sysjobs j
JOIN msdb.dbo.sysjobhistory jh ON j.job_id = jh.job_id
WHERE j.name = 'Integration_Test_Job'
ORDER BY jh.run_date DESC, jh.run_time DESC
```

---

## 05 - Adding a New Node

Steps to prepare a new VM and join it to an existing AAG as a new replica.

### Prerequisites
- [ ] Windows Server installed on new VM
- [ ] SQL Server same version as existing nodes
- [ ] VM accessible from existing nodes (ping test)
- [ ] Firewall rules opened (port 1433, 5022)

### Step 1 — Verify network connectivity from new node

```powershell
# Test connectivity to existing nodes
Test-NetConnection -ComputerName "node1-server" -Port 1433
Test-NetConnection -ComputerName "node1-server" -Port 5022
```

### Step 2 — Check existing AAG configuration

```sql
-- Run on existing primary
SELECT
    ag.name,
    ar.replica_server_name,
    ar.availability_mode_desc,
    ar.failover_mode_desc,
    ars.role_desc
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
JOIN sys.dm_hadr_availability_replica_states ars ON ar.replica_id = ars.replica_id
```

### Step 3 — Add new replica to AG

```sql
-- Run on primary
ALTER AVAILABILITY GROUP [AG_NAME]
ADD REPLICA ON 'NEW-NODE-SERVER'
WITH (
    ENDPOINT_URL = 'TCP://NEW-NODE-SERVER:5022',
    AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
    FAILOVER_MODE = MANUAL,
    SEEDING_MODE = AUTOMATIC
)
```

---

## 🛠️ Environment

- SQL Server 2019 / 2022
- Windows Server Failover Cluster (WSFC)
- 3-node AAG (2 sync + 1 async)
- Distributed AG for multi-site setup

---

## 👤 Author

**Rika Afriyani** — Junior DBA  
💼 [LinkedIn](https://linkedin.com/in/rika-afriyani-b86457191) | 🐙 [GitHub](https://github.com/Rikajo90)
