# System Architecture

## Overview

The DP-SCU is a dual-channel safety bridge between a PROFINET IO controller
(DP-MainUnit) and PROFIBUS DP / PROFIsafe field devices. Both channels (A + B)
run identical firmware on separate XMC4500 processors and cross-compare all
safety-relevant outputs before committing them to the field bus.

---

## Physical Topology

```
                 ┌──────────────────────────────────────────────────┐
  PROFINET IO    │                    DP-SCU PCB                    │
  (Ethernet)     │                                                  │
  ───────────────┤  ┌──────────────┐    DPRAM     ┌──────────────┐ │
       100 Mbit  │  │  Channel A   │◄────────────►│  Channel B   │ │
       UDP/IP    │  │  XMC4500     │  (16 KB,     │  XMC4500     │ │
                 │  │              │   dual-port)  │              │ │
                 │  │ ETH0 ◄──────┤              ┌┤──────► ETH0  │ │
                 │  │              │              ││              │ │
                 │  └──────┬───────┘              │└──────┬───────┘ │
                 │         │                      │       │        │
                 │         │  ┌───────────────┐   │       │        │
                 │         └──┤ PROFIBUS ASIC ├───┘       │        │
                 │            │ (VPC3+C)      │           │        │
                 │            └───────┬───────┘           │        │
                 └────────────────────┼───────────────────┘        │
                                      │                            │
                               PROFIBUS DP / PROFIsafe
                                      │
                 ┌────────────────────┼────────────────────┐
                 │                    │                    │
          ┌──────┴──────┐     ┌──────┴──────┐     ┌──────┴──────┐
          │  F-Device 1 │     │  F-Device 2 │     │  F-Device 3 │
          │  Safety DI  │     │  VFD STO    │     │  Safety DO  │
          └─────────────┘     └─────────────┘     └─────────────┘
```

## Channel Architecture (per XMC4500)

Each channel runs the same firmware image. The only difference is a hardware
channel ID pin (P0.0): low = Channel A, high = Channel B. Channel A is the
primary Ethernet responder; Channel B mirrors all processing and validates
through DPRAM.

```
  ┌────────────────────────────────────────────────────────────┐
  │                     XMC4500 (per channel)                  │
  │                                                            │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
  │  │   HAL    │  │ Ethernet │  │  Safety   │  │   App    │  │
  │  │          │  │          │  │  Core     │  │          │  │
  │  │ SysTick  │  │ MAC+DMA  │  │ Cross-   │  │ State    │  │
  │  │ GPIO     │  │ PHY MDIO │  │ monitor  │  │ Machine  │  │
  │  │ UART     │  │ UDP/IP   │  │ RAM test │  │ Cycle    │  │
  │  │ WDT      │  │ ARP      │  │ ROM test │  │ Manager  │  │
  │  │ DWT      │  │          │  │ CPU test │  │          │  │
  │  └────┬─────┘  └────┬─────┘  │ Flow mon │  └────┬─────┘  │
  │       │              │        └────┬─────┘       │        │
  │       │              │             │              │        │
  │       └──────────────┴─────────────┴──────────────┘        │
  │                         │                                  │
  │                    ┌────┴────┐                              │
  │                    │  Comm   │                              │
  │                    │ PROFIs. │                              │
  │                    │ Heartbt │                              │
  │                    │ Diag    │                              │
  │                    └─────────┘                              │
  └────────────────────────────────────────────────────────────┘
```

---

## Data Flow — Receive Path

```
  ETH DMA RX IRQ
       │
       ▼
  eth_mac_receive()          ── DMA descriptor → frame buffer
       │
       ▼
  udp_ip_process()           ── validate IP checksum, demux by port
       │
       ├── port 0x9876 ──► profisafe_decode()    ── CRC1 + CRC2 + seqnr
       ├── port 0x9877 ──► heartbeat_process()   ── partner alive check
       └── port 0x9878 ──► diag_frame_process()  ── remote diagnostics
       │
       ▼
  cross_monitor_submit()     ── write decoded data + hash to DPRAM
       │
       ▼
  cross_monitor_compare()    ── read partner's DPRAM, byte-compare
       │
       ├── MATCH ──► safe_output_commit()
       └── MISMATCH ──► safe_state_passivate()
```

