---
title: "Could Not Find a Version That Satisfies"
description: "A security bump went out this morning and the invoices API no longer builds. pip cannot find a set of versions that satisfies the two direct dependencies. This host has no network. SSH still works."
tags:
  - "python"
  - "pip"
  - "build"
---

# Could Not Find a Version That Satisfies

> Practice debugging this real-world production issue. This challenge is based on authentic SRE incident response patterns.

---

## Scenario

A security bump went out this morning and the invoices API no longer builds. pip cannot find a set of versions that satisfies the two direct dependencies. This host has no network. SSH still works.

Get both libraries installed from the wheels already on the box, then run the invoices app so it writes a complete run log. Retry `python3 /app/app.py` in the shell as often as you need before Check.

### Objectives

- `python3 /app/app.py` exits 0
- Both direct dependencies remain declared and import

---

## Interactive Sandbox

<div class="challenge-cta" markdown>

Try this in a live terminal before you open the solution.

**Catalogue ID:** PA-015 | **Difficulty:** Hard (400) | **Estimated Time:** 25 minutes | **Focus:** `python`, `pip`

[Launch challenge on Paged Again](https://pagedagain.com/incidents/dependency-conflict?utm_source=challenges&utm_medium=writeup&utm_campaign=dependency-conflict){ .md-button .md-button--primary }

</div>

---

## Production Context

A CVE bump on a shared library often lands as a new pin next to an old framework pin. The resolver error is the clue. Read each package's METADATA, then pick versions whose ranges overlap. Uninstalling one of the directs makes the error go away and drops a feature.

---

## Hints

??? tip "Hint 1"
    Install from the app directory and read the resolver error

??? tip "Hint 2"
    Inspect each package's metadata for its requirement range. See which wheels are already on the box

??? tip "Hint 3"
    The two pins demand click ranges that do not intersect. Raise the web framework pin so it accepts click 8, then install offline from the local wheel set

---

## Solution

??? success "View Root Cause and Resolution"

    ### Root Cause
    `requirements.txt` pins `flask==1.1.4` and `click==8.1.7`. Flask 1.1.4 requires `click>=5.1,<8`. Click 8.1.7 was the security bump. Those ranges do not intersect, so pip cannot resolve. Wheels for Flask 2.3.3 (which accepts click 8) are already under `/opt/packages`.

    ### Diagnostic Steps
    ```bash
    cat /app/requirements.txt
    pip3 install --no-index --find-links=/opt/packages -r /app/requirements.txt
    ls /opt/packages
    python3 - << 'PY'
    import zipfile, pathlib
    for w in pathlib.Path("/opt/packages").glob("*.whl"):
        with zipfile.ZipFile(w) as z:
            names = [n for n in z.namelist() if n.endswith("METADATA")]
            if not names:
                continue
            text = z.read(names[0]).decode()
            reqs = [ln for ln in text.splitlines() if ln.startswith("Requires-Dist:")]
            if reqs:
                print(w.name)
                print("\n".join(reqs[:8]))
                print()
    PY
    ```

    ### Resolution
    Raise the Flask pin so it accepts click 8, then install from the local wheels.
    
    ```bash
    cat > /app/requirements.txt << 'EOF'
    flask==2.3.3
    click==8.1.7
    EOF
    pip3 install --no-index --find-links=/opt/packages -r /app/requirements.txt
    ```

---

## Learning Points

- Two direct pins can be incompatible through a shared library even when each pin looks fine alone
- Wheel METADATA lists the real requirement ranges
- Dropping one requirement is not a resolution

---

## Best Practices

- Bump a framework and its dependencies in the same change
- Keep an offline wheelhouse that includes the working set, not only the broken pins
- Run `pip check` in CI after every pin change

---

## References

- https://pip.pypa.io/en/stable/topics/dependency-resolution/

---

<a class="star-cta" href="https://github.com/pagedagain/sre-challenges">Found this useful? <span class="star-cta-link">⭐ Star the repo</span> to help others discover it</a>
