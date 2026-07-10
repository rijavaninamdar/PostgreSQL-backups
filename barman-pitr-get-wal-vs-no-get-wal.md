# Barman PITR: Understanding `--get-wal` vs `--no-get-wal` Options During Recovery

*Author: Rijavan Inamdar*

---

## Introduction

When performing Point-in-Time Recovery (PITR) using Barman, the `barman recover` command provides two key options that control how WAL (Write-Ahead Log) files are fetched and applied during the recovery process:

- `--get-wal`
- `--no-get-wal`

These options determine whether Barman automatically retrieves WAL files on demand for recovery, or expects them to be handled manually.

## Table of Contents

- [Option 1: Using `--get-wal`](#option-1-using---get-wal)
- [Option 2: Using `--no-get-wal`](#option-2-using---no-get-wal)
- [Considerations](#considerations)
- [Further Reading](#further-reading)
- [Tested Versions](#tested-versions)

---

## Option 1: Using `--get-wal`

**Example recovery command:**

```bash
barman recover <server_name> <backup_id> <data_directory> \
  --remote-ssh-command "ssh postgres@<target_host>" \
  --target-tli latest \
  --get-wal
```

The `--get-wal` option writes the `restore_command` using `barman-wal-restore`, enabling PostgreSQL to automatically fetch and restore required WAL segments on demand from the remote Barman server during recovery.

**Purpose:** `--get-wal` automatically retrieves only the WAL files needed, up to the specified recovery target. This ensures a fully automated PITR — simplifying recovery to a specific point in time while reducing manual steps and human error.

### Example Configuration

It writes the following parameters to the recovery configuration in `postgresql.auto.conf`:

```ini
restore_command = 'barman-wal-restore -P -U barman <server_name> <backup_name> %f %p'
recovery_target_timeline = latest
```

> `barman-wal-restore` is a client-side utility included in the `barman-cli` package, used to securely retrieve WALs over SSH from the Barman server.

**Note:** This requires installing `barman-cli` on the PostgreSQL host. It's a lightweight client tool, not a daemon, and is only used for WAL retrieval during recovery.

---

## Option 2: Using `--no-get-wal`

**Example recovery command:**

```bash
barman recover <server_name> <backup_id> <data_directory> \
  --remote-ssh-command "ssh postgres@<target_host>" \
  --target-tli latest \
  --no-get-wal
```

When you use `--no-get-wal`, Barman copies all required WAL segments in advance during the recovery process. This ensures PostgreSQL already has all necessary WAL files locally available, so it doesn't need to fetch them on demand during recovery.

With this approach:

- On-demand WAL fetching is **disabled**, and PostgreSQL uses the WAL files placed by Barman directly.
- The `restore_command` doesn't need `barman-wal-restore`, so you **don't** need to install `barman-cli` on the target PostgreSQL node.

### Trade-offs

| Trade-off | Impact |
|---|---|
| Extra WAL files may be copied beyond what's actually needed | Larger data transfers, increased disk usage |
| Larger data volume to prepare | Slightly longer recovery preparation time, especially for large databases |
| PostgreSQL still scans all copied WAL files | Unnecessary segments are discarded during recovery, which can further slow the process |

### Example Configuration

It writes the following parameters to the recovery configuration in `postgresql.auto.conf`:

```ini
restore_command = 'cp /data_dir/barman_wal/%f %p'
recovery_end_command = 'rm -fr /data_dir/barman_wal'
recovery_target_timeline = latest
```

---

## Considerations

> ⚠️ If you use `--no-get-wal`, you will **not** be able to perform a later or future recovery beyond the WAL files that were copied during the initial restore.

- Since on-demand WAL fetching is disabled, no additional WAL segments can be retrieved automatically from the Barman server at a later stage.
- Any future recovery will require **manually copying** the necessary WAL files to the PostgreSQL host.
- If you install `barman-cli` on the DB host afterward, you can update `restore_command` to use `barman-wal-restore` and enable on-demand recovery going forward.

---

## Further Reading

For additional information, refer to the [official Barman documentation](https://docs.pgbarman.org/).

---

## Tested Versions

This article has been tested against the following versions:

| Component | Version |
|---|---|
| PostgreSQL | 17 |
| Barman | 3.15.0 |
