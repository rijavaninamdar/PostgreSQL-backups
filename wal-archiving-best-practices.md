# WAL Archiving Best Practices

*Author: Rijavan Inamdar*

---

## Introduction

The core premise of PostgreSQL DR/PITR scenarios is combining physical base backups with WAL archiving.

When archiving WAL logs for potential DR/PITR scenarios, there are many solutions available, which we discuss in this article.

We highly recommend using a supported tool for a comprehensive physical backup solution — specifically **[Barman](https://docs.pgbarman.org/)** or **[pgBackRest](https://pgbackrest.org/)**, for their reliability and features.

While the core behavior of base backups and WAL archives is achieved by both Barman and pgBackRest, each has various additional functionality, discussed in later sections. See our [comparison of backup method features](https://www.enterprisedb.com/products/backup-recovery) for more detail.

> **Note:** All commands mentioned in this article can behave differently across operating systems, and should be tested thoroughly in a test environment before being applied to production.

## Table of Contents

- [WAL Archiving Options](#wal-archiving-options)
- [1. Barman](#1-barman)
- [2. pgBackRest](#2-pgbackrest)
- [3. `archive_library` & `basic_archive` (PG15+, native)](#3-archive_library--basic_archive-pg15-native)
- [4. Native `pg_receivewal`, Replication Slot, and `systemd` Scheduler](#4-native-pg_receivewal-replication-slot-and-systemd-scheduler)
- [5. `archive_command` Using `rsync`](#5-archive_command-using-rsync)
- [6. `archive_command` Using `cp`](#6-archive_command-using-cp)
- [7. `archive_command` Using `scp`](#7-archive_command-using-scp)

---

## WAL Archiving Options

| # | Solution | WAL Archiving Durability | Complexity Level | Method |
|---|----------|:---:|:---:|:---:|
| 1 | Barman backup tool | High | Medium | Push and/or Pull |
| 2 | pgBackRest backup tool | High | Medium | Push |
| 3 | Native `archive_library` (PG15+) | High | Medium | Push |
| 4 | Native `pg_receivewal`, replication slot, and `systemd` scheduler | High | High | Pull |
| 5 | Native `archive_command` using `rsync` | Medium | Easy | Push |
| 6 | Native `archive_command` using `cp` | Low | Easy | Push |
| 7 | Native `archive_command` using `scp` | Low | Easy | Push |

**Column definitions:**

- **Solution** — the different options that can be used for WAL archiving.
- **WAL Archiving Durability** — how safe the archiving procedure is. Key factors (elaborated below) include whether WAL files are `fsync`'d to disk, and how networking issues are handled.
- **Complexity Level** — how difficult the solution is to set up and maintain in production. Even a theoretically safer method can become unsafe in practice if it's too complex for the team's needs and gets implemented incorrectly.
- **Method** — the broad principle behind how WAL logs are archived:
  - **Push** — PostgreSQL sends WAL files to a location after creating them.
  - **Pull** — PostgreSQL replication is used to retrieve WAL data as it's created and store it in a location.

---

## 1. Barman

### Barman Background

[Barman](https://docs.pgbarman.org/) is one of our recommended physical backup tools. Its `barman cron` function is very effective at managing backup processes.

Barman includes many features, such as:

- [Incremental](https://docs.pgbarman.org/release/3.5.0/#incremental-backup) and [full](https://docs.pgbarman.org/release/latest/#backup) base backup creation
- [WAL archiving through `archive_command` and `barman-wal-archive`](https://docs.pgbarman.org/release/latest/#wal-archiving-via-barman-wal-archive)
- [Streaming WALs through `pg_receivewal`](https://docs.pgbarman.org/release/latest/#wal-streaming)
- [Backup restoration](https://docs.pgbarman.org/release/latest/#recover)
- [Retention policies](https://docs.pgbarman.org/release/latest/#retention-policies)
- [Parallel jobs](https://docs.pgbarman.org/release/latest/#parallel-jobs)
- [Geographical redundancy](https://docs.pgbarman.org/release/latest/#geographical-redundancy)
- [Cloud snapshots](https://docs.pgbarman.org/release/latest/#backup-with-cloud-snapshots)

Full usage documentation is available [on the Barman website](https://docs.pgbarman.org/).

### `archive_command` & `barman-wal-archive`

Barman improves on PostgreSQL's `archive_command` parameter through its [`barman-wal-archive`](https://docs.pgbarman.org/release/latest/#wal-archiving-via-barman-wal-archive) command:

- `barman-wal-archive` guarantees that WAL file content is `fsync`'d to disk, avoiding production issues caused by the fragility of `rsync`, `cp`, or `scp`.
- It also reduces the risk of copying a WAL file into the wrong directory on the Barman host, since the only parameters used in `archive_command` are the server's ID and the WAL path.

**Example:**

```bash
archive_command = 'barman-wal-archive barmanserver db01 %p'
```

- `barmanserver` — the Barman hostname
- `db01` — the database server being backed up, specified as `[db01]` in a Barman configuration file (e.g., `/etc/barman.d/db01.conf`)
- `%p` — the pathname of each WAL file PostgreSQL creates (`$PGDATA/pg_wal/wal_file_name`); PostgreSQL re-runs the command with each new file name

### Archiving to Multiple Servers

Some use cases call for archiving to multiple locations. Use [Barman geo-redundancy](https://docs.pgbarman.org/release/latest/#geographical-redundancy) to achieve this — multiple Barman servers can be configured, where one Barman server uses another Barman server's backups as its primary data source.

> ⚠️ **Do not combine `rsync`/`cp`/`scp` with `barman-wal-archive`.**

We've seen customers run into production issues from `archive_command`s structured like this:

```bash
# DO NOT DO THIS!
archive_command = 'test ! -f /archive_location/%f && cp %p /archive_location/%f && barman-wal-archive barmanserver db01 %p'
```

Using `rsync`/`cp`/`scp` alongside `barman-wal-archive` negates the durability that `barman-wal-archive` provides. In this example, if the `cp` step fails, the use of `&&` also prevents `barman-wal-archive` from ever running. While swapping `&&` for `;` would fix that specific issue, it's still inferior to Barman's cascading backups — the first archival attempt would still rely on the less-durable `cp`.

### Barman `pg_receivewal`

Barman also supports [streaming WAL archives](https://docs.pgbarman.org/release/latest/#wal-streaming) through `pg_receivewal`, which minimizes RPO. This is discussed further in [pg_receivewal background](#pg_receivewal-background) below.

---

## 2. pgBackRest

### pgBackRest Background

[pgBackRest](https://pgbackrest.org/) is also one of our recommended physical backup tools. It uses file- and directory-level `fsync` to ensure durability.

pgBackRest includes many features, such as:

- [Full/base, differential, and incremental backup creation](https://pgbackrest.org/user-guide-rhel.html#concept/backup)
- [WAL archiving via `archive_command` and `pgbackrest archive-push`](https://pgbackrest.org/user-guide-rhel.html#quickstart/configure-archiving)
- [Backup restoration](https://pgbackrest.org/user-guide-rhel.html#quickstart/perform-restore)
- [Retention policies](https://pgbackrest.org/user-guide-rhel.html#retention)
- [Parallel backup & restore](https://pgbackrest.org/user-guide-rhel.html#parallel-backup-restore)
- [Page checksum validation during backups](https://pgbackrest.org/configuration.html#section-backup/option-checksum-page)
- [Backup repository encryption](https://pgbackrest.org/user-guide-rhel.html#quickstart/configure-encryption)
- [Multiple compression types](https://pgbackrest.org/configuration.html#section-general/option-compress-type)
- [Parallel and asynchronous archiving](https://pgbackrest.org/user-guide-rhel.html#async-archiving)
- [Resuming a failed backup](https://pgbackrest.org/configuration.html#section-backup/option-resume)
- S3, Azure, and GCS compatible object store support
- [Using multiple repositories simultaneously](https://www.enterprisedb.com/docs/supported-open-source/pgbackrest/08-multiple-repositories)

Full usage documentation is available [on the pgBackRest website](https://pgbackrest.org/) and in the [EDB docs](https://www.enterprisedb.com/docs/supported-open-source/pgbackrest/).

### `archive_command` & pgBackRest

If using this solution, configure [`pgbackrest archive-push`](https://pgbackrest.org/command.html#command-archive-push) in your `archive_command`.

**Example:**

```bash
archive_command = 'pgbackrest --stanza=demo archive-push %p'
```

This is demonstrated in the [pgBackRest quickstart guide](https://pgbackrest.org/user-guide.html#quickstart/configure-archiving).

---

## 3. `archive_library` & `basic_archive` (PG15+, Native)

If you're on PG15+ and unable to use Barman or pgBackRest, we recommend using the native [`archive_library`](https://www.postgresql.org/docs/current/runtime-config-wal.html#GUC-ARCHIVE-LIBRARY) and [`basic_archive`](https://www.postgresql.org/docs/current/basic-archive.html) modules.

- This functionality `fsync`s the WAL file, ensuring archiving durability.
- The server avoids recycling or removing WAL files until the module confirms they were successfully archived.
- After a server crash, temporary `archtemp` files should be deleted before restarting the server — see the [documentation notes](https://www.postgresql.org/docs/current/basic-archive.html#id-1.11.7.15.6).

### Example Setup (EPAS 15, applies to PostgreSQL 15 as well)

```sql
edb=# show shared_preload_libraries;

             shared_preload_libraries              
$libdir/dbms_pipe,$libdir/edb_gen,$libdir/dbms_aq,$libdir/basic_archive

edb=# alter system set shared_preload_libraries='$libdir/dbms_pipe','$libdir/edb_gen','$libdir/dbms_aq','$libdir/basic_archive';
```

```bash
$ /usr/local/enterprisedb/bin/pg_ctl -D /var/lib/edb/as15 -l /var/lib/edb/as15/log/logfile restart
```

```sql
edb=# show archive_library;

edb=# show basic_archive.archive_directory;

edb=# alter system set archive_library = 'basic_archive';
edb=# alter system set basic_archive.archive_directory = '/var/lib/edb/archive_test';

edb=# SELECT pg_reload_conf();

edb=# show archive_library;
 basic_archive

edb=# show basic_archive.archive_directory;
 /var/lib/edb/archive_test

edb=# create table test( i int);
edb=# insert into test generate_series(1,1000);
edb=# select pg_switch_wal();
edb=# insert into test generate_series(1,1000);
edb=# select pg_switch_wal();

edb=# select * from pg_stat_archiver;

 archived_count | last_archived_wal        | last_archived_time               | failed_count | last_failed_wal | last_failed_time | stats_reset
----------------+--------------------------+-----------------------------------+--------------+------------------+-------------------+------------------------------
 23             | 000000010000000000000019 | 09-FEB-23 17:26:38.722981 +00:00 | 0            |                  |                   | 07-FEB-23 16:28:15.290586 +00:00
```

```bash
$ ls -l /var/lib/edb/archive_test

-rw------- 1 enterprisedb enterprisedb 16777216 Feb  9 17:26 000000010000000000000018
-rw------- 1 enterprisedb enterprisedb 16777216 Feb  9 17:26 000000010000000000000019

$ ps waux | grep archive
postgres: archiver last was 000000010000000000000019
```

---

## 4. Native `pg_receivewal`, Replication Slot, and `systemd` Scheduler

### `pg_receivewal` Background

If you're on PG14 or older and unable to use Barman or pgBackRest, `archive_command` can be replaced with the native PostgreSQL tool [`pg_receivewal`](https://www.postgresql.org/docs/current/app-pgreceivewal.html), which streams WAL logs via replication to an archive location.

- `pg_receivewal` archiving is more durable than using `rsync`, `cp`, or `scp` inside `archive_command`. When writing WALs to `pg_wal` and sending them via replication, PostgreSQL uses [`fsync`](https://www.postgresql.org/docs/current/runtime-config-wal.html#id-1.6.7.8.3.2.2.1.3) to ensure updates are physically written to disk.
- Because of this, `pg_receivewal` lowers RPO — it streams the write-ahead log in real time instead of waiting for an entire WAL file to fill.
- Synchronous archiving can be enabled via `pg_receivewal --synchronous`, `synchronous_standby_names`, and `synchronous_commit`, to guarantee every transaction is archived and achieve an RPO of 0 — though this comes at the cost of performance. Per the [documentation](https://www.postgresql.org/docs/current/app-pgreceivewal.html), this should **not** be used alongside `synchronous_commit=remote_apply`.

### Standalone `pg_receivewal` (Without Barman) Complexity

> Because it requires an external scheduling system to maintain, **this approach is more complex than using Barman and would benefit from a professional services engagement** to evaluate fit, fully test, and implement the solution. It effectively replaces functionality that `barman cron` already provides.

- Maintenance includes restarting the process whenever it goes offline (e.g., PostgreSQL restarts and kills `pg_receivewal`, or the archive server host reboots). A custom process — such as a `systemd` unit — needs to be written to manage this.
- If `pg_receivewal` is your only WAL archiving solution, it's strongly recommended to use a [physical replication slot](https://www.postgresql.org/docs/current/warm-standby.html#STREAMING-REPLICATION-SLOTS-CONFIG); otherwise WAL data can be lost. With a replication slot, if `pg_receivewal` stops (networking issues, process failure, archive host down), the slot becomes inactive and the primary won't recycle WALs until the process resumes (assuming `max_slot_wal_keep_size=1`).
- However, if the primary holds onto WAL logs indefinitely, disk usage can reach 100% and take the primary down. During a prolonged outage, you'll need to make a judgment call:
  - Reduce the risk of losing backup data, at the risk of the primary crashing due to disk exhaustion, **or**
  - Preserve primary availability, at the risk of losing WAL segments (impacting backups and standbys).

**Monitoring** `pg_receivewal` archiving is achieved through:

- Database replication: `SELECT * FROM pg_stat_replication;`, `SELECT * FROM pg_replication_slots;`
- The scheduling process: `systemctl status <systemd_name>`

---

## 5. `archive_command` Using `rsync`

If a backup tool isn't an option, an `archive_command` using [`rsync`](https://linux.die.net/man/1/rsync) can be used.

- `rsync` first creates a temporary copy of the file, and only renames it to the target filename once the copy succeeds.
- Once customer environments widely support [rsync 3.2.4+](https://download.samba.org/pub/rsync/NEWS#ENHANCEMENTS-3.2.4), we'll cover using [`rsync --fsync`](https://download.samba.org/pub/rsync/rsync.1#opt--fsync).

**Example:**

```bash
archive_command = 'test ! -f enter_backup_location/%f && rsync -a %p enter_backup_location/%f'
```

- `-a` — runs `rsync` in archive mode (recursive, preserving all but hard links)
- `%p` — the pathname of each WAL file PostgreSQL creates (`$PGDATA/pg_wal/wal_file_name`); PostgreSQL re-runs the command for each new file
- `%f` — the name of the WAL segment file
- `test ! -f enter_backup_location/%f` — checks the target filename doesn't already exist before writing; if it does, the command fails rather than risk overwriting data that may still be needed

`test ! -f` exists specifically for the case where a same-named file already exists due to a prior problem — PostgreSQL will retain its WALs until the archive location is fixed and the conflict is resolved manually. Without this check, the existing file would simply be overwritten.

> ⚠️ Do **not** use the `--ignore-existing` flag. If a file with the same name already exists, PostgreSQL would skip archiving it (even if its contents differ), since the Linux exit status would be `0` — silently losing backup data.

---

## 6. `archive_command` Using `cp`

- `cp` is prone to networking issues.
- `cp` is not `ENOSPC`-safe: if the disk fills up and postmaster hits `ENOSPC`, the `cp` process can get stuck even after disk space is freed, leaving a partial file that requires manual removal. `rsync` avoids this issue.
- `rsync` is recommended over `cp` since it's more robust to networking issues and avoids the `ENOSPC` problem — making it less likely to trigger a WAL archiving failure.
- If you're currently using `cp` in `archive_command`, switching to `rsync` is a low-effort, high-value change worth testing in your test environment.

---

## 7. `archive_command` Using `scp`

`scp` is not recommended — it's inferior to both `barman-wal-archive` and `rsync`, and is also [being deprecated](https://www.redhat.com/en/blog/openssh-scp-deprecation-rhel-9-what-you-need-know).

If you're currently using `scp` in `archive_command`, switching to `rsync` is a low-effort, high-value change worth testing in your test environment.
