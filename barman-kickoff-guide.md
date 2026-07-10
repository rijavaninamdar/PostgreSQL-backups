# Barman Kickoff Guide

**What you need to know before installing Barman**

*Author: Rijavan Inamdar*

---

## Overview

Before the installation of Barman on your Linux server can begin, we ask that you provide some additional information about your disaster recovery requirements. We'd also like to work with you to understand and/or help you define your disaster recovery processes.

This document details the information needed to successfully kick off your Barman installation. Please fill out the [questionnaire](#barman-kickoff-questionnaire) at the end to the best of your ability.

## Table of Contents

- [Server Access](#server-access)
- [Why a Separate Server Is Necessary for Barman](#why-a-separate-server-is-necessary-for-barman)
- [Which Server(s) Will Barman Backup](#which-servers-will-barman-backup)
- [Disk Space Necessary for Barman](#disk-space-necessary-for-barman)
- [Virtual Machine or Physical Server?](#virtual-machine-or-physical-server)
- [Linux Distribution](#linux-distribution)
- [Storage Type](#storage-type)
- [Backup Method](#backup-method)
- [Dedicated Recovery Server(s)](#dedicated-recovery-servers)
- [Integration with Monitoring Infrastructure](#integration-with-monitoring-infrastructure)
- [Recovery Point Objective (RPO)](#recovery-point-objective-rpo)
- [Recovery Time Objective (RTO)](#recovery-time-objective-rto)
- [S3 Relay](#s3-relay)
- [Integration with Standby Servers](#integration-with-standby-servers)
- [Training and Disaster Recovery Simulations](#training-and-disaster-recovery-simulations)
- [Barman Kickoff Questionnaire](#barman-kickoff-questionnaire)

---

## Server Access

In order to install and configure Barman, we will need to access the Barman and PostgreSQL servers through SSH. Please let us know if we can access them directly using a pool of trusted, static IP addresses, or if we will need to access through VPN.

## Why a Separate Server Is Necessary for Barman

> It is strongly recommended to have a separate server dedicated to Barman, ideally on different storage than the PostgreSQL server.

Having a separate server simplifies the overall architecture of your PostgreSQL servers, making it easier to maintain — especially when standby servers are involved.

It's typically suggested to share just the network between the PostgreSQL server and Barman, in order to reduce the risk of data loss after a failure or disaster — both of which are inevitable.

**Key benefits of a dedicated Barman server:**

- **`get-wal` capability** — The WAL archive of Barman can act as an "infinite" basin for all your standby servers, serving as a fallback if streaming replication has problems (see [Integration with Standby Servers](#integration-with-standby-servers)).
- **Control room for recovery operations** — The Barman server can also function as a monitoring hub, which is especially valuable if you plan to share one Barman server across multiple PostgreSQL servers.

It's recommended to keep the Barman server in close proximity to the PostgreSQL servers — ideally in the same data centre. At minimum, ensure connectivity is sufficient, especially if you intend to run Barman in a **"zero data loss"** context with **RPO = 0**.

## Which Server(s) Will Barman Backup

We'll need the IP addresses and hostnames of the PostgreSQL servers to be backed up with Barman, including their PostgreSQL version. If available, please also provide the IP address and hostname of the Barman server itself.

## Disk Space Necessary for Barman

Barman will back up one or more existing PostgreSQL servers. To estimate the disk space required, we need to know:

- Total size of all databases on the PostgreSQL server
- WAL production rate
- Frequency of full backups (weekly / daily)
- Retention policies required by your business (e.g., 4 weeks, or last 2 backups)
- Expected growth rate of the database

## Virtual Machine or Physical Server?

Will Barman run on a virtual machine or a dedicated physical server? Choose whichever approach best suits your business needs — Barman simply needs disk space and CPU cores, scaled to the number and size of PostgreSQL servers it backs up.

## Linux Distribution

Barman packages are developed and supported for major Linux distributions. Please let us know which distribution the Barman server will run on.

## Storage Type

Let us know your storage strategy for Barman — for example, local disks on a physical server, an NFS volume on a VMware virtual machine, or another approach.

## Backup Method

Barman supports two backup methods:

| Method | Description | Notes |
|---|---|---|
| **rsync/ssh** | Requires a passwordless SSH connection between the Barman server (`barman` user) and the PostgreSQL server (`postgres` user). | Enables incremental backup (via hard links) and network compression. |
| **postgres** | Relies on `pg_basebackup`, PostgreSQL's native physical backup tool. Requires a PostgreSQL streaming replication connection between Barman and the PostgreSQL server. | No SSH required, but no incremental backup or network compression. Requires PostgreSQL 9.1+. **Only option for PostgreSQL on Windows.** |

## Dedicated Recovery Server(s)

> **A backup that is not tested is not a valid backup.**

It's suggested that one or more servers be dedicated to recovery, including for Business Intelligence or staging purposes. Using hook scripts, Barman lets you automate the rebuild of recovery PostgreSQL servers, enabling unsupervised, ongoing testing of your backups.

## Integration with Monitoring Infrastructure

Monitoring is one of the most important characteristics of any ICT infrastructure with business continuity requirements.

The `barman check` command has been present since Barman's earliest prototype and is a handy way to verify that all major components of a PostgreSQL disaster recovery solution are working smoothly. If Nagios/Icinga are in use, Barman can natively act as an NRPE agent.

Let us know if you plan to monitor Barman proactively — this is recommended.

## Recovery Point Objective (RPO)

RPO is a business continuity metric defining the *maximum targeted period during which data might be lost from an IT service due to a major incident*.

Properly configured, an open-source solution combining Barman, PostgreSQL, and streaming replication can achieve an **RPO of 0**.

*Please indicate your desired RPO.*

## Recovery Time Objective (RTO)

RTO is a business continuity metric defining the *targeted duration of time and service level within which a business process must be restored after a disaster or disruption*, in order to avoid unacceptable consequences.

In Barman, this is generally measured by the time it takes to rebuild a PostgreSQL server from scratch and recover the latest available data (or recover to a specific point in time) from an available backup.

*Please indicate your desired RTO.*

## S3 Relay

Let us know if you'd like Barman to relay both WAL files and base backups to an Amazon S3 bucket in the cloud, adding resilience to your disaster recovery solution.

## Integration with Standby Servers

Barman stores WAL files for all your backups, so depending on your retention policy you may have several days, weeks, or months' worth of WAL files available.

Using Barman's `get-wal` command together with the [`barman-cli`](https://pgbarman.org) package (installed on every PostgreSQL server), your standby servers can pull required WAL files from Barman — a useful fallback in case of temporary or prolonged streaming replication issues.

## Training and Disaster Recovery Simulations

We value regular testing and simulation of disaster scenarios. Let us know if you'd like basic training on Barman and disaster recovery, along with assistance building out your PostgreSQL disaster recovery plan.

---

## Barman Kickoff Questionnaire

Please complete the following before installation begins.

1. **Where is the Barman server located?**
   - [ ] Same data centre as PostgreSQL
   - [ ] Different data centre in the same metropolitan area
   - [ ] Remote data centre

2. **How will we access your servers?**
   *(Direct access via a pool of trusted static IPs, or via VPN?)*

3. **List IP addresses and hostnames of the PostgreSQL server(s) to be backed up** *(including their version number):*

4. **Where will you host Barman?**
   - [ ] Virtual Machine
   - [ ] Physical Server

5. **Which Linux distribution will you install on the Barman server?**
   - [ ] CentOS 7
   - [ ] RHEL 7
   - [ ] Ubuntu 16.04
   - [ ] Other (please specify): __________

6. **What type of storage will you use?**
   - [ ] Local disks
   - [ ] NFS volume on a VMware virtual machine
   - [ ] Other (please specify): __________

7. **Please provide the required disk space details:**
   - Total size of all databases on the PostgreSQL server:
   - WAL production rate:
   - Frequency of full backups (weekly / daily):
   - Retention policies required by your business (e.g., 4 weeks or last 2 backups):
   - Estimated growth rate of the database:

8. **Which backup method would you prefer to use?**
   - [ ] rsync/ssh
   - [ ] PostgreSQL's `pg_basebackup`

9. **What is your desired Recovery Point Objective (RPO)?**

10. **What is your desired Recovery Time Objective (RTO)?**

11. **Do you plan to relay your files to S3?**

12. **Will you require integration with a monitoring infrastructure?**

13. **Will you have dedicated server(s) for recovery?**

14. **Will you require integration with standby servers?**

---

*Questions? Reach out for further information on Barman and next steps for installation.*
