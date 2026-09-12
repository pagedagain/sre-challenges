---
title: "Active Users Report Undercount"
description: "The monthly active users report has been short by about 30 percent for two quarters. The trend still looked smooth, so nobody caught it until a new analyst compared the report against a raw table coun"
tags:
  - "sql"
  - "reports"
  - "null"
---

# Active Users Report Undercount

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

The monthly active users report has been short by about 30 percent for two quarters. The trend still looked smooth, so nobody caught it until a new analyst compared the report against a raw table count. Finance wants the real number before the board pack goes out.

Connect with `psql -U postgres -d reports`. The report query is `/opt/reports/active_users.sql`. Write a corrected SQL query to `/work/answer.sql`. Re-run that query in `psql` as often as you need before Check.

### Objectives

- `/work/answer.sql` is a SQL query that counts every user who is not cancelled
- Cancelled users stay out of the count. The users table is not rewritten to hide the bug

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-016 | **Difficulty:** Easy (100) | **Estimated Time:** 15 minutes | **Focus:** `sql`, `reports`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/null-surprise?utm_source=challenges&utm_medium=writeup&utm_campaign=null-surprise){ .md-button .md-button--primary }

</div>

---

## Production Context

A status column that is nullable after a migration will silently fall out of every `!=` / `=` filter. The report still runs, the trend still looks smooth, and the miss shows up only when someone compares against `COUNT(*)`.

---

## Hints

??? tip "Hint 1"
    Connect with psql and compare the report count to a plain count of the users table

??? tip "Hint 2"
    Look at distinct status values. A filter that looks right can still drop rows

??? tip "Hint 3"
    Rows with no status are excluded by !=. Count users with IS DISTINCT FROM cancelled, or treat NULL as not cancelled

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    `/opt/reports/active_users.sql` counts with `WHERE status != 'cancelled'`. In SQL, `NULL != 'cancelled'` is unknown, not true, so the 3000 legacy rows with a NULL status are dropped. 6000 active + 3000 unset should be 9000. The report returns 6000.

    ### Diagnostic Steps
    ```bash
    psql -U postgres -d reports
    cat /opt/reports/active_users.sql
    ```
    
    ```sql
    SELECT COUNT(*) FROM users;
    SELECT COUNT(*) FROM users WHERE status != 'cancelled';
    SELECT COUNT(*) FROM users WHERE status IS NULL;
    SELECT DISTINCT status FROM users;
    ```

    ### Resolution
    Write a SQL query that treats NULL as not cancelled:
    
    ```bash
    cat > /work/answer.sql << 'EOF'
    SELECT COUNT(*) FROM users WHERE status IS DISTINCT FROM 'cancelled';
    EOF
    ```
    
    `WHERE status != 'cancelled' OR status IS NULL` and `WHERE COALESCE(status, 'active') != 'cancelled'` also pass. Do not `UPDATE` the NULL rows to dodge the filter.

---

## Learning Points

- `NULL != 'cancelled'` is unknown, so those rows miss a WHERE clause
- `IS DISTINCT FROM` compares NULLs as values
- Changing production rows to make a broken query look right hides the next report that uses the same filter

---

## Best Practices

- Make status NOT NULL with a real default when "unknown" is not a business state
- Prefer `IS DISTINCT FROM` / `IS NOT DISTINCT FROM` when NULL must participate in equality
- Compare report totals against a raw `COUNT(*)` and a breakdown by status, including NULL

---

## References

- https://www.postgresql.org/docs/14/functions-comparison.html

---

<a class="star-cta" href="https://github.com/pagedagain/sre-challenges">Found this useful? <span class="star-cta-link">⭐ Star the repo</span> to help others discover it</a>
