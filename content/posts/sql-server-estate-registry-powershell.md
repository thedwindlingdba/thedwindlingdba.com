---
title: "Build a SQL Server Estate Registry: Inventory 25K Databases with PowerShell"
description: "A runnable PowerShell + dbatools estate registry: schema, daily discovery sweeps, drift detection, and the review workflow that keeps 25,000 databases enumerated."
date: 2026-10-12
lastmod: 2026-10-12
author: "Dustin Marzolf"
canonicalURL: "https://thedwindlingdba.com/posts/sql-server-estate-registry-powershell/"
tags: ["sql server inventory script powershell", "sql server estate registry", "database inventory automation", "dbatools", "powershell"]
categories: ["sql-server-fleet-management"]
ShowToc: true
TocOpen: true
jsonld: |
  {"@context":"https://schema.org","@graph":[{"@type":"BlogPosting","@id":"https://thedwindlingdba.com/posts/sql-server-estate-registry-powershell#article","headline":"Build a SQL Server Estate Registry: Inventory 25K Databases with PowerShell","description":"A runnable PowerShell + dbatools estate registry: schema, daily discovery sweeps, drift detection, and the review workflow that keeps 25,000 databases enumerated.","datePublished":"2026-10-12","dateModified":"2026-10-12","author":{"@id":"https://thedwindlingdba.com/author/dustin-marzolf#person"},"publisher":{"@id":"https://thedwindlingdba.com#organization"},"image":{"@id":"https://thedwindlingdba.com/posts/sql-server-estate-registry-powershell#primaryimage"},"mainEntityOfPage":{"@type":"WebPage","@id":"https://thedwindlingdba.com/posts/sql-server-estate-registry-powershell"},"wordCount":1717,"articleBody":"# Build a SQL Server Estate Registry: Inventory 25K Databases with PowerShell Every estate-management failure I have watched starts the same way. Somebody asks how many databases we have, and the room","keywords":["sql server inventory script powershell","sql server estate registry","database inventory automation","dbatools","powershell"]},{"@type":"Person","@id":"https://thedwindlingdba.com/author/dustin-marzolf#person","name":"Dustin Marzolf","jobTitle":"Senior Database Administrator","url":"https://thedwindlingdba.com/author/dustin-marzolf"},{"@type":"Organization","@id":"https://thedwindlingdba.com#organization","name":"The Dwindling DBA","url":"https://thedwindlingdba.com","logo":{"@type":"ImageObject","url":"https://thedwindlingdba.com/logo.png"}},{"@type":"BreadcrumbList","@id":"https://thedwindlingdba.com/posts/sql-server-estate-registry-powershell#breadcrumb","itemListElement":[{"@type":"ListItem","position":1,"name":"Home","item":"https://thedwindlingdba.com"},{"@type":"ListItem","position":2,"name":"SQL Server Fleet Management","item":"https://thedwindlingdba.com/categories/sql-server-fleet-management/"},{"@type":"ListItem","position":3,"name":"Build a SQL Server Estate Registry","item":"https://thedwindlingdba.com/posts/sql-server-estate-registry-powershell"}]},{"@type":"FAQPage","@id":"https://thedwindlingdba.com/posts/sql-server-estate-registry-powershell#faq","mainEntity":[{"@type":"Question","name":"How do I inventory all my SQL Servers with PowerShell?","acceptedAnswer":{"@type":"Answer","text":"Use dbatools: enumerate your instance list with `Get-DbaRegisteredServer`, then pull databases with `Get-DbaDatabase` per instance and upsert into a registry database. The full sweep script in this post handles instances, databases, and timestamps in one loop."}},{"@type":"Question","name":"Should I use a spreadsheet or a database for my SQL Server inventory?","acceptedAnswer":{"@type":"Answer","text":"A database. Spreadsheets cannot tell you when they went stale. Your automation cannot query them. And they cannot distinguish \"database is gone\" from \"nobody updated the row.\" A registry with FirstSeen/LastSeen timestamps does all three for free."}},{"@type":"Question","name":"How often should I scan my SQL Server estate for changes?","acceptedAnswer":{"@type":"Answer","text":"Daily for steady state. Drift moves at human speed, and hourly sweeps mostly reconfirm the registry you already had while adding connection load to every instance. Sweep hourly only during deliberate high-churn work like migrations, then stand down."}}]}]}
