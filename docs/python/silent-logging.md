---
title: "Silent Service Logs"
description: "A deploy went out after lunch and the orders worker is still finishing jobs, but the log file has not been touched since. On-call has nothing to go on. SSH still works."
tags:
  - "python"
  - "logging"
  - "handlers"
---

# Silent Service Logs

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

A deploy went out after lunch and the orders worker is still finishing jobs, but the log file has not been touched since. On-call has nothing to go on. SSH still works.

Get the worker logging again so an INFO line from the work path lands in the service log.

### Objectives

- python3 /app/worker.py exits 0
- The service log contains an INFO line for processed order=1001

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-010 | **Difficulty:** Medium (200) | **Estimated Time:** 20 minutes | **Focus:** `python`, `logging`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/silent-logging?utm_source=challenges&utm_medium=writeup&utm_campaign=silent-logging){ .md-button .md-button--primary }

</div>

---

## Production Context

Silent logs after a deploy are usually more than one mistake. A leftover `CRITICAL` from a debug session, a log directory that was never created on the new host, and a `basicConfig` call that runs too late will each hide the others. Read the setup, then check the directory.

---

## Hints

??? tip "Hint 1"
    Run the worker under /app in the foreground. It exits, but stdout is empty

??? tip "Hint 2"
    Read the logging setup. Check the level, the file path, and what runs after the handler is attached

??? tip "Hint 3"
    Level is CRITICAL, /var/log/svc/app is missing, and basicConfig(force=True) after addHandler replaces the FileHandler. Set INFO, mkdir, remove that basicConfig

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    Three layered logging bugs landed in the same deploy. The root logger level is `CRITICAL`, so INFO is dropped. `FileHandler` writes `/var/log/svc/app/worker.log` but `/var/log/svc/app` does not exist. `logging.basicConfig(..., force=True)` runs after `addHandler` and replaces the file handler.

    ### Diagnostic Steps
    ```bash
    python3 /app/worker.py
    cat /app/worker.py
    ls -la /var/log/svc
    ls -la /var/log/svc/app
    ```

    ### Resolution
    Create the log directory, set the logger to INFO, and remove the `basicConfig` call that wipes the file handler.
    
    ```bash
    mkdir -p /var/log/svc/app
    sed -i 's/logging.CRITICAL/logging.INFO/g' /app/worker.py
    sed -i '/basicConfig/d' /app/worker.py
    ```

---

## Learning Points

- Logger level filters before any handler runs. `CRITICAL` hides INFO even when the handler is correct
- `FileHandler` does not create missing parent directories
- `basicConfig(force=True)` after `addHandler` replaces existing handlers

---

## Best Practices

- Create the log directory in the same change that points a `FileHandler` at it
- Configure logging once, in one place, before the first emit
- Do not call `basicConfig` after handlers are already attached

---

## References

- https://docs.python.org/3/library/logging.html#logging.basicConfig
