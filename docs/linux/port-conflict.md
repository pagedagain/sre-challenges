---
title: "Port Already in Use"
description: "The orders API deploy failed at 01:12. The new build exits immediately: bind: address already in use on port 8080."
tags:
  - "linux"
  - "ports"
  - "ss"
  - "lsof"
---

# Port Already in Use

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

The orders API deploy failed at 01:12. The new build exits immediately: bind: address already in use on port 8080.

SSH still works. Something already holds the port. Get the current release listening on 8080 and stop the leftover from coming back.

### Objectives

- Port 8080 is served by the current API release
- The leftover start script cannot bring the old listener back

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-006 | **Difficulty:** Medium (200) | **Estimated Time:** 20 minutes | **Focus:** `linux`, `ports`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/port-conflict?utm_source=challenges&utm_medium=writeup&utm_campaign=port-conflict){ .md-button .md-button--primary }

</div>

---

## Production Context

Failed deploys often leave the previous process on the port. Killing the PID is not enough if a start script, unit, or supervisor will spawn it again. Confirm what owns the port, then disable the leftover before starting the new release.

---

## Hints

??? tip "Hint 1"
    ss -lntp or lsof -i :8080. The new process cannot bind if something else already holds the port

??? tip "Hint 2"
    Inspect the process before you kill it. tr '\0' ' ' < /proc/<pid>/cmdline and ls -l /proc/<pid>/cwd. A leftover start script next to that tree can bring the old listener back

??? tip "Hint 3"
    Stop the process from /opt/releases/v1, rename its start.sh so it cannot bind 8080 again, then bash /opt/releases/v2/start.sh

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    A leftover `python3 -m http.server 8080` from `/opt/releases/v1` was still bound to 8080. The current orders-api (`/opt/releases/v2/app.py`) could not bind, and `v1/start.sh` was still enabled so the old listener could come back.

    ### Diagnostic Steps
    ```bash
    ss -lntp
    lsof -i :8080
    PID=$(ss -lntp | grep 8080 | grep -oE 'pid=[0-9]+' | head -1 | cut -d= -f2)
    tr '\0' ' ' < /proc/$PID/cmdline
    ls -l /proc/$PID/cwd
    ls /opt/releases /opt/releases/v1 /opt/releases/v2
    cat /opt/releases/v1/start.sh
    cat /opt/releases/v2/start.sh
    ```

    ### Resolution
    Stop the leftover listener, disable its start script, then start the current release. `/opt/releases` is noexec, so invoke the start script with bash.
    
    ```bash
    pkill -f 'python3 -m http.server 8080' || true
    mv /opt/releases/v1/start.sh /opt/releases/v1/start.sh.disabled
    bash /opt/releases/v2/start.sh >/tmp/orders-api.log 2>&1 &
    ss -lntp | grep 8080
    ```

---

## Learning Points

- `ss -lntp` and `lsof -i` map a port to a PID; cmdline and `/proc/PID/cwd` show which release it is
- Address already in use means something is still listening, not that the new binary is broken
- Disable the leftover start path or the old process will return

---

## Best Practices

- Cut over by stopping the old listener and disabling its unit or script before starting the new one
- Check what is on the port after deploy, not only that the new process exited 0

---

## References

- https://man7.org/linux/man-pages/man8/ss.8.html
- https://man7.org/linux/man-pages/man8/lsof.8.html
