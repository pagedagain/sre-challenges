---
title: "Postgres Cluster Will Not Start"
description: "Postgres will not start. The shop database went with it. Nightly dumps are on the box."
tags:
  - "sql"
  - "recovery"
  - "backups"
---

# Postgres Cluster Will Not Start

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

Postgres will not start. The shop database went with it. Nightly dumps are on the box.

There is no systemd. Use `pg_ctlcluster` with `--skip-systemctl-redirect`:

```bash
pg_lsclusters
pg_ctlcluster --skip-systemctl-redirect 14 main start
pg_ctlcluster --skip-systemctl-redirect 14 main stop
ls /var/log/postgresql
```

Try `psql -U postgres -d shop` when the cluster is up. Fix the cluster so that connection works and the dumped orders are back. Re-run `psql` as often as you need before Check.

### Objectives

- Postgres accepts local connections
- The shop database has the dumped orders. The nightly dump file is still on the box

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-020 | **Difficulty:** Hard (400) | **Estimated Time:** 25 minutes | **Focus:** `sql`, `recovery`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/corrupt-datadir?utm_source=challenges&utm_medium=writeup&utm_campaign=corrupt-datadir){ .md-button .md-button--primary }

</div>

---

## Production Context

A crashed disk, a bad copy, or a truncated control file takes the whole instance down. `pg_resetwal` can make Postgres start and still serve garbage. The recovery path is a new data directory plus a verified logical dump (or a physical base backup, which this box does not have).

---

## Hints

??? tip "Hint 1"
    pg_ctlcluster --skip-systemctl-redirect 14 main start, then read the fatal line in /var/log/postgresql. pg_lsclusters shows the data directory

??? tip "Hint 2"
    Nightly dumps live under /var/backups. The data directory is still there but the control file is not usable

??? tip "Hint 3"
    Wipe the data directory, initdb as postgres with data-checksums and wal-segsize=8, start the cluster, restore /var/backups/postgresql/shop.sql. Do not pg_resetwal

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    `global/pg_control` in `/var/lib/postgresql/14/main` was overwritten. `pg_ctlcluster` fails at start (incorrect checksum in the control file). `/etc/postgresql/14/main` is intact, so the cluster definition is still there. A logical dump of the `shop` database is at `/var/backups/postgresql/shop.sql` (5000 `orders` rows, including id 4242 sku `FORGE-1`).

    ### Diagnostic Steps
    ```bash
    pg_lsclusters
    pg_ctlcluster --skip-systemctl-redirect 14 main start
    ls /var/log/postgresql
    ls /var/lib/postgresql/14/main/global
    ls /var/backups/postgresql
    head /var/backups/postgresql/shop.sql
    ```

    ### Resolution
    Do not `pg_dropcluster`. The cluster definition under `/etc/postgresql` is still valid. Rebuild only the data directory, then restore the dump:
    
    ```bash
    pg_ctlcluster --skip-systemctl-redirect 14 main stop || true
    rm -rf /var/lib/postgresql/14/main
    mkdir -p /var/lib/postgresql/14/main
    chown -R postgres:postgres /var/lib/postgresql
    chmod 700 /var/lib/postgresql/14/main
    su -s /bin/bash postgres -c '/usr/lib/postgresql/14/bin/initdb -D /var/lib/postgresql/14/main --data-checksums --wal-segsize=8 --encoding=UTF8 --locale=en_US.UTF-8'
    pg_ctlcluster --skip-systemctl-redirect 14 main start
    psql -U postgres -c "CREATE DATABASE shop;"
    psql -U postgres -d shop -f /var/backups/postgresql/shop.sql
    ```
    
    `--wal-segsize=8` keeps WAL inside the tmpfs. Default 16 MB segments can fill the data directory. `pg_resetwal` is not a restore.

---

## Learning Points

- A damaged control file stops the cluster before any SQL runs
- Debian keeps config under `/etc/postgresql` and data under `/var/lib/postgresql`. Rebuild data, not the cluster definition
- `initdb` plus `pg_dump` restore is the logical recovery path. WAL replay needs a base backup taken with `wal_level=replica`

---

## Best Practices

- Take logical dumps (and/or base backups) off the data disk
- Never `pg_resetwal` on a cluster you still plan to serve
- Run `initdb` as `postgres` with the same checksum and WAL segment size the host was built with

---

## References

- https://www.postgresql.org/docs/14/app-initdb.html
- https://www.postgresql.org/docs/14/app-pgdump.html
- https://www.postgresql.org/docs/14/app-pgresetwal.html

---

<a class="star-cta" href="https://github.com/pagedagain/sre-challenges">Found this useful? <span class="star-cta-link">⭐ Star the repo</span> to help others discover it</a>
