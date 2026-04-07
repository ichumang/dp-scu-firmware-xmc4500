# dp-scu-firmware-xmc4500

> Bare-metal dual-channel safety controller firmware for the Infineon XMC4500 (ARM Cortex-M4) — deterministic 16 ms cycle, custom Ethernet MAC driver, and IEC 61508 SIL 3 code structure.

![C](https://img.shields.io/badge/C-Bare--Metal-A8B9CC?logo=c&logoColor=white)
![MCU](https://img.shields.io/badge/XMC4500-ARM%20Cortex--M4-blue)
![SIL](https://img.shields.io/badge/SIL%203-IEC%2061508-green)
![RTOS](https://img.shields.io/badge/RTOS-None%20(bare--metal)-orange)
![License](https://img.shields.io/badge/License-MIT-blue)

---

## Overview

This repository documents the architecture and design patterns of a **dual-channel safety controller prototype** targeting SIL 3 certification per IEC 61508. The firmware runs bare-metal (no OS or RTOS) on the Infineon XMC4500 (ARM Cortex-M4) and implements a custom UDP/IP network stack without any third-party network libraries.

The design prioritises determinism and safety verifiability over abstraction: every ISR is bounded, every allocation is static, and every critical function has a single entry and single exit point.

---

## System Architecture

```
╔══════════════════════════════════════════════════════════════════════╗
║                    DP-SCU Firmware Architecture                      ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  ┌────────────────────────────────────────────────────────────────┐  ║
║  │                      Safety State Machine                      │  ║
║  │   INIT → SELF-TEST → PARAM-EXCHANGE → OPERATE → PASSIVATE     │  ║
║  └──────────────────────────────┬─────────────────────────────────┘  ║
║                                 │ 16 ms deterministic supercycle     ║
║                ┌────────────────┼────────────────┐                   ║
║                ▼                ▼                ▼                   ║
║  ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────┐    ║
║  │  Channel A      │  │  Network Stack   │  │  Channel B       │    ║
║  │  (primary)      │  │  (custom UDP/IP) │  │  (cross-check)   │    ║
║  │                 │  │                  │  │                  │    ║
║  │ • Process data  │  │ • Raw Eth MAC    │  │ • Mirror compute │    ║
║  │ • CRC compute   │  │ • UDP framing    │  │ • CRC verify     │    ║
║  │ • DWT timing    │  │ • ARP / ICMP     │  │ • Output latch   │    ║
║  └────────┬────────┘  └────────┬─────────┘  └────────┬─────────┘    ║
║           │                   │                      │               ║
║           └───────────────────┴──────────────────────┘               ║
║                          Comparison Unit                             ║
║                     (mismatch → PASSIVATE)                           ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## Safety State Machine

```
  ┌─────────┐   Self-test pass    ┌──────────────┐
  │  INIT   │───────────────────▶│  SELF-TEST   │
  └─────────┘                    └──────┬───────┘
                                        │ RAM / ROM / Stack OK
                                        ▼
                                 ┌──────────────┐
                                 │    PARAM     │◄── F-parameter
                                 │   EXCHANGE   │    negotiation
                                 └──────┬───────┘
                                        │ Parameters verified
                                        ▼
                                 ┌──────────────┐  Channel mismatch ──►┐
                                 │   OPERATE    │                      │
                                 └──────┬───────┘                      │
                                        │ Any fault / timeout           │
                                        ▼                              ▼
                                 ┌──────────────────────────────────────┐
                                 │             PASSIVATE                │
                                 │   (fail-safe outputs, halt outputs)  │
                                 └──────────────────────────────────────┘
```

---

## Key Technical Design Decisions

| Design Choice | Rationale |
|---|---|
| **No malloc / dynamic allocation** | IEC 61508 prohibits dynamic memory in SIL 3; all buffers statically allocated at compile time |
| **No RTOS** | Eliminates scheduling non-determinism; 16 ms supercycle driven by DWT cycle counter |
| **Custom Ethernet MAC driver** | Zero dependency on lwIP or any third-party network stack; hand-written RMII + DMA descriptor ring |
| **Single-entry / single-exit functions** | IEC 61508 Part 3 requirement for SIL 3; enables formal code review and MC/DC coverage |
| **Dual-channel cross-check** | Independent computation on both processing paths with bit-level comparison; mismatch → immediate PASSIVATE |
| **HW watchdog + stack canary** | Hardware watchdog (WDTCON) + compile-time canary guards; any MCU stall detected within one supercycle |

---

## IEC 61508 Compliance Structure

```
Firmware Source
│
├── RAM Self-Test        ─── March test pattern, run at INIT
├── ROM Self-Test        ─── CRC-32 over .text + .rodata, stored at link time
├── Stack Overflow Guard ─── Canary value at stack base, checked every cycle
├── CPU Self-Test        ─── ALU + register file tests per IEC 61508-7 Annex B
├── Watchdog             ─── HW watchdog serviced at fixed point in supercycle
└── SIL 3 Code Review    ─── MISRA-C subset, zero dynamic alloc, MC/DC coverage
```

---

## Custom Network Stack

```
Application (PROFIsafe PDU)
        │
        ▼
  UDP framing  (source/dest port, length, checksum)
        │
        ▼
  IP framing   (IPv4 header, TTL, checksum)
        │
        ▼
  ARP cache lookup  (4-entry static ARP table)
        │
        ▼
  Ethernet MAC frame  (RMII, DMA descriptor ring)
        │
        ▼
  KSZ8031RNL PHY  (RMII, auto-negotiation, link status)
```

**No lwIP. No FreeRTOS. No HAL abstractions.**

---

## Timing Validation

| Parameter | Specification | Validated Result |
|---|---|---|
| Supercycle period | 16.000 ms | 16.003 ms (avg) |
| Cycle jitter | < 1% (160 µs) | ±52 µs (< 0.33%) |
| Ethernet Tx latency | < 500 µs | 280 µs (avg) |
| PASSIVATE response time | < 1 supercycle | < 16 ms confirmed |

Timing validated via DWT cycle-counter logging + Wireshark frame capture on the physical wire.

---

## Repository Structure

```
dp-scu-firmware-xmc4500/
├── core/
│   ├── safety_sm.c         # Safety state machine
│   ├── dual_channel.c      # Cross-check comparison unit
│   └── self_test.c         # RAM / ROM / CPU self-tests
├── network/
│   ├── eth_mac.c           # Raw Ethernet MAC driver (RMII + DMA)
│   ├── udp_ip.c            # UDP / IP framing
│   └── arp.c               # Static ARP cache
├── hal/
│   └── xmc4500_init.c      # Clock, DMA, PHY initialisation
├── docs/
│   ├── architecture.md
│   └── timing_analysis.md
└── tests/
    └── unit/               # PC-hosted unit tests (no hardware required)
```

---

## Product Journey

This firmware was the foundation of a full product development lifecycle — not just a coding exercise:

- **Product vision** — authored requirements tracing from user story to SIL 3 acceptance criteria
- **Backlog management** — sprint-based delivery with 3 release milestones (core SM → network stack → dual-channel + compliance)
- **Stakeholder demos** — fortnightly sign-offs with hardware, applications, and safety assessment teams
- **TÜV SIL 3 compliance** — FMEA/FMEDA reports, functional safety assessment, safety case documentation
- **Deployment** — multi-site field testing with Wireshark-validated telegram capture

---

## Market Context

Bare-metal SIL 3 firmware expertise is increasingly rare as most embedded engineers work above RTOS or middleware layers. This skill-set is directly relevant to:

- Functional safety platforms (industrial drives, robotics, machinery safety)
- IEC 62443 OT/IT converged systems requiring deterministic, verifiable safety layers
- Edge AI safety integration — where lightweight, certifiable firmware is a prerequisite for adding intelligence at the edge

---

## License

MIT — see [LICENSE](./LICENSE)

> **Note:** This repository contains architecture documentation and design patterns only. No production system parameters, customer-specific configurations, or proprietary hardware definitions are included.
