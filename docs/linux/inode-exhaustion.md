---
title: "No Space Left With Disk Free"
description: "The orders cache volume started returning \"No space left on device\" after a traffic spike. df -h still shows more than half the volume free. New session files fail anyway. SSH still works."
tags:
  - "linux"
  - "disk"
  - "cache"
---

# No Space Left With Disk Free

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

The orders cache volume started returning "No space left on device" after a traffic spike. df -h still shows more than half the volume free. New session files fail anyway. SSH still works.

Get the cache accepting new sessions again without wiping the volume's quota marker. Retry `check-mounts` in the shell as often as you need before Check.

> Note: `df -h` on this mount can look healthy while creates still fail.

### Objectives

- `check-mounts` exits 0
- The quota marker file is still present

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-013 | **Difficulty:** Hard (400) | **Estimated Time:** 25 minutes | **Focus:** `linux`, `disk`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/inode-exhaustion?utm_source=challenges&utm_medium=writeup&utm_campaign=inode-exhaustion){ .md-button .md-button--primary }

</div>

---

## Production Context

Caches and mail spools often run out of inodes before they run out of bytes. Thousands of small files look cheap on `df -h` and still block every create. Pair `df -h` with `df -i` before you add disks.

---

## Hints

??? tip "Hint 1"
    Byte usage can look fine while new files still fail. There is another kind of capacity besides space

??? tip "Hint 2"
    Check whether the volume is out of inodes. Then find the directory packed with tiny files

??? tip "Hint 3"
    Old session files under /var/spool/appcache/sessions used every inode. Remove those files. Keep the quota marker

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    A runaway session cache filled `/var/spool/appcache/sessions` with thousands of tiny files. The mount has a 48 MB cap and only 4k inodes, so inode use hit 100% while `df -h` still showed free space. Creates fail with ENOSPC until those files are removed. The quota marker `QUOTA` must stay.

    ### Diagnostic Steps
    ```bash
    df -h /var/spool/appcache
    df -i /var/spool/appcache
    find /var/spool/appcache -xdev -type f | wc -l
    ls -la /var/spool/appcache
    ```

    ### Resolution
    Prune session files older than one day. Leave the quota marker in place.
    
    ```bash
    find /var/spool/appcache/sessions -type f -mtime +1 -delete
    ```

---

## Learning Points

- ENOSPC is not only a full block count. A filesystem can be out of inodes
- `df -h` and `df -i` answer different questions
- Clearing a cache directory is not the same as wiping the mount

---

## Best Practices

- Alert on inode percent as well as byte percent for caches and mail queues
- Cap session files by age and count, not only by size
- Keep marker files outside the directory you prune

---

## References

- https://man7.org/linux/man-pages/man1/df.1.html

---

<a class="star-cta" href="https://github.com/pagedagain/sre-challenges">Found this useful? <span class="star-cta-link">⭐ Star the repo</span> to help others discover it</a>