## Data Flow — Transmit Path

```
  cycle_manager (TX slot)
       │
       ▼
  profisafe_encode()         ── build telegram: data + status + CRC1 + CRC2 + seqnr
       │
       ▼
  udp_ip_build()             ── wrap in UDP/IP, compute checksums
       │
       ▼
  eth_mac_transmit()         ── DMA descriptor → wire
       │
       ▼
  heartbeat_tick()           ── if 500 ms elapsed, queue heartbeat frame
```

---

## Memory Map

### Flash Layout (1024 KB)

| Region | Start | Size | Contents |
|--------|-------|------|----------|
| Vector table | `0x0800_0000` | 1 KB | ISR vectors, stack pointer |
| Firmware | `0x0800_0400` | 200 KB | Application code |
| CRC-32 reference | `0x0803_2800` | 4 B | Flash integrity check value |
| Configuration | `0x080F_0000` | 64 KB | Non-volatile parameters |
| Reserved | — | — | Future use (OTA staging) |

### SRAM Layout (160 KB)

| Region | Start | Size | Contents |
|--------|-------|------|----------|
| Stack | `0x2000_0000` | 8 KB | Main stack (grows down), canary at bottom |
| BSS + Data | `0x2000_2000` | 16 KB | Globals, static buffers |
| ETH DMA | `0x2000_6000` | 8 KB | TX/RX descriptor rings + frame buffers |
| DPRAM shadow | `0x2000_8000` | 4 KB | Local copy of cross-compare data |
| Heap | — | 0 B | No dynamic allocation (SIL 3 rule) |
| March-C backup | `0x2002_7000` | 32 B | Temporary backup during RAM test |

### DPRAM (External, Dual-Port, 16 KB)

| Offset | Size | Owner | Contents |
|--------|------|-------|----------|
| `0x0000` | 256 B | Ch A → Ch B | Output data + sequence hash |
| `0x0100` | 256 B | Ch B → Ch A | Output data + sequence hash |
| `0x0200` | 4 B | Ch A | Alive counter |
| `0x0204` | 4 B | Ch B | Alive counter |
| `0x0208` | 4 B | Ch A | Cycle timestamp |
| `0x020C` | 4 B | Ch B | Cycle timestamp |
| `0x0400` | — | — | Reserved |

---

## Interrupt Priorities

| Priority | IRQ | Handler | Max Latency |
|----------|-----|---------|-------------|
| 0 (highest) | SysTick | `SysTick_Handler` — cycle trigger | — |
| 1 | ETH DMA RX | `ETH0_0_IRQHandler` — frame arrival | 5 µs |
| 2 | WDT | `SCU_0_IRQHandler` — watchdog NMI | — |
| 3 | UART TX | `USIC0_0_IRQHandler` — debug output | non-critical |

All safety processing runs in the SysTick context (priority 0). The ETH
receive interrupt only copies the frame into a ring buffer; decoding happens
in the main cycle. This keeps the safety path fully deterministic.

---

## Thread Safety

There are no threads. The firmware is a **bare-metal super-loop** with
a single interrupt (SysTick) gating the cycle start. Shared state between
the ISR and the main loop is protected by volatile access and a single
`cycle_ready` flag — no mutexes, no RTOS, no priority inversion risk.

```c
/* SysTick_Handler sets the flag */
volatile uint32_t g_cycle_ready;

/* main loop polls */
while (1) {
    if (g_cycle_ready) {
        g_cycle_ready = 0;
        cycle_manager_run();   /* entire 16 ms budget */
        wdt_kick();
    }
}
```
