---
title: "Daily Revenue Doubled Overnight"
description: "The daily revenue report to the CEO doubled overnight. Gateway settlement still matches the orders table. The report query has not changed in two years."
tags:
  - "sql"
  - "joins"
  - "duplicates"
---

# Daily Revenue Doubled Overnight

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

The daily revenue report to the CEO doubled overnight. Gateway settlement still matches the orders table. The report query has not changed in two years.

Connect with `psql -U postgres -d ecommerce`. The report is `/opt/reports/daily_revenue.sql`. Fix the data so that report's revenue adds up to the gateway (orders-only) total. Re-run the report in `psql` as often as you need before Check.

### Objectives

- `/opt/reports/daily_revenue.sql` matches the orders-only total
- Original product rows are kept. The same product_id cannot land twice

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-018 | **Difficulty:** Hard (400) | **Estimated Time:** 25 minutes | **Focus:** `sql`, `joins`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/wrong-join-fanout?utm_source=challenges&utm_medium=writeup&utm_campaign=wrong-join-fanout){ .md-button .md-button--primary }

</div>

---

## Production Context

Dimension tables that are upserted by `product_id` without a unique constraint silently grow duplicates. Revenue-by-SKU reports then fan out overnight and look like a traffic spike.

---

## Hints

??? tip "Hint 1"
    Run the product revenue report. Sum its revenue column and compare to SUM(amount) from orders with no join. Then pick one product_id and compare that line to its orders-only sum

??? tip "Hint 2"
    For a product_id whose report line is high, count how many rows it has in products and look at sync_id

??? tip "Hint 3"
    Nightly ETL duplicated products (same product_id, different sync_id). Delete extra rows, keep MIN(sync_id) per product_id, and add a unique constraint on product_id. DISTINCT on the report is not the fix

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    A nightly ETL sync inserted extra `products` rows for 200 SKUs (`product_id` 801-1000), same `product_id`, new `sync_id`. Twenty of those SKUs got a third copy. `/opt/reports/daily_revenue.sql` joins `orders` to `products` for `product_id` and `name`, so every order for a duplicated SKU is counted once per copy. The query did not change. `orders` has 50,000 rows; the gateway total is `SUM(amount)` with no join.

    ### Diagnostic Steps
    ```bash
    psql -U postgres -d ecommerce
    cat /opt/reports/daily_revenue.sql
    ```
    
    ```sql
    SELECT SUM(amount) FROM orders;
    SELECT SUM(revenue) FROM (
        SELECT SUM(o.amount) AS revenue
        FROM orders o
        JOIN products p ON o.product_id = p.product_id
        GROUP BY (o.created_at)::date, p.product_id, p.name
    ) r;
    SELECT product_id, COUNT(*), MIN(sync_id), MAX(sync_id)
    FROM products
    GROUP BY product_id
    HAVING COUNT(*) > 1;
    SELECT product_id, sync_id, created_at FROM products WHERE product_id IN (801, 981) ORDER BY 1, 2;
    ```

    ### Resolution
    Keep the lowest `sync_id` per `product_id` (the original row), then prevent the next sync from inserting another copy:
    
    ```bash
    psql -U postgres -d ecommerce -v ON_ERROR_STOP=1 <<'EOF'
    DELETE FROM products
    WHERE (product_id, sync_id) NOT IN (
        SELECT product_id, MIN(sync_id)
        FROM products
        GROUP BY product_id
    );
    ALTER TABLE products DROP CONSTRAINT IF EXISTS products_product_id_unique;
    ALTER TABLE products ADD CONSTRAINT products_product_id_unique UNIQUE (product_id);
    EOF
    ```
    
    `MIN(sync_id)` is per `product_id`. SKUs with copies 2, 3, and 4 keep 2, not every row. `DISTINCT` on the report hides the fan-out for this one query. Deleting every copy of a duplicated SKU and re-inserting drops the original `created_at`. `DELETE ... WHERE sync_id = 2` leaves the third copies.

---

## Learning Points

- A join on a non-unique key multiplies rows. The query can be unchanged and still be wrong
- Compare the report total to an orders-only aggregate before rewriting the SELECT
- Deduplicate the dimension table and add a unique constraint so ETL cannot recur

---

## Best Practices

- Unique constraints on natural keys (`product_id`) belong on dimension tables, not only in the ETL job
- Prefer `INSERT ... ON CONFLICT` (or equivalent) over blind inserts into lookup tables
- Collapse duplicates with `MIN(sync_id)` (or another business key) grouped by `product_id`, not a hardcoded copy count

---

## References

- https://www.postgresql.org/docs/14/queries-table-expressions.html#QUERIES-JOIN
- https://www.postgresql.org/docs/14/ddl-constraints.html#DDL-CONSTRAINTS-UNIQUE-CONSTRAINTS

---

<a class="star-cta" href="https://github.com/pagedagain/sre-challenges">Found this useful? <span class="star-cta-link">⭐ Star the repo</span> to help others discover it</a>
