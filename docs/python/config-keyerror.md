---
title: "Mismatched Config Key"
description: "A deploy went out an hour ago and the orders batch job now crashes on startup. The traceback bottoms out in a config loader instead of pointing at a missing setting. SSH still works."
tags:
  - "python"
  - "traceback"
  - "config"
---

# Mismatched Config Key

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

A deploy went out an hour ago and the orders batch job now crashes on startup. The traceback bottoms out in a config loader instead of pointing at a missing setting. SSH still works.

Get the job running again so it writes a complete run log.

### Objectives

- python3 /app/job.py exits 0
- /app/run.log records the configured timeout (30)

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-009 | **Difficulty:** Medium (200) | **Estimated Time:** 20 minutes | **Focus:** `python`, `traceback`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/config-keyerror?utm_source=challenges&utm_medium=writeup&utm_campaign=config-keyerror){ .md-button .md-button--primary }

</div>

---

## Production Context

Config renames in a deploy often land in YAML first. A helper that swallows the original exception makes the crash look like a loader bug. Read past the top frame, then diff the lookup against the file.

---

## Hints

??? tip "Hint 1"
    Run the batch job under /app and read the traceback past the first frame

??? tip "Hint 2"
    The loader wraps the original error. Compare the lookup in the loader with the file under /etc/batchjob

??? tip "Hint 3"
    Code still reads db.timeout_seconds. config.yaml renamed it to db.timeout_s. Align them and keep the timeout at 30

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    A deploy renamed `db.timeout_seconds` to `db.timeout_s` in `/etc/batchjob/config.yaml`. The loader still looks up `timeout_seconds` and wraps `KeyError` with `from None`, so the traceback shows `ConfigError: required setting missing` instead of the key.

    ### Diagnostic Steps
    ```bash
    python3 /app/job.py
    cat /app/config_loader.py
    cat /app/job.py
    cat /etc/batchjob/config.yaml
    ```

    ### Resolution
    Add the key the loader still reads. Keep the timeout at 30.
    
    ```bash
    cat > /etc/batchjob/config.yaml << 'EOF'
    db:
      host: 127.0.0.1
      timeout_s: 30
      timeout_seconds: 30
    EOF
    ```

---

## Learning Points

- `raise ... from None` hides the original `KeyError`, including the key name
- The next frame up still shows the lookup. Compare that with the config file
- Aligning either side is a valid fix. The run log must still show timeout 30

---

## Best Practices

- Rename keys in config and code in the same change, or accept both names during a transition
- Do not swallow `KeyError` without including the missing key in the new error
- Assert configured values in the run log so a stubbed default cannot hide a mismatch

---

## References

- https://docs.python.org/3/tutorial/errors.html#exception-chaining

---

<a class="star-cta" href="https://github.com/pagedagain/sre-challenges">Found this useful? <span class="star-cta-link">⭐ Star the repo</span> to help others discover it</a>
