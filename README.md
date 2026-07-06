# pgBackRest - An Enterprise Grade Backup Solution for PostgreSQL

The `pg_basebackup` utility included in the PostgreSQL binaries offers a great set of features for hot binary backups, remote backups, and standby building, etc. This is sufficient for small deployments. However, advanced users may want more sophisticated features offered by a complex backup solution.

pgBackRest is one such solution that addresses the shortcomings of `pg_basebackup`. It implements all backup features internally using a custom protocol for communicating with remote systems. Powerful features include parallel backup and restore, local or remote operation, full/incremental/differential backup types, backup rotation, archive expiration, backup integrity, page checksums, backup resume, streaming compression and checksums, delta restore, and much more.

pgBackRest doesn't rely on traditional backup tools like `tar` and `rsync`. This custom solution and protocol is a well-tuned fit for PostgreSQL-specific backup challenges. It allows for more flexibility and limits the types of connections required to perform a backup, which increases security. pgBackRest is a simple but feature-rich, reliable backup and restore system that can seamlessly scale up to the largest databases and workloads.

> **Version note:** From v2.00 onwards, pgBackRest uses a native C implementation for archive push, making `archive_command` processing significantly faster. PostgreSQL 12 removed `recovery.conf` and moved recovery parameters into regular PostgreSQL parameters — this is accommodated starting from pgBackRest v2.18. **If you run PostgreSQL 12+, you need pgBackRest 2.18 or above.**

---

## Process

### Concept

The most important concept in pgBackRest is the **stanza**. Each stanza defines configuration related to one PostgreSQL instance. Most DB servers will only have one PostgreSQL cluster (and therefore one stanza), whereas backup servers will have a stanza for every cluster that needs to be backed up. Each stanza definition specifies the backup repository to be used, and based on the backup policies specified, pgBackRest maintains retention and takes full or incremental backups.

---

## Features of pgBackRest

- Parallel backup / restore with compression — very high throughput
- Custom protocol — local or remote backup without direct access to PostgreSQL remotely, high security
- Full, incremental, and differential backup with custom logic
- Retention policies for backups and archive WALs
- Backups in data directory format — pgBackRest can maintain a backup data directory, so restore operations can be avoided in many cases
- Backup checksum and page checksum for integrity
- Ability to resume an aborted backup
- Streaming with compression and checksum
- Parallel, asynchronous WAL push & get
- Tablespace & link support with ability to remap locations
- Amazon S3 can be used as a backup repository
- Encrypted backups

---

## Installation

A prerequisite for installation is the PGDG repository. If it's not already installed, set it up from `yum.postgresql.org`.

```bash
$ sudo yum install pgbackrest
```

pgBackRest is developed in Perl, so when you install it, all dependent Perl libraries are also installed if not already present.

---

## Configuring Backup

The first step is creating a stanza definition in `/etc/pgbackrest.conf`. Here's a simple example. All `pg` options are specified with a `1`, which serves as the index of the configuration — used for configuring multiple PostgreSQL hosts. For example, a single master is configured with `pg1-path`, `pg1-host`, etc. If a standby is configured, index its options as `pg2-` (e.g. `pg2-host`, `pg2-path`).

```ini
sudo bash -c 'cat << EOF  > /etc/pgbackrest.conf
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2

[pg0app]
pg1-path=/var/lib/pgsql/11/data
pg1-port=5432
EOF'
```

### Create the Stanza

This needs to be done on the server where the repository is located. It's highly recommended to use the same non-root user under which the PostgreSQL process runs.

```bash
$ pgbackrest stanza-create --stanza=pg0app --log-level-console=info
```

**Sample output:**
```
2018-11-06 11:33:38.214 P00   INFO: stanza-create command begin 2.06: --log-level-console=info --pg1-path=/var/lib/pgsql/11/data --pg1-port=5432 --repo1-path=/var/lib/pgbackrest --stanza=pg0app
2018-11-06 11:33:38.710 P00   INFO: stanza-create command end: completed successfully (498ms)
```

### Enable WAL Archiving

At a minimum, WAL archiving should be enabled and `archive_command` should use pgbackrest, as shown below.

```sql
ALTER SYSTEM SET wal_level = 'replica';
ALTER SYSTEM SET archive_mode = 'on';
ALTER SYSTEM SET archive_command = 'pgbackrest --stanza=pg0app archive-push %p';
ALTER SYSTEM SET max_wal_senders = '10';
ALTER SYSTEM SET hot_standby = 'on';
```

These parameter changes require a restart of the PostgreSQL instance.

```bash
$ sudo systemctl restart postgresql-11
```

Verify the parameter changes from a fresh `psql` connection:

```sql
select name,setting
   from pg_settings
where name in ('wal_level','archive_mode','archive_command','max_wal_senders','hot_standby');
```

### Verify Configuration

```bash
$ pgbackrest check --stanza=pg0app --log-level-console=info
```

**Sample output if configuration is correct:**
```
2018-11-06 10:56:05.711 P00   INFO: check command begin 2.06: --log-level-console=info --pg1-path=/var/lib/pgsql/11/data --pg1-port=5432 --repo1-path=/var/lib/pgbackrest --stanza=pg0app
2018-11-06 10:56:07.674 P00   INFO: WAL segment 000000010000000000000001 successfully stored in the archive at '/var/lib/pgbackrest/archive/pg0app/11-1/0000000100000000/000000010000000000000001-3343303c5f71e04efe0eeba1844b69a813b0205c.gz'
2018-11-06 10:56:07.675 P00   INFO: check command end: completed successfully (1964ms)
```

---

## Running Backups

```bash
$ pgbackrest backup --stanza=pg0app --log-level-console=info
```

By default, pgbackrest takes an **incremental** backup when a full backup already exists. Output looks like:

