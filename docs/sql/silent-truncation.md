---
title: "Invalid Credentials After Migration"
description: "A schema change went out last Friday with zero errors in the migration log. Logins started failing for accounts that were fine on Thursday. The error is \"invalid credentials\", not \"user not found\". Th"
tags:
  - "sql"
  - "migration"
  - "varchar"
---

# Invalid Credentials After Migration

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

A schema change went out last Friday with zero errors in the migration log. Logins started failing for accounts that were fine on Thursday. The error is "invalid credentials", not "user not found". The engineer who ran it is on holiday.

Connect with `psql -U postgres -d auth`. The failing lookups are `/opt/reports/login_check.sql`. Fix the data so those exact-match logins work again. Re-run the lookups in `psql` as often as you need before Check.

### Objectives

- The accounts in `/opt/reports/login_check.sql` exact-match again
- The pre-migration snapshot table is still there

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-017 | **Difficulty:** Medium (200) | **Estimated Time:** 20 minutes | **Focus:** `sql`, `migration`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/silent-truncation?utm_source=challenges&utm_medium=writeup&utm_campaign=silent-truncation){ .md-button .md-button--primary }

</div>

---

## Production Context

`USING` on `ALTER TYPE` is how teams suppress "value too long" errors during a "harmless" column shrink. The command returns success, the app keeps sending the original string, and the failure looks like a bad password.

---

## Hints

??? tip "Hint 1"
    Run the login lookups, then select those same ids and compare the stored email

??? tip "Hint 2"
    Check the email column type and the longest stored value. Look for a table that still has the pre-change rows

??? tip "Hint 3"
    The migration shortened email to VARCHAR(50) with USING left. Widen the column and restore from users_backup

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    Friday's migration ran `ALTER TABLE users ALTER COLUMN email TYPE VARCHAR(50) USING left(email, 50)`. PostgreSQL accepted it with no error and shortened every email longer than 50 characters. Login still looks up the full address, so those rows no longer match. About 750 of 5000 users are affected. A pre-change copy remains in `users_backup`.

    ### Diagnostic Steps
    ```bash
    psql -U postgres -d auth
    cat /opt/reports/login_check.sql
    ```
    
    ```sql
    SELECT id, email, length(email) FROM users WHERE id IN (1234, 2345, 3456);
    SELECT MAX(length(email)) FROM users;
    \d users
    \dt
    SELECT id, email, length(email) FROM users_backup WHERE id = 1234;
    ```

    ### Resolution
    Widen the column, then copy the original emails back:
    
    ```bash
    psql -U postgres -d auth -v ON_ERROR_STOP=1 <<'EOF'
    ALTER TABLE users ALTER COLUMN email TYPE VARCHAR(255);
    UPDATE users u SET email = b.email FROM users_backup b WHERE u.id = b.id;
    EOF
    ```
    
    Widening alone leaves the truncated values. Restoring without widening fails because the long strings still do not fit. Do not drop `users_backup`.

---

## Learning Points

- `ALTER COLUMN ... TYPE varchar(n) USING left(...)` truncates quietly
- Login `WHERE email = $1` needs the stored value to equal the full address
- A backup table is the recovery path; shrinking the lookup string hides the data loss

---

## Best Practices

- Shrink a VARCHAR only after `SELECT MAX(length(col))` and a check for rows that would not fit
- Prefer an error on overflow over a `USING` clause that clips production data
- Keep a pre-migration snapshot until the app has been verified against the new type

---

## References

- https://www.postgresql.org/docs/14/sql-altertable.html
- https://www.postgresql.org/docs/14/datatype-character.html

---

<a class="star-cta" href="https://github.com/pagedagain/sre-challenges">Found this useful? <span class="star-cta-link">⭐ Star the repo</span> to help others discover it</a>
