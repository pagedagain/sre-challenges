---
title: "Python Syntax Error"
description: "The deployment pipeline ran without issues, but the automation script is failing in production. The CI linter was disabled for this hotfix branch."
tags:
  - "python"
  - "syntax"
  - "debugging"
---

# Python Syntax Error

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

The deployment pipeline ran without issues, but the automation script is failing in production. The CI linter was disabled for this hotfix branch.

Fix the syntax error and verify the script runs.

### Objectives

- Python script compiles successfully
- Script processes each item (alpha, beta, gamma) and prints Done.

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-003 | **Difficulty:** Medium (200) | **Estimated Time:** 15 minutes | **Focus:** `python`, `syntax`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/syntax-error?utm_source=challenges&utm_medium=writeup&utm_campaign=syntax-error){ .md-button .md-button--primary }

</div>

---

## Production Context

Syntax errors slip through when linting is skipped on hotfix branches. Always verify locally before deploy.

---

## Hints

??? tip "Hint 1"
    Run the script manually using `python3 app.py` and read the error

??? tip "Hint 2"
    Check indentation carefully. Python is whitespace-sensitive

??? tip "Hint 3"
    Use `python3 -m py_compile app.py` to verify syntax

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    The `print` statement inside the `for` loop was not indented, causing an `IndentationError`.

    ### Diagnostic Steps
    ```bash
    python3 app.py
    python3 -m py_compile app.py
    ```

    ### Resolution
    Fix indentation in `/app/app.py`:
    
    ```python
    for item in items:
        print(f"Processing: {item}")
    ```

---

## Learning Points

- Python uses indentation for block structure
- `py_compile` is a quick syntax check without running the script
- Read the stack trace. It points to the exact line

---

## Best Practices

- Never disable linters on hotfix branches
- Run `python3 -m py_compile` in CI
- Use an IDE with Python syntax highlighting

---

## References

- Python IndentationError documentation
