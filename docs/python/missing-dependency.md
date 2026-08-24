---
title: "Missing Python Dependency"
description: "The deployment pipeline completed successfully, but the automation script fails at runtime with a Python import error. The container runs with `--network none` for security, so you cannot reach PyPI."
tags:
  - "python"
  - "pip"
  - "dependencies"
---

# Missing Python Dependency

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

The deployment pipeline completed successfully, but the automation script fails at runtime with a Python import error. The container runs with `--network none` for security, so you cannot reach PyPI.

Get `app.py` running by installing the missing dependency from resources available inside the container.

### Objectives

- Python script runs and exits 0

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-002 | **Difficulty:** Easy (100) | **Estimated Time:** 15 minutes | **Focus:** `python`, `pip`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/missing-dependency?utm_source=challenges&utm_medium=writeup&utm_campaign=missing-dependency){ .md-button .md-button--primary }

</div>

---

## Production Context

Air-gapped or network-restricted environments require offline package installation. Production containers often run with no egress.

---

## Hints

??? tip "Hint 1"
    Run `python3 app.py` and read the error message

??? tip "Hint 2"
    This container has no network access. PyPI is unreachable

??? tip "Hint 3"
    Check /opt/packages/ for offline install options

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    The `requests` library was not installed in the runtime container. The deployment assumed dependencies were pre-installed but they were missing.

    ### Diagnostic Steps
    ```bash
    python3 app.py
    ls /opt/packages/
    pip3 install --no-index --find-links=/opt/packages requests==2.31.0
    ```

    ### Resolution
    Install from the local wheel:
    
    ```bash
    pip3 install --no-index --find-links=/opt/packages requests==2.31.0
    python3 app.py
    ```

---

## Learning Points

- `ModuleNotFoundError` means the package isn't installed
- `--network none` blocks PyPI; use local wheels
- `pip install --no-index --find-links` for offline installs

---

## Best Practices

- Bake dependencies into images during CI
- Keep offline package mirrors for air-gapped deploys
- Pin versions in requirements.txt

---

## References

- pip offline installation docs
