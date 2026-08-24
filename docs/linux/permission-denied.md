---
title: "Log Shipper Permission Denied"
description: "On-call got paged at 02:14: the log-shipping agent on this box stopped forwarding after last night's cleanup job. The process starts and exits immediately."
tags:
  - "linux"
  - "permissions"
  - "chown"
  - "chmod"
---

# Log Shipper Permission Denied

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

On-call got paged at 02:14: the log-shipping agent on this box stopped forwarding after last night's cleanup job. The process starts and exits immediately.

SSH still works. Start it the same way the box does: `/opt/shipper/run.sh`. Get the agent running again as the user it is supposed to run as.

### Objectives

- The log-shipping agent runs as `shipper` and exits 0
- Its config file is not world-writable

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-005 | **Difficulty:** Easy (100) | **Estimated Time:** 15 minutes | **Focus:** `linux`, `permissions`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/permission-denied?utm_source=challenges&utm_medium=writeup&utm_campaign=permission-denied){ .md-button .md-button--primary }

</div>

---

## Production Context

Cleanup and log-rotate jobs often run as a different user than the service. A recursive chown or a too-tight chmod on a config file is a common reason a process that "starts and dies" shows Permission denied.

---

## Hints

??? tip "Hint 1"
    Run /opt/shipper/run.sh and read the error. Boot also wrote /tmp/shipper-boot.log

??? tip "Hint 2"
    ls -l the paths in the launch script. Compare owner and mode to the user it runs as

??? tip "Hint 3"
    Restore shipper ownership on /opt/shipper. config.yaml needs 0644, directories 0755. Do not chmod 777

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    Last night's cleanup job left `/opt/shipper` owned by user `cleanup` and set `config.yaml` to mode `0000`. The agent runs as `shipper`, so it could not read its config or write its spool.

    ### Diagnostic Steps
    ```bash
    cat /tmp/shipper-boot.log
    /opt/shipper/run.sh
    ls -l /opt/shipper /opt/shipper/config.yaml /opt/shipper/spool
    grep User /opt/shipper/run.sh
    id shipper
    ```

    ### Resolution
    Restore the service user and safe modes. Do not make the config world-writable.
    
    ```bash
    chown -R shipper:shipper /opt/shipper
    chmod 0755 /opt/shipper /opt/shipper/bin /opt/shipper/spool
    chmod 0755 /opt/shipper/bin/shipper /opt/shipper/run.sh
    chmod 0644 /opt/shipper/config.yaml
    /opt/shipper/run.sh
    ```

---

## Learning Points

- `Permission denied` is about the process user, not whether you are root in the shell
- `ls -l` shows owner and mode; compare them to the user in the launch script
- World-writable config is not a fix

---

## Best Practices

- Run cleanup as the service user, or restore ownership before the service starts
- Config files should be owner-readable, not `0000` and not `0777`

---

## References

- https://man7.org/linux/man-pages/man1/chmod.1.html
- https://man7.org/linux/man-pages/man1/chown.1.html

---

<a class="star-cta" href="https://github.com/pagedagain/sre-challenges">Found this useful? <span class="star-cta-link">⭐ Star the repo</span> to help others discover it</a>
