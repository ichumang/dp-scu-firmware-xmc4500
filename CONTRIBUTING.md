# Contributing

This is a **proprietary project** developed under employer NDA.

## Access

Contributions require:

1. A signed Non-Disclosure Agreement (NDA) with the project owner
2. Written approval from the project maintainer
3. Familiarity with IEC 61508 (Functional Safety) and XMC4500 platform

## Code Standards

All firmware changes must comply with:

- C99, no dynamic allocation (`malloc`/`free` prohibited)
- No recursion (stack depth must be statically analyzable)
- MISRA C:2012 — Required and Advisory rules
- `-Wall -Wextra -Werror -Wpedantic -Wmissing-declarations`
- All safety-relevant functions must have documented pre/post conditions
- Program flow monitoring macros (`FLOW_ENTER` / `FLOW_EXIT`) on every safety function

## Review Requirements

- Full unit test suite must pass (191 tests)
- Branch coverage ≥ 95% on safety-critical modules
- Static analysis clean (no new warnings)
- Review by a certified functional safety engineer (CFSPE or equivalent)
- Timing analysis: no cycle overrun (Durchlaufzeit ≤ 14 ms)

## Contact

**Umang Panchal** — [GitHub](https://github.com/ichumang) · [github.com/ichumang](https://github.com/ichumang)
