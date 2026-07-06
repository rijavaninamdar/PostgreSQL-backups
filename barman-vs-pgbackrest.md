# Barman Limitations and Comparison to pgBackRest

## Introduction

You may be curious about the differences between Barman and pgBackRest.

## Root Cause

### Barman Limitations

Barman is just a wrapper for `rsync` and `pg_basebackup`, so all the limitations of both tools apply here.

**For the RSYNC method, the limitations are:**

- The copies of files can sometimes be inconsistent/corrupted if they are taken during peak hours on files that were accessed concurrently by multiple processes.
- Requires passwordless SSH access to the database server, which could be an issue from a security point of view.

**`pg_basebackup` limitations are:**

- Only full backups are supported prior to PostgreSQL version 17, where incremental backup support was introduced to `pg_basebackup`.
- Always works with a single process only, so for big databases, it can be problematic due to the amount of time required to finish the backup.

## pgBackRest vs. Barman Comparison

### Where Barman Has the Advantage

Barman has two advantages over pgBackRest:

1. **Streaming archiver** — works just like a streaming replica and sends all WAL entries immediately to the archive on the Barman server, allowing you to achieve an RPO close to zero. A replica server usually plays this role.
2. **Native replication protocol** — it can perform backups using the native replication protocol, so it is possible to take backups without installing or configuring anything on the OS/in the container.

### Where pgBackRest Has the Advantage

It is, however, losing in many other aspects. pgBackRest can:

- Perform incremental and differential backups for all PostgreSQL versions, not only the latest PG17. Differential backups in Barman are not possible at all.
- Perform **delta restoration**, restoring only files that are different from backups based on checksums, which can greatly improve restore speed. The bigger the DB, the better the result.
- Encrypt the backup repository.
- Use the TLS protocol for safer file copying.
- Asynchronously archive and restore WAL files using multiple parallel processes, which significantly increases WAL archive and recovery throughput.
- Automatically detect which server in the cluster is the primary and take backups of most of the database files from a standby, greatly reducing the impact of backup performance on the primary.
- Restore only one database from a backup, which is not intended for production purposes, but allows restoring just what's needed and moving that to the production cluster.

---

*This solution is part of Percona's Customer Knowledge Base, providing a vast library of solutions that Percona Engineers have created while supporting our customers. To give you the knowledge you need the instant it becomes available, some articles may present a raw and unedited form.*

*This article has been tested for the following versions of technologies:*