```
2018-11-06 12:12:53.897 P00   INFO: last backup label = 20181106-121059F, version = 2.06
2018-11-06 12:12:54.631 P00   INFO: execute non-exclusive pg_start_backup() with label "pgBackRest backup started at 2018-11-06 12:12:53": backup begins after the next regular checkpoint completes
2018-11-06 12:13:03.780 P00   INFO: backup start archive = 00000001000000000000000D, lsn = 0/D000108
2018-11-06 12:13:06.240 P01   INFO: backup file /var/lib/pgsql/11/data/base/13878/16418 (64MB, 82%) checksum 7a1d5ecb0f1ed90a908fe63e5946bc95bcc64e86
```

If no relevant full backup exists per the stanza's backup policy, pgbackrest automatically switches to a **full** backup:

```
WARN: no prior backup exists, incr backup has been changed to full
2018-11-06 11:21:59.825 P00   INFO: execute non-exclusive pg_start_backup() with label "pgBackRest backup started at 2018-11-06 11:21:58": backup begins after the next regular checkpoint completes
2018-11-06 11:22:00.042 P00   INFO: backup start archive = 000000010000000000000003, lsn = 0/3000028
2018-11-06 11:22:01.502 P01   INFO: backup file /var/lib/pgsql/11/data/base/13878/1255 (608KB, 2%) checksum 743dc8ba40501dd88670492108c9b1b9fed2ee8c
2018-11-06 11:22:01.522 P01   INFO: backup file /var/lib/pgsql/11/data/base/13877/1255 (608KB, 5%) checksum 743dc8ba40501dd88670492108c9b1b9fed2ee8c
```

---

## Listing Backups

The `info` command lets admins check, list, and verify available backups.

```bash
$ pgbackrest info
```

If no backups are available:

```
stanza: pg0app
    status: error (no valid backups)

    db (current)
        wal archive min/max (10-1): none present
```

Otherwise, it lists backup information:

```
stanza: pg0app
    status: ok
    cipher: none

    db (current)
        wal archive min/max (11-1): 000000010000000000000008 / 000000030000000000000018

        full backup: 20181106-121059F
            timestamp start/stop: 2018-11-06 12:10:59 / 2018-11-06 12:12:28
            wal start/stop: 000000010000000000000008 / 000000010000000000000008
            database size: 98.4MB, backup size: 98.4MB
            repository size: 6.7MB, repository backup size: 6.7MB

        incr backup: 20181106-121059F_20181106-121253I
            timestamp start/stop: 2018-11-06 12:12:53 / 2018-11-06 12:13:07
            wal start/stop: 00000001000000000000000D / 00000001000000000000000D
            database size: 98.4MB, backup size: 77.3MB
            repository size: 6.7MB, repository backup size: 4.2MB
            backup reference list: 20181106-121059F
```

---

## Restoring Backup

Restoration works when there is an empty data directory. Recreate the data directory beforehand if required.

```bash
$ pgbackrest restore --stanza=pg0app --log-level-console=info
```

**Sample restore steps:**

1. Stop the PostgreSQL instance.
   ```bash
   $ sudo systemctl stop postgresql-11
   ```

2. Clean the data directory of the corrupt instance.
   ```bash
   $ rm -rf /var/lib/pgsql/11/data
   ```

3. Create the data directory and set correct permissions.
   ```bash
   $ mkdir /var/lib/pgsql/11/data/
   $ chmod 0700 /var/lib/pgsql/11/data
   ```

4. Restore the backup. This restores the most recent successful full backup plus all incremental backups and archives pushed to the repository.
   ```bash
   $ pgbackrest restore --stanza=pg0app --log-level-console=info
   ```

5. Start the PostgreSQL service.
   ```bash
   $ sudo systemctl start postgresql-11
   ```

So far, we've covered basic backup and restoration using pgBackRest with a filesystem-based repository.

---

## Backup to Amazon S3 Bucket

Many organizations prefer backing up their database to the cloud, since cloud storage is often cheaper and more reliable. Here's how to set up backup to S3.

An S3 bucket needs to be set up per Amazon's documentation. Before configuring the PostgreSQL backup, verify access using the AWS CLI and push a test file to the bucket:

```bash
$ aws s3 cp /tmp/test.txt s3://pg-percona-bkup
```

Find the S3 bucket's region/location if unknown:

```bash
$ aws s3api get-bucket-location --bucket pg-percona-bkup
```

**Sample output:**
```json
{
    "LocationConstraint": "us-east-2"
}
```

> Refer to Amazon's documentation to find the correct endpoint for the S3 bucket's region.

Now specify this configuration in the pgBackRest configuration file. A sample S3 configuration in `/etc/pgbackrest.conf`:

```ini
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=5
repo1-s3-bucket=pg-percona-bkup
repo1-s3-endpoint=s3.us-east-2.amazonaws.com
repo1-s3-key=AKIxxxxxxxxxxxxxxxxxxxxA
repo1-s3-key-secret=tKxxxxxxxxxxxxxxxxxxxxxxxxxxxxX
repo1-s3-region=us-east-2
repo1-s3-verify-ssl=n
repo1-type=s3

[pg0app]
pg1-path=/var/lib/pgsql/11/data
pg1-port=5432
```

⚠️ **Security note:** Never commit real access keys/secrets to a config file in version control or documentation. Use placeholders (as above) and store actual credentials in a secrets manager or environment-restricted config.

Once the configuration is in place, create the stanza and push the backup using the same steps as before:

```bash
$ pgbackrest stanza-create --stanza=pg0app --log-level-console=info
$ pgbackrest backup --type=full --stanza=pg0app --log-level-console=info
```

The rest of the steps are the same as discussed earlier.
