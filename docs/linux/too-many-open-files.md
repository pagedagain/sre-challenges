---
title: "Too Many Open Files"
description: "The orders worker processed fine for days, then every new job started failing with \"Too many open files\". A restart last night bought about a day. The same failures are back. SSH still works."
tags:
  - "linux"
  - "worker"
  - "logs"
---

# Too Many Open Files

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

The orders worker processed fine for days, then every new job started failing with "Too many open files". A restart last night bought about a day. The same failures are back. SSH still works.

Get the worker through a burst of jobs. Retry `run-burst` in the shell as often as you need before Check.

### Objectives

- `run-burst` exits 0
- Forty job logs record processed work

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-014 | **Difficulty:** Hard (400) | **Estimated Time:** 25 minutes | **Focus:** `linux`, `worker`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/too-many-open-files?utm_source=challenges&utm_medium=writeup&utm_campaign=too-many-open-files){ .md-button .md-button--primary }

</div>

---

## Production Context

A worker that opens a file per job and never closes it will eventually hit its file-descriptor limit. Restarting clears the descriptors, so the service looks fine for a while. Raising `ulimit` only delays the same failure.

---

## Hints

??? tip "Hint 1"
    The orders worker is still running. Watch how many files it has open

??? tip "Hint 2"
    Compare that count with the process limits. See whether each job leaves a file handle behind

??? tip "Hint 3"
    Each job in /opt/worker/orders-worker opens a log and never closes it. Close the descriptor in the loop. Restarting without that patch will fail again under load

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    `/opt/worker/orders-worker` opens a per-job log with `exec {fd}>>` and never closes it. The worker and `run-burst` run under `ulimit -n 24`, so a burst of 40 jobs hits "Too many open files". Restarting drops the leaked descriptors until the loop fills them again.

    ### Diagnostic Steps
    ```bash
    pid=$(cat /run/orders-worker.pid)
    lsof -p $pid
    ls /proc/$pid/fd | wc -l
    cat /proc/$pid/limits
    cat /opt/worker/orders-worker
    ```

    ### Resolution
    In `/opt/worker/orders-worker`, `process_one` opens a log and writes to it, but never closes the descriptor. Add `exec {fd}>&-` after the echo. That closes the fd that `{fd}>>` opened.
    
    ```bash
    process_one() {
        local job=$1
        exec {fd}>>"${OUT}/${job}.log"
        echo "processed ${job}" >&$fd
        exec {fd}>&-
    }
    ```

---

## Learning Points

- EMFILE means this process hit its own file limit, not that the host is out of files
- If `/proc/PID/fd` keeps growing, something is opening files and not closing them
- Restarting or raising `ulimit` only postpones the leak

---

## Best Practices

- Close every descriptor you open in a loop, or use a context manager
- Load-test workers under a low `ulimit -n` in staging
- Alert when a process fd count trends toward its limit

---

## References

- https://man7.org/linux/man-pages/man2/getrlimit.2.html

---

<a class="star-cta" href="https://github.com/pagedagain/sre-challenges">Found this useful? <span class="star-cta-link">⭐ Star the repo</span> to help others discover it</a>
