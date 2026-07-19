# PostgreSQL Backups 🐘💾

A curated collection of PostgreSQL backup, recovery, and WAL archiving documentation — covering **Barman** and **pgBackRest**, with practical guides for point-in-time recovery (PITR), enterprise-grade backup strategy, and WAL management best practices.

This repo is maintained as a reference for setting up, comparing, and operating backup/DR solutions in production PostgreSQL environments.

---

## 📚 Contents

| Document | Description |
|---|---|
| [`barman-kickoff-guide.md`](./barman-kickoff-guide.md) | Getting-started guide for setting up Barman — installation, configuration, and initial backup setup. |
| [`barman-pitr-get-wal-vs-no-get-wal.md`](./barman-pitr-get-wal-vs-no-get-wal.md) | Deep dive into Point-in-Time Recovery (PITR) with Barman, comparing `get-wal` vs `no-get-wal` restore approaches. |
| [`barman-vs-pgbackrest.md`](./barman-vs-pgbackrest.md) | Feature-by-feature comparison between Barman and pgBackRest to help choose the right tool for your environment. |
| [`pgBackRest-Enterprise-Backup-Solution.md`](./pgBackRest-Enterprise-Backup-Solution.md) | Enterprise-scale backup architecture and implementation using pgBackRest. |
| [`wal-archiving-best-practices.md`](./wal-archiving-best-practices.md) | Best practices for configuring and managing WAL archiving in PostgreSQL. |

---

## 🎯 Purpose

Reliable backup and disaster recovery are critical for any production PostgreSQL deployment. This repository documents:

- ✅ Backup tool setup and configuration (Barman, pgBackRest)
- ✅ Point-in-time recovery (PITR) strategies
- ✅ WAL archiving configuration and tuning
- ✅ Tool comparisons to guide architecture decisions
- ✅ Enterprise-grade backup design considerations (RTO/RPO alignment)

## 🛠️ Who This Is For

Database administrators, SREs, and platform engineers responsible for PostgreSQL high availability, backup, and disaster recovery, looking for practical, field-tested guidance beyond vendor documentation.

## 📖 How to Use

Each `.md` file is self-contained and can be read independently. It's recommended to start with the kickoff guide for your chosen tool (Barman or pgBackRest), then reference the comparison and best-practices docs as needed when designing your backup strategy.

## 🤝 Contributing

Suggestions, corrections, and additions are welcome — feel free to open an issue or submit a pull request.

## 📄 License

This documentation is provided as-is for educational and reference purposes.

---

*Maintained by [Rijavan Inamdar](https://github.com/rijavaninamdar) — Senior Database Administrator specializing in PostgreSQL, MySQL, and MongoDB high availability, backup/DR, and performance tuning.*
