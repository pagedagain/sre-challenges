---
title: "No Space Left After Cleanup"
description: "On-call deleted the orders log to free space after a volume-full alert. du now looks empty, df still says the volume is full, and new writes fail. SSH still works."
tags:
  - "linux"
  - "disk"
  - "lsof"
---

# No Space Left After Cleanup

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

On-call deleted the orders log to free space after a volume-full alert. du now looks empty, df still says the volume is full, and new writes fail. SSH still works.

Get the orders log volume accepting writes again. The logger must still be running.

> Note: `df /` may look fine. Check the orders log mount, then compare it with `du`.

### Objectives

- The orders log volume accepts an 8 MB write
- The orders logger process is still running

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-012 | **Difficulty:** Hard (400) | **Estimated Time:** 25 minutes | **Focus:** `linux`, `disk`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/deleted-file-held-open?utm_source=challenges&utm_medium=writeup&utm_campaign=deleted-file-held-open){ .md-button .md-button--primary }

</div>

---

## Production Context

On-call often deletes a fat log to recover from ENOSPC. If the writer is still running, space does not come back. The usual tell is `df` and `du` disagreeing, plus a `(deleted)` file in `lsof` or `/proc/PID/fd`.

---

## Hints

??? tip "Hint 1"
    df and du on the orders log volume disagree. Writes still fail

??? tip "Hint 2"
    Look for open files with no directory entry. lsof or /proc file descriptors

??? tip "Hint 3"
    Truncate the deleted descriptor under /proc/PID/fd, or restart the logger without the old fd. Killing it and leaving it down is not enough

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    Someone deleted `/var/log/app/orders.log` (~60 MB on a 64 MB volume) while the orders logger still held the file open. Unlink removes the directory entry, so `du` looks empty, but the inode stays allocated until that descriptor closes or is truncated. `df` stays full and writes still fail.

    ### Diagnostic Steps
    ```bash
    df -h /var/log/app
    du -sh /var/log/app
    lsof +L1
    ls -l /proc/$(cat /run/orders-logger.pid)/fd
    ```

    ### Resolution
    Truncate the deleted descriptor. The logger stays up and the volume can accept writes again.
    
    ```bash
    PID=$(cat /run/orders-logger.pid)
    for fd in /proc/"$PID"/fd/*; do
        [ -e "$fd" ] || continue
        link=$(readlink "$fd" || true)
        case "$link" in
            *'(deleted)')
                : > "$fd"
                ;;
        esac
    done
    ```
    
    Restarting the logger without reopening the old descriptor is also a valid fix.

---

## Learning Points

- `unlink` does not free space while a process still holds the file
- `df` counts the inode. `du` only sees directory entries
- Truncating `/proc/PID/fd/N` reclaims space without killing the writer

---

## Best Practices

- Truncate or reopen logs instead of deleting a file a daemon still has open
- Alert when `df` and `du` on the same mount disagree
- After an ENOSPC cleanup, confirm the writer is still healthy and that space actually returned

---

## References

- https://man7.org/linux/man-pages/man5/proc.5.html

---

<a class="star-cta" href="https://github.com/pagedagain/sre-challenges">Found this useful? <span class="star-cta-link">⭐ Star the repo</span> to help others discover it</a>