cover:
  image: "images/a1-estate-registry-hero.png"
  alt: "A holographic inventory ledger counting database icons above rows of server racks in a dark datacenter"
  relative: true
---

# Build a SQL Server Estate Registry: Inventory 25K Databases with PowerShell

Every estate-management failure I have watched starts the same way. Somebody asks how many databases we have, and the room goes quiet. The honest answer is a shrug and a spreadsheet someone updated in March.

A registry fixes the shrug. Not a spreadsheet. Not a wiki page. A queryable store that answers, in seconds: what exists, where, what version, who owns it, and how much it matters. This post builds one, with the actual scripts.

This is the first deep-dive from the [estate management operating model](/posts/sql-server-estate-management/). Everything there reads from or writes to the registry you are about to build.

> **Key Takeaways**
> - A minimum viable registry is four tables: Instance, Database, Ownership, Environment. Everything else is decoration until those answer queries.
> - Discovery is a solved problem: PowerShell plus dbatools sweeps any estate that answers a connection string. The unsolved parts are cadence and drift.
> - Daily sweeps beat hourly ones at fleet scale. Drift moves at human speed, and a sweep that costs more than the drift it catches is overhead, not hygiene.
> - New databases get marked for review, missing ones get quarantined, and nothing gets deleted by automation. Deletion is a human decision with a paper trail.

## Why Spreadsheets Die

The failure mode is never laziness. It is entropy with a work queue.

New databases appear because an app team restored a backup to "just test something." Old databases zombie because the migration finished but nobody dropped the source. A vendor install creates three databases and mentions it in a PDF nobody reads. None of these events page the DBA. The spreadsheet learns about none of them.

At twenty instances you can outrun entropy by walking the CMS list every Friday. At three hundred, the walking is the job. The registry inverts the relationship: automation feeds the store, and the store feeds the automation.

<!-- [ORIGINAL DATA] -->
In the estate I operate, daily sweeps catch a handful of undocumented databases in a typical week. Some weeks none, some weeks a dozen. The count matters less than the certainty that nothing lives more than 24 hours unregistered.

## The Minimum Viable Registry Schema

Four tables. Resist the urge to add a fifth until the first four answer questions in production.

```sql
CREATE TABLE registry.Instance (
    InstanceName    sysname NOT NULL PRIMARY KEY,
    DnsName         nvarchar(260) NOT NULL,
    Version         nvarchar(64)  NULL,
    Edition         nvarchar(128) NULL,
    PatchLevel      nvarchar(64)  NULL,
    FirstSeen       datetime2 NOT NULL DEFAULT sysutcdatetime(),
    LastSeen        datetime2 NOT NULL DEFAULT sysutcdatetime()
);

CREATE TABLE registry.Database (
    DatabaseId      int IDENTITY PRIMARY KEY,
    InstanceName    sysname NOT NULL REFERENCES registry.Instance(InstanceName),
    DatabaseName    sysname NOT NULL,
    SizeMB          bigint NULL,
    RecoveryModel   nvarchar(20) NULL,
    CompatLevel     smallint NULL,
    LastBackup      datetime2 NULL,
    LastDbcc        datetime2 NULL,
    FirstSeen       datetime2 NOT NULL DEFAULT sysutcdatetime(),
    LastSeen        datetime2 NOT NULL DEFAULT sysutcdatetime(),
    CONSTRAINT UQ_Database UNIQUE (InstanceName, DatabaseName)
);

CREATE TABLE registry.Ownership (
    DatabaseId      int NOT NULL REFERENCES registry.Database(DatabaseId),
    Application     nvarchar(128) NULL,
    Team            nvarchar(128) NULL,
    CriticalityTier tinyint NULL  -- 1 = crown jewels, 3 = mystery box
);

CREATE TABLE registry.Environment (
    DatabaseId      int NOT NULL REFERENCES registry.Database(DatabaseId),
    EnvironmentName nvarchar(20) NOT NULL  -- prod / staging / dev / unknown
);
```

