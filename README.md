# DP-SCU Safety Controller Firmware — XMC4500 ARM Cortex-M4

![SIL 3](https://img.shields.io/badge/SIL-3-red)
![IEC 61508](https://img.shields.io/badge/IEC-61508-blue)
![ARM Cortex-M4](https://img.shields.io/badge/ARM-Cortex--M4-orange)
![16ms Cycle](https://img.shields.io/badge/Cycle-16ms-brightgreen)
![XMC4500](https://img.shields.io/badge/MCU-XMC4500-informational)
![PROFIsafe](https://img.shields.io/badge/Protocol-PROFIsafe-purple)

Production firmware for the **Distributed Safety Control Unit (DP-SCU)** —
a dual-channel SIL 3 safety processor bridging PROFINET and PROFIsafe/PROFIBUS
field networks. Bare-metal C99 on Infineon XMC4500, deterministic 16 ms real-time
cycle, zero dynamic allocation.

> **Note:** Source implementation (`src/*.c`) and test suite (`tests/`) are
> proprietary. Public headers and documentation are provided for portfolio
> reference. See [NOTICE.md](NOTICE.md) for details.

---

## System Architecture

```
  ┌─────────────────────┐         Ethernet (100 Mbit)         ┌──────────────────────────────────┐
  │                     │◄───────────────────────────────────► │          DP-SCU Module           │
  │   DP-MainUnit       │   UDP/IP  (black-channel PROFIsafe) │                                  │
  │   (PROFINET IO      │                                     │  ┌────────────┐ ┌────────────┐   │
  │    Controller)       │                                     │  │ Channel A  │ │ Channel B  │   │
  │                     │                                     │  │ XMC4500    │ │ XMC4500    │   │
  │   - Cycle sync      │                                     │  │            │ │            │   │
  │   - F-Host role     │                                     │  │  Safety    │ │  Safety    │   │
  │   - Diagnostics     │                                     │  │  Core      │ │  Core      │   │
  └─────────────────────┘                                     │  │            │ │            │   │
                                                              │  └─────┬──────┘ └──────┬─────┘   │
                                                              │        │   DPRAM        │        │
                                                              │        │  (cross-       │        │
                                                              │        │  comparison)   │        │
                                                              │        └───────┬────────┘        │
                                                              │                │                 │
                                                              └────────────────┼─────────────────┘
                                                                               │
                                                              PROFIBUS DP / PROFIsafe
                                                                               │
                                                     ┌─────────────────────────┼─────────────────────────┐
                                                     │                         │                         │
                                              ┌──────┴──────┐          ┌──────┴──────┐          ┌──────┴──────┐
                                              │  F-Device   │          │  F-Device   │          │  F-Device   │
                                              │  Safety I/O │          │  VFD Drive  │          │  Safety I/O │
                                              │  (16 DI/8DO)│          │  (STO/SS1)  │          │  (Encoder)  │
                                              └─────────────┘          └─────────────┘          └─────────────┘
```

The DP-SCU acts as a transparent safety bridge: it receives PROFIsafe telegrams
from the DP-MainUnit over Ethernet (black-channel UDP), performs safety
validation (CRC, sequence number, watchdog), and forwards safe process data
to field devices via PROFIBUS/PROFIsafe. Both Channel A and Channel B execute
the same safety logic independently and cross-compare results through shared
DPRAM before any output is committed.

---

## Firmware Layers

| Layer | Directory | Responsibility |
|-------|-----------|----------------|
| **HAL** | `src/hal/` | XMC4500 BSP, GPIO, SysTick (16 ms tick), DWT cycle counter, UART debug, hardware watchdog (WDT) |
| **Ethernet** | `src/eth/` | Raw MAC driver (ETH0, KSZ8031RNL PHY), MDIO read/write, DMA ring descriptors, minimal UDP/IP stack (no lwIP dependency) |
| **Communication** | `src/comm/` | PROFIsafe telegram encode/decode, heartbeat generation, diagnostic frame assembly, telegram buffer management |
| **Safety Core** | `src/safety/` | Dual-channel cross-monitor, safe state management, passivation logic, RAM/ROM/CPU self-tests, program flow monitoring |
| **Application** | `src/app/` | Main state machine (INIT → CONFIGURE → RUN → SAFE_STATE → ERROR), 16 ms cycle orchestration, diagnostic counters |
| **Utilities** | `src/util/` | CRC-16/CRC-32, safe string helpers, static assertion macros, endian conversion |

### Layer Dependencies

```
  Application
      │
      ├──► Safety Core ──► HAL
      │        │
      │        └──► Utilities
      │
      ├──► Communication ──► Utilities
      │
      └──► Ethernet ──► HAL
```

No upward or circular dependencies. Each layer only calls the layer directly
below it. Enforced by `-Wmissing-declarations` and include guard discipline.

---

## Real-Time Architecture

The system runs a strict **16 ms non-preemptive cycle** driven by SysTick.
All processing fits within the cycle budget with measured worst-case margin.

```
  0 ms          2 ms          6 ms          8 ms     9 ms         16 ms
  │──── RX ─────│──── PROCESS ─│──── TX ─────│─ DIAG ─│─── IDLE ───│
  │              │              │              │        │            │
  │ ETH DMA     │ PROFIsafe    │ Build TX     │ RAM    │ WDT kick   │
  │ receive     │ decode       │ telegram     │ test   │ Background │
  │ UDP parse   │ Cross-check  │ ETH DMA      │ ROM    │ logging    │
  │ Telegram    │ State logic  │ transmit     │ test   │            │
  │ validate    │ Output calc  │ Heartbeat    │ DWT    │            │
  └──────────────┴──────────────┴──────────────┴────────┴────────────┘
```

| Metric | Value |
|--------|-------|
| Base cycle | 16 ms (SysTick) |
| RX slot | 0 – 2 ms |
| Process slot | 2 – 6 ms |
| TX slot | 6 – 8 ms |
| Diagnostics slot | 8 – 9 ms |
| Idle / background | 9 – 16 ms |
| Worst-case execution (measured) | 8.7 ms |
| CPU load (worst case) | 54.4% |
| Jitter (measured, 100k cycles) | ±12 µs |
| Timing source | DWT CYCCNT @ 120 MHz |

### Durchlaufzeit Monitoring

Every cycle, the DWT cycle counter measures actual execution time.  If any
cycle exceeds 14 ms (87.5% of budget), a `DIAG_TIMING_WARNING` is logged.
If a cycle exceeds 16 ms, the system transitions to `SAFE_STATE` immediately.

---

## Safety Features

### Dual-Channel Cross-Monitoring

Both Channel A and Channel B execute identical safety logic on separate
XMC4500 processors. After the Process slot, each channel writes its computed
outputs and a sequence hash to shared DPRAM. Both channels then compare:

1. Output data — byte-for-byte comparison
2. Sequence counter hash — verifies identical program flow
3. Alive counter — confirms the partner channel is responsive

Mismatch → immediate passivation, both channels drive outputs to safe state.

### Memory Integrity

| Test | Algorithm | Coverage | Per-Cycle Slice |
|------|-----------|----------|-----------------|
| RAM test | March-C (destructive, with backup/restore) | Full SRAM 160 KB | 32 bytes / cycle → full pass every ~82 s |
| ROM test | CRC-32 over flash | Full 1 MB flash | 4 KB / cycle → full pass every ~4 s |

### CPU Self-Test

- ALU: predetermined pattern computation, compare against known-good
- Register file: walking-1 pattern across R0–R12, SP, LR
- Interrupt controller: NVIC priority inversion test
- Stack: canary word `0xDEADBEEF` at stack bottom, checked every cycle

### Program Flow Monitoring

Each safety-relevant function increments a sequence counter on entry and exit.
At the end of the Process slot, the accumulated counter is compared against
the expected value. Deviation indicates corrupted or skipped execution →
passivation.

### Watchdog

| Type | Timeout | Purpose |
|------|---------|---------|
| Hardware WDT | 50 ms | System-level recovery (3× cycle budget) |
| Software watchdog | 18 ms | Cycle overrun detection (16 ms + 2 ms margin) |
| PROFIsafe watchdog | 100 ms | Communication timeout (F-Host side) |

---

## Module Structure

```
src/
├── hal/
│   ├── xmc4500_bsp.h          — Board support package interface
│   ├── xmc4500_bsp.c          — Clock, GPIO, pin mux, SysTick, DWT
│   ├── gpio.h / gpio.c        — Safe GPIO with readback verification
│   ├── uart.h / uart.c        — UART0 debug output (115200, 8N1)
│   └── wdt.h / wdt.c          — Hardware watchdog driver
├── eth/
│   ├── eth_mac.h / eth_mac.c  — ETH0 MAC init, DMA ring, TX/RX
│   ├── phy_ksz8031.h / .c     — KSZ8031RNL PHY driver via MDIO
│   ├── udp_ip.h / udp_ip.c    — Minimal UDP/IP (ARP, ICMP echo)
│   └── eth_buffers.h          — Static TX/RX descriptor pools
├── comm/
│   ├── profisafe_codec.h / .c — PROFIsafe telegram encode/decode
│   ├── heartbeat.h / .c       — Cyclic alive telegram (500 ms)
│   └── diag_frame.h / .c      — Diagnostic frame assembly
├── safety/
│   ├── cross_monitor.h / .c   — DPRAM cross-comparison logic
│   ├── safe_state.h / .c      — Passivation, safe output driver
│   ├── ram_test.h / .c        — March-C RAM test (sliced)
│   ├── rom_test.h / .c        — CRC-32 flash integrity (sliced)
│   ├── cpu_test.h / .c        — ALU, register, NVIC self-test
│   └── flow_monitor.h / .c    — Sequence counter program flow check
├── app/
│   ├── scu_state_machine.h    — State machine interface + states enum
│   ├── scu_state_machine.c    — Main FSM implementation
│   ├── cycle_manager.h / .c   — 16 ms cycle orchestration
│   └── main.c                 — Entry point, init sequence
└── util/
    ├── crc.h / crc.c          — CRC-16 (PROFIsafe), CRC-32 (ROM test)
    ├── safe_string.h / .c     — Bounded memcpy, memcmp, memset
    └── assert.h               — Static + runtime assertion macros
```

---

## Build & Debug

| Tool | Version | Purpose |
|------|---------|---------|
| DAVE IDE | 4.5.0 | Project management, XMCLib integration |
| GCC ARM | 10.3-2021.10 | Cross-compiler (`arm-none-eabi-gcc`) |
| J-Link | V7.60+ | JTAG debugging and flash programming |
| SEGGER RTT | 7.60+ | Real-time printf over debug probe |
| Wireshark | 4.0+ | Ethernet capture validation |
| Python 3 | 3.10+ | Test scripts, Wireshark dissector |

### Compiler Flags

```
-mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16
-O2 -Wall -Wextra -Werror -Wpedantic -Wmissing-declarations
-ffunction-sections -fdata-sections -fno-common
-DXMC4500_F100x1024 -DSIL3_BUILD
```

### Flash & Debug

```bash
# Flash via J-Link Commander
JLinkExe -device XMC4500-1024 -if SWD -speed 4000 -autoconnect 1 \
    -CommandFile flash_commands.jlink

# RTT console (live debug output)
JLinkRTTClient
```

---

## Test Results

| Category | Tests | Pass | Coverage | Notes |
|----------|-------|------|----------|-------|
| Unit — Safety Core | 87 | 87 | 96.4% | Cross-monitor, RAM/ROM test, flow check |
| Unit — Communication | 42 | 42 | 94.1% | PROFIsafe codec, heartbeat, diag frames |
| Unit — Ethernet | 23 | 23 | 91.8% | MAC driver, UDP/IP, ARP |
| Integration — Dual-Channel | 18 | 18 | — | DPRAM cross-comparison on real hardware |
| Integration — PROFIsafe E2E | 12 | 12 | — | Full telegram round-trip with DP-MainUnit |
| Wireshark Validation | 8 | 8 | — | Packet structure, CRC, timing |
| Stress — 72h Soak | 1 | 1 | — | 16.2M cycles, 0 missed deadlines |

Total: **191 tests**, all passing. Branch coverage across safety-critical
modules: **96.4%** (target: ≥ 95% per project quality plan).

---

## Hardware

### XMC4500 Relax Lite Kit V1

| Parameter | Value |
|-----------|-------|
| MCU | XMC4500-F100x1024 |
| Core | ARM Cortex-M4F @ 120 MHz |
| Flash | 1024 KB |
| SRAM | 160 KB |
| Ethernet | ETH0 with RMII |
| PHY | KSZ8031RNL (on-board) |
| Debug | J-Link OB (on-board) |

### RMII Pin Mapping

| Signal | XMC4500 Pin | Port | Function |
|--------|-------------|------|----------|
| ETH_RMII_TXD0 | P2.8 | P2 | ALT1 |
| ETH_RMII_TXD1 | P2.9 | P2 | ALT1 |
| ETH_RMII_TX_EN | P2.5 | P2 | ALT1 |
| ETH_RMII_RXD0 | P2.2 | P2 | Input |
| ETH_RMII_RXD1 | P2.3 | P2 | Input |
| ETH_RMII_CRS_DV | P15.9 | P15 | Input |
| ETH_RMII_REF_CLK | P15.8 | P15 | Input |
| ETH_MDIO | P2.0 | P2 | ALT1 |
| ETH_MDC | P2.7 | P2 | ALT1 |
| PHY_RESET | P2.6 | P2 | GPIO Output |

### UART Debug

| Signal | Pin | Config |
|--------|-----|--------|
| UART_TX | P1.5 | USIC0_CH0, 115200 8N1 |
| UART_RX | P1.4 | USIC0_CH0 |

---

## References

- IEC 61508:2010 — Functional Safety of E/E/PE Systems
- IEC 61784-3-3 — PROFIsafe
- IEC 61158 — PROFINET IO / PROFIBUS DP
- Infineon XMC4500 Reference Manual, V1.6
- KSZ8031RNL Datasheet, Rev. 2.1
- DP-SCU Hardware Specification, Rev. 3.0

---

## Author

**Umang Panchal** — [GitHub](https://github.com/ichumang)

- GitHub: [github.com/ichumang](https://github.com/ichumang)

---

*This repository demonstrates the design and architecture of a production
safety controller. Implementation source code is proprietary — see
[NOTICE.md](NOTICE.md).*
