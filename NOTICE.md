# Notice

## Proprietary Software

This repository contains the **DP-SCU Safety Controller Firmware** developed
by Umang Panchal as a personal project.

The implementation source code (`src/**/*.c`) and test suite (`tests/`) are
**proprietary** and protected under NDA. They are not included in this public
repository.

## What Is Public

| Path | Contents |
|------|----------|
| `README.md` | Project overview, architecture, real-time analysis, hardware setup |
| `docs/` | System architecture, timing analysis, safety concept, hardware setup, API reference |
| `src/app/scu_state_machine.h` | State machine public API — design showcase |
| `src/hal/xmc4500_bsp.h` | Board support package API — hardware knowledge showcase |
| `CONTRIBUTING.md` | Contribution policy and coding standards |
| `LICENSE` | Proprietary license terms |

The public materials demonstrate the design, architecture, safety methodology,
and hardware expertise of the project without exposing the safety-critical
implementation.

## Why

The DP-SCU firmware is SIL 3 certified per IEC 61508 and developed under NDA
with the system integrator. Publishing the full source would violate contractual
obligations and potentially compromise safety certification integrity.

## Contact

For code review access, collaboration, or licensing inquiries:

**Umang Panchal**
- GitHub: [github.com/ichumang](https://github.com/ichumang)
