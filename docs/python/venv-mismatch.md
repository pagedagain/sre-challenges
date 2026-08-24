---
title: "Invoices Job Import Error"
description: "The nightly invoices job dies with ModuleNotFoundError after a host rebuild. On-call was told the dependency is installed. The engineer who set this box up has left. SSH still works."
tags:
  - "python"
  - "venv"
  - "sys.path"
---

# Invoices Job Import Error

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

The nightly invoices job dies with ModuleNotFoundError after a host rebuild. On-call was told the dependency is installed. The engineer who set this box up has left. SSH still works.

Get the nightly wrapper running again so it writes a complete invoice report.

### Objectives

- /opt/app/run-invoices.sh exits 0
- /opt/app/invoices.txt records Invoice 4401 total 82.50

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-011 | **Difficulty:** Medium (200) | **Estimated Time:** 20 minutes | **Focus:** `python`, `venv`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/venv-mismatch?utm_source=challenges&utm_medium=writeup&utm_campaign=venv-mismatch){ .md-button .md-button--primary }

</div>

---

## Production Context

A host rebuild often leaves a venv in place while cron wrappers and shebangs keep pointing at `/usr/bin/python3`. `pip list` inside the venv looks healthy. The job still dies because it never uses that interpreter.

---

## Hints

??? tip "Hint 1"
    Run the nightly wrapper under /opt/app and read the ModuleNotFoundError

??? tip "Hint 2"
    Compare the interpreter the wrapper calls with which python3, sys.path, and pip list. Look under /opt

??? tip "Hint 3"
    Wrapper and shebang call /usr/bin/python3. The package is in /opt/venv. Point the wrapper at /opt/venv/bin/python3

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    The invoices library is installed in a virtualenv at `/opt/venv`. The nightly wrapper invokes `/usr/bin/python3`, and `invoices.py` still has a `/usr/bin/python3` shebang, so the system interpreter raises `ModuleNotFoundError`.

    ### Diagnostic Steps
    ```bash
    /opt/app/run-invoices.sh
    cat /opt/app/run-invoices.sh
    head -1 /opt/app/invoices.py
    which python3
    /usr/bin/python3 -c 'import sys; print(sys.path)'
    ls /opt/venv/lib/*/site-packages
    /opt/venv/bin/pip list
    ```

    ### Resolution
    Point the nightly wrapper at the virtualenv interpreter.
    
    ```bash
    sed -i 's|/usr/bin/python3|/opt/venv/bin/python3|' /opt/app/run-invoices.sh
    ```

---

## Learning Points

- `pip list` reports whatever environment is active. The wrapper may call a different Python
- A shebang is ignored when the wrapper names an interpreter explicitly
- Installing the package onto the system interpreter hides the mismatch

---

## Best Practices

- Cron and service wrappers should call the venv binary, not `/usr/bin/python3`
- Keep shebangs and wrappers aligned with the environment that has the dependencies
- Prefer `#!/usr/bin/env python3` only after PATH selects the venv

---

## References

- https://docs.python.org/3/library/venv.html

---

<a class="star-cta" href="https://github.com/pagedagain/sre-challenges">Found this useful? <span class="star-cta-link">⭐ Star the repo</span> to help others discover it</a>
