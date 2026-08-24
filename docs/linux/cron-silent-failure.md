---
title: "Silent Cron Failure"
description: "The nightly metrics rollup has not written a heartbeat in three days. App logs are quiet. The crontab looks fine."
tags:
  - "linux"
  - "cron"
  - "PATH"
---

# Silent Cron Failure

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

The nightly metrics rollup has not written a heartbeat in three days. App logs are quiet. The crontab looks fine.

SSH still works. The crontab names a script without a full path. Find where that script lives, then get the job producing today's heartbeat the way cron would run it, not only from an interactive shell.

### Objectives

- The crontab job succeeds in cron's environment
- Today's heartbeat file exists under /var/lib/metrics

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-007 | **Difficulty:** Medium (200) | **Estimated Time:** 20 minutes | **Focus:** `linux`, `cron`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/cron-silent-failure?utm_source=challenges&utm_medium=writeup&utm_campaign=cron-silent-failure){ .md-button .md-button--primary }

</div>

---

## Production Context

Cron does not load `.bashrc` or a login profile. It starts with a stripped environment and a short PATH, usually `/usr/bin:/bin`. A crontab that looks fine in your interactive shell often fails with `command not found` or `Permission denied`, and unless mail is configured the failure is silent.

---

## Hints

??? tip "Hint 1"
    crontab -l. Check whether today's heartbeat exists under /var/lib/metrics

??? tip "Hint 2"
    Cron PATH is not your shell PATH. Run the job with env -i and cron's PATH, not from an interactive shell

??? tip "Hint 3"
    chmod +x /opt/rollup/rollup.sh and put an absolute path (or PATH=) in the crontab. Bare rollup.sh is not on cron's PATH

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    The root crontab called a bare `rollup.sh`, which is not on cron's default PATH (`/usr/bin:/bin`). The script at `/opt/rollup/rollup.sh` was also not executable. The job never wrote `/var/lib/metrics/rollup-YYYY-MM-DD.ok`.

    ### Diagnostic Steps
    `env -i` starts a command with an empty environment. You then pass only the variables cron actually sets. That is how you reproduce a cron failure in an interactive terminal instead of waiting for 02:00.
    
    ```bash
    crontab -l
    ls -l /var/lib/metrics
    find /opt -name 'rollup.sh'
    ls -l /opt/rollup/rollup.sh
    
    # Same PATH cron uses. Bare name: command not found.
    env -i HOME=/root LOGNAME=root PATH=/usr/bin:/bin SHELL=/bin/sh /bin/sh -c 'rollup.sh'
    
    # Absolute path: Permission denied until the script is executable.
    env -i HOME=/root LOGNAME=root PATH=/usr/bin:/bin SHELL=/bin/sh /bin/sh -c '/opt/rollup/rollup.sh'
    ```

    ### Resolution
    Make the script executable and invoke it by absolute path. Putting `PATH=/opt/rollup:/usr/bin:/bin` at the top of the crontab is also valid. Do not wait for 02:00. The checker runs the crontab in cron's environment.
    
    ```bash
    chmod 0755 /opt/rollup/rollup.sh
    crontab - << 'EOF'
    SHELL=/bin/sh
    # Nightly metrics rollup
    0 2 * * * /opt/rollup/rollup.sh
    EOF
    ```

---

## Learning Points

- `env -i` drops your shell's PATH and profile so you see what cron sees
- Cron's PATH is `/usr/bin:/bin` unless the crontab sets `PATH=`
- A script that is not executable fails even with an absolute path
- Running the job in your shell is not the same as running it the way cron does

---

## Best Practices

- Use absolute paths in crontabs, or set `PATH=` at the top of the crontab
- `chmod +x` job scripts, then re-test with `env -i PATH=/usr/bin:/bin`
- Do not rely on `.bashrc`. Cron will not source it
- Capture cron output to a log if you are not reading cron mail

---

## References

- https://man7.org/linux/man-pages/man5/crontab.5.html
- https://man7.org/linux/man-pages/man1/env.1.html
