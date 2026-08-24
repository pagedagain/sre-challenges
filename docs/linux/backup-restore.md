---
title: "Restore from Backup"
description: "A bad migration wiped the appd config directory overnight. The service will not start without it. Three nightly archives are on the box."
tags:
  - "linux"
  - "tar"
  - "backups"
  - "checksums"
---

# Restore from Backup

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

A bad migration wiped the appd config directory overnight. The service will not start without it. Three nightly archives are on the box.

SSH still works. Restore config from the most recent good backup. An archive that does not verify is not a restore source.

### Objectives

- /etc/appd has the current good config (listen 8443, cluster replicas 3)
- The damaged archive was not used as the restore source

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-008 | **Difficulty:** Medium (200) | **Estimated Time:** 20 minutes | **Focus:** `linux`, `tar`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/backup-restore?utm_source=challenges&utm_medium=writeup&utm_campaign=backup-restore){ .md-button .md-button--primary }

</div>

---

## Production Context

A backup you have not listed and checksummed is not a restore source. Nightly jobs can write a short or corrupt file and still leave a `.tar.gz` name. Restore the newest archive that verifies, not the newest filename.

---

## Hints

??? tip "Hint 1"
    Backups are under /var/backups. Use tar to list each archive. The newest may not list cleanly

??? tip "Hint 2"
    Check the archives against SHA256SUMS. Restore the newest one that verifies

??? tip "Hint 3"
    Restore from appd-2026-08-21.tar.gz under /var/backups. The 22nd archive is truncated

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    A migration emptied `/etc/appd`. Of the three nightly archives in `/var/backups`, `appd-2026-08-22.tar.gz` is truncated and does not match `SHA256SUMS`. The newest good archive is `appd-2026-08-21.tar.gz`. The 20th archive is an older listen port.

    ### Diagnostic Steps
    ```bash
    ls -l /var/backups
    ls -l /etc/appd
    (cd /var/backups && sha256sum -c SHA256SUMS)
    tar -tzf /var/backups/appd-2026-08-20.tar.gz
    tar -tzf /var/backups/appd-2026-08-21.tar.gz
    tar -tzf /var/backups/appd-2026-08-22.tar.gz
    ```

    ### Resolution
    Extract the newest archive that verifies onto `/`. Do not restore the truncated 22nd archive.
    
    ```bash
    tar -C / -xzf /var/backups/appd-2026-08-21.tar.gz
    rm -f /etc/appd/.migrate-wip
    ```

---

## Learning Points

- `tar -tzf` lists an archive without writing files; a truncated gzip fails here
- `sha256sum -c SHA256SUMS` tells you which nightly file still matches
- An unverified backup is not a backup

---

## Best Practices

- Checksum archives as they are written, and check again before restore
- Keep more than one generation so a bad newest file is not the only copy
- Restore to a known path and confirm required files and hashes

---

## References

- https://man7.org/linux/man-pages/man1/tar.1.html
- https://man7.org/linux/man-pages/man1/sha256sum.1.html
