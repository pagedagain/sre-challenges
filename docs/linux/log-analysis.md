---
title: "Suspicious Traffic Spike"
description: "Security flagged a traffic spike on your web tier. The WAF blocked some of the abusive requests, but you still need to identify which client IP generated the most requests during the incident window."
tags:
  - "linux"
  - "logs"
  - "awk"
  - "grep"
---

# Suspicious Traffic Spike

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

Security flagged a traffic spike on your web tier. The WAF blocked some of the abusive requests, but you still need to identify which client IP generated the most requests during the incident window.

An nginx access log from the affected period has been preserved on the host. Find the top offending IP and record it for the incident report.

### Objectives

- Identify the IP address with the most requests
- Write the correct IP to /work/answer.txt

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-004 | **Difficulty:** Medium (200) | **Estimated Time:** 20 minutes | **Focus:** `linux`, `logs`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/log-analysis?utm_source=challenges&utm_medium=writeup&utm_campaign=log-analysis){ .md-button .md-button--primary }

</div>

---

## Production Context

Log analysis is a core SRE skill during security incidents, capacity events, and outage investigations.

---

## Hints

??? tip "Hint 1"
    The access log is at /var/log/nginx/access.log

??? tip "Hint 2"
    Try: sort | uniq -c | sort -rn to count occurrences

??? tip "Hint 3"
    The client IP is the first field: awk '{print $1}' /var/log/nginx/access.log

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    A single IP (203.0.113.42) generated 500 requests (400 served, 100 blocked by the WAF). That is 10x more than any other client, so it looks like a scanner or a misconfigured integration.

    ### Diagnostic Steps
    ```bash
    awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -5
    echo "203.0.113.42" > /work/answer.txt
    ```

    ### Resolution
    Find the most frequent IP and write it to `/work/answer.txt`.

---

## Learning Points

- `awk '{print $1}'` extracts the client IP from nginx combined log format
- `sort | uniq -c | sort -rn` is the classic log aggregation pattern

---

## Best Practices

- Automate top-N IP analysis in runbooks
- Correlate with WAF and CDN logs
- Rate-limit or block abusive IPs at the edge

---

## References

- nginx log format documentation
