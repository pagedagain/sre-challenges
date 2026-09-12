---
title: "Customer Search Slow After Config Change"
description: "A customer search that responded in 20ms now takes 12 seconds after a config change that only changed how the parameter is passed. The table has not grown."
tags:
  - "sql"
  - "explain"
  - "indexes"
---

# Customer Search Slow After Config Change

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

A customer search that responded in 20ms now takes 12 seconds after a config change that only changed how the parameter is passed. The table has not grown.

Connect with `psql -U postgres -d crm`. The lookup is `/opt/reports/customer_lookup.sql`. Write a corrected SQL query to `/work/answer.sql`. Re-run that query in `psql` as often as you need before Check. Use `EXPLAIN` (not `EXPLAIN ANALYZE`) to inspect the plan.

### Objectives

- `/work/answer.sql` is a SQL query that returns the customer for the lookup in `/opt/reports/customer_lookup.sql`
- The customers table schema and stored phone numbers are unchanged

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-019 | **Difficulty:** Easy (100) | **Estimated Time:** 15 minutes | **Focus:** `sql`, `explain`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/index-not-used?utm_source=challenges&utm_medium=writeup&utm_campaign=index-not-used){ .md-button .md-button--primary }

</div>

---

## Production Context

A driver or config change that sends an integer is often "fixed" by casting the column to match. The lookup returns the right row. Latency is the only symptom.

---

## Hints

??? tip "Hint 1"
    Run EXPLAIN on the lookup. Note the scan type. Do not wait on EXPLAIN ANALYZE

??? tip "Hint 2"
    Check the phone_number column type and which side of the comparison is being cast

??? tip "Hint 3"
    phone_number is varchar. Casting the column to integer skips the btree. Compare it to a quoted string. Do not add a cast index or turn seqscan off

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    `/opt/reports/customer_lookup.sql` compares with `phone_number::integer = 2125551234`. PostgreSQL 8.3 dropped implicit `varchar = integer`, so an unquoted integer would error. Casting the column makes the query run, but the filter is `(phone_number)::integer = 2125551234` and `customers_phone_number_idx` (btree on the stored strings) cannot be used. The table has 500,000 rows. The matching customer is id 4242. Many numbers are stored with a leading zero, so changing the column to integer would also lose data.

    ### Diagnostic Steps
    ```bash
    psql -U postgres -d crm
    cat /opt/reports/customer_lookup.sql
    ```
    
    ```sql
    EXPLAIN SELECT id, name, phone_number FROM customers WHERE phone_number::integer = 2125551234;
    EXPLAIN SELECT id, name, phone_number FROM customers WHERE phone_number = '2125551234';
    \d customers
    SELECT phone_number FROM customers WHERE id = 1;
    ```

    ### Resolution
    Compare the varchar column to a string so the btree can be used:
    
    ```bash
    cat > /work/answer.sql << 'EOF'
    SELECT id, name, phone_number
    FROM customers
    WHERE phone_number = '2125551234';
    EOF
    ```
    
    Do not add an expression index on the cast, SET `enable_seqscan = off`, or change `phone_number` to integer.

---

## Learning Points

- `EXPLAIN` (not `EXPLAIN ANALYZE`) shows Seq Scan vs Index Scan without waiting on the slow plan
- Casting a varchar column to integer for comparison disables the btree on the stored text
- The parameter should match the column type, not the other way around

---

## Best Practices

- Bind search parameters as text when the column is text, even if the value looks numeric
- Keep leading zeros in identifiers (phone, zip, account) as varchar
- After a type-error "fix" that adds a column cast, compare `EXPLAIN` before and after

---

## References

- https://www.postgresql.org/docs/14/using-explain.html
- https://www.postgresql.org/docs/14/typeconv-union-case.html
- https://www.postgresql.org/docs/14/indexes-types.html

---

<a class="star-cta" href="https://github.com/pagedagain/sre-challenges">Found this useful? <span class="star-cta-link">⭐ Star the repo</span> to help others discover it</a>