Two design notes. First, `FirstSeen` and `LastSeen` are the whole drift-detection system. Everything else derives from comparing them. Second, criticality tier is a tinyint, not a paragraph. You will use it to sort patch rings later, and paragraphs do not sort.

## The Discovery Sweep

dbatools does the heavy lifting. If you are running fleet operations without it, stop here and install it first; writing fleet code without dbatools is self-harm.

```powershell
# Sweep-EstateRegistry.ps1
# Daily discovery sweep: enumerate instances, upsert registry, mark LastSeen.
[CmdletBinding()]
param(
    [Parameter(Mandatory)]
    [string]$RegistryServer,      # where the registry database lives

    [Parameter(Mandatory)]
    [string]$CmsGroup,            # registered-server group to sweep

    [string]$RegistryDatabase = 'EstateRegistry'
)

$ErrorActionPreference = 'Stop'
$targets = Get-DbaRegisteredServer -SqlInstance $RegistryServer -Group $CmsGroup

foreach ($target in $targets) {
    try {
        $inst = Get-DbaInstance -SqlInstance $target.Name -ErrorAction Stop
        Invoke-DbaQuery -SqlInstance $RegistryServer -Database $RegistryDatabase -Query @"
MERGE registry.Instance AS t
USING (SELECT '$($inst.Name)' AS InstanceName) AS s
ON t.InstanceName = s.InstanceName
WHEN MATCHED THEN UPDATE SET
    DnsName = '$($inst.ComputerName)',
    Version = '$($inst.Version)',
    Edition = '$($inst.Edition -replace "'","''")',
    PatchLevel = '$($inst.Build)',
    LastSeen = sysutcdatetime()
WHEN NOT MATCHED THEN INSERT (InstanceName, DnsName, Version, Edition, PatchLevel)
    VALUES (s.InstanceName, '$($inst.ComputerName)', '$($inst.Version)',
            '$($inst.Edition -replace "'","''")', '$($inst.Build)');
"@
        $dbs = Get-DbaDatabase -SqlInstance $target.Name -ExcludeSystem
        foreach ($db in $dbs) {
            Invoke-DbaQuery -SqlInstance $RegistryServer -Database $RegistryDatabase -Query @"
MERGE registry.Database AS t
USING (SELECT '$($inst.Name)' AS InstanceName, '$($db.Name -replace "'","''")' AS DatabaseName) AS s
ON t.InstanceName = s.InstanceName AND t.DatabaseName = s.DatabaseName
WHEN MATCHED THEN UPDATE SET
    SizeMB = $($db.SizeMB), RecoveryModel = '$($db.RecoveryModel)',
    CompatLevel = $($db.CompatibilityLevel -replace 'Version',''),
    LastBackup = '$($db.LastFullBackup)', LastSeen = sysutcdatetime()
WHEN NOT MATCHED THEN INSERT (InstanceName, DatabaseName, SizeMB, RecoveryModel, CompatLevel, LastBackup)
    VALUES (s.InstanceName, s.DatabaseName, $($db.SizeMB), '$($db.RecoveryModel)',
            '$($db.CompatibilityLevel -replace 'Version','')', '$($db.LastFullBackup)');
"@
        }
    }
    catch {
        Write-Warning "Sweep failed for $($target.Name): $($_.Exception.Message)"
        # Failure is data too: an instance that does not answer is an instance to chase.
    }
}
```

The script is deliberately boring. Connect, enumerate, MERGE, move on. Every clever thing you add here is a thing that breaks at 2 a.m. in month seven.

Run it from a SQL Agent job on the registry host, or from any scheduler that can hold a PowerShell session. The scheduler does not matter. The cadence does.

## Cadence: Why Daily, Not Hourly

Hourly sweeps feel diligent. They are mostly waste.

Drift moves at human speed. Databases get created during business hours by humans doing work. A sweep that runs at 04:00 catches everything that happened yesterday with hours to spare before the 08:00 standup. The sixteen sweeps in between catch the same registry you already had, plus connection load on every instance in the estate.

At three hundred instances, a full sweep touches three hundred listeners, three hundred logins, three hundred error logs. Hourly that is 7,200 touches a day of pure overhead. Daily is 300 touches and you lose nothing. The drift you would have caught at 11:00 still gets caught at 04:00. And the review workflow does not start until a human reads it anyway.

