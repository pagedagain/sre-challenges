---
title: "Disk Full Emergency"
description: "Your monitoring alerts fired at 3 AM: the app cannot rotate logs and is throwing \"No space left on device\" for its log volume path. SSH still works."
tags:
  - "linux"
  - "disk"
  - "df"
  - "du"
---

# Disk Full Emergency

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

Your monitoring alerts fired at 3 AM: the app cannot rotate logs and is throwing "No space left on device" for its log volume path. SSH still works.

A runaway process dumped large junk files somewhere under `/var/log`. Find that bloat and reclaim the space without deleting critical service files.

> Note: On shared container hosts, `df -h /` may still look fine. Use `du` to find what is actually consuming space under `/var/log`.

### Objectives

- Find and reclaim the runaway log bloat under `/var/log` (remove that directory, or bring it under 50MB)
- Keep `/var/log/important-service.log` intact

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-001 | **Difficulty:** Easy (100) | **Estimated Time:** 15 minutes | **Focus:** `linux`, `disk`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/disk-full){ .md-button .md-button--primary }

</div>

---

## Production Context

Disk incidents are not always "df shows 100%". On large shared volumes, a single path can fill an application budget or inode quota while `df /` still looks healthy. Always pair `df` with `du`.

---

## Hints

??? tip "Hint 1"
    Root df may look fine. Check directory sizes with du

??? tip "Hint 2"
    Try: du -sh /var/log/* | sort -h

??? tip "Hint 3"
    Large junk lives under /var/log/incidentforge-fill/. Clean it without deleting important-service.log

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    A runaway process filled `/var/log/incidentforge-fill/` with large junk files (~90MB on a 96MB log volume). The app could not write its logs to that area even when overall root disk still had free space.

    ### Diagnostic Steps
    ```bash
    df -h /
    df -h /var/log
    du -sh /var/log/*
    du -sh /var/log/incidentforge-fill/
    ls -lh /var/log/incidentforge-fill/
    ls -l /var/log/important-service.log
    ```

    ### Resolution
    Remove the fill files or the entire directory, keep the important log. The checker passes when that directory is gone or under 50MB:
    
    ```bash
    rm -rf /var/log/incidentforge-fill/
    # verify:
    du -sh /var/log/incidentforge-fill/ 2>/dev/null || echo "fill dir gone"
    test -f /var/log/important-service.log && echo "important log ok"
    ```

---

## Learning Points

- `df` shows filesystem totals; `du` finds what is using space in a path
- Clean the bloat path; do not blindly `rm -rf /var/log`
- Preserve known-critical files during cleanup

---

## Best Practices

- Alert on path-level growth for app log dirs, not only root df %
- Log rotation and retention policies
- Separate volumes for app logs when possible

---

## References

- `man df`, `man du`