The exception is decommissioning projects and migration waves, where drift is deliberately high. Then you sweep the affected ring hourly for the duration and stand the cadence back down.

## Drift Handling: Review and Quarantine

The sweep makes drift visible. These two queries make it actionable:

```sql
-- New arrivals: first seen in the last sweep, never reviewed
SELECT i.InstanceName, d.DatabaseName, d.FirstSeen
FROM registry.Database d
JOIN registry.Instance i ON i.InstanceName = d.InstanceName
WHERE d.FirstSeen >= dateadd(day, -2, sysutcdatetime())
  AND NOT EXISTS (SELECT 1 FROM registry.Ownership o WHERE o.DatabaseId = d.DatabaseId)
ORDER BY d.FirstSeen DESC;

-- Zombies: not seen for 7 days
SELECT InstanceName, DatabaseName, LastSeen
FROM registry.Database
WHERE LastSeen < dateadd(day, -7, sysutcdatetime())
ORDER BY LastSeen;
```

New arrivals go into a review queue. Somebody looks at each one, assigns ownership and criticality, and moves on. In my queue that takes minutes a week, because the queue is the only place undocumented work can hide.

Zombies get quarantined, not deleted. Seven days unseen means the database went offline, got renamed, got dropped, or the instance stopped answering. Each of those has a different correct response, and none of them is "delete the row." Quarantine the row, chase the owner, and let a human close it out. Automation that deletes registry rows is automation that gaslights future you.

## The Registry Feeds Everything Else

This is the payoff. Once the registry is trustworthy, every other discipline reads from it.

The automation library builds its target lists from the registry instead of from hardcoded server names. Patch rings are just `WHERE CriticalityTier = n`. Monitoring scoping is `WHERE EnvironmentName = 'prod'`. Migration assessment is a GROUP BY over size, version, and dependency metadata that already exists because the sweep put it there.

That is the subject of the next posts. The automation library walkthrough covers the discovery-audit-remediation-verify structure that consumes this registry: [INTERNAL-LINK: the automation library → B1 spoke]. Documentation also drifts the moment you generate it. The SqlPowerDoc modernization for SQL Server 2019, 2022, and 2025 fills that gap: [INTERNAL-LINK: document modern SQL Server versions → A2 spoke].

## Getting Started

First, today: run `Get-DbaRegisteredServer` against your CMS and count what comes back. Then compare it to what you thought you had. The delta is the reason this post exists.

Second: stand up the four tables on any instance you control. A dedicated tiny registry database on your most boring server is fine. The registry should be the least interesting database you own.

Third: schedule the sweep for tonight. Tomorrow morning you will have a review queue. It will contain at least one database nobody can explain. That is not a failure of the registry. That is the registry working.

## Frequently Asked Questions

### How do I inventory all my SQL Servers with PowerShell?

Use dbatools: enumerate your instance list with `Get-DbaRegisteredServer`, then pull databases with `Get-DbaDatabase` per instance and upsert into a registry database. The full sweep script in this post handles instances, databases, and timestamps in one loop.

### Should I use a spreadsheet or a database for my SQL Server inventory?

A database. Spreadsheets cannot tell you when they went stale. Your automation cannot query them. And they cannot distinguish "database is gone" from "nobody updated the row." A registry with FirstSeen/LastSeen timestamps does all three for free.

### How often should I scan my SQL Server estate for changes?

Daily for steady state. Drift moves at human speed, and hourly sweeps mostly reconfirm the registry you already had while adding connection load to every instance. Sweep hourly only during deliberate high-churn work like migrations, then stand down.

## Conclusion

The registry is the foundation the whole operating model stands on. Four tables, one sweep, two review queries, and the discipline to run them daily. Everything else in this series reads from it.

The estate does not care how good your intentions are. It cares what runs on a schedule.

### Continue Learning

- [SQL Server Estate Management at 25,000 Databases: the operating model this post builds on](/posts/sql-server-estate-management/)
- [INTERNAL-LINK: The DBA Automation Library → B1 spoke]
- [INTERNAL-LINK: SqlPowerDoc for Modern SQL Server → A2 spoke]
