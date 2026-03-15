# Timing Analysis

## Cycle Budget

The DP-SCU operates on a strict **16 ms non-preemptive cycle**. The cycle is
divided into fixed time slots. All measurements are taken on XMC4500 at 120 MHz
using the DWT cycle counter (`DWT->CYCCNT`).

---

## Slot Breakdown

```
  ┌─────────┬──────────────┬─────────┬──────────┬──────────────────┐
  │   RX    │   PROCESS    │   TX    │   DIAG   │      IDLE        │
  │ 0-2 ms  │   2-6 ms     │ 6-8 ms  │ 8-9 ms   │    9-16 ms       │
  └─────────┴──────────────┴─────────┴──────────┴──────────────────┘
  ▲                                                                 ▲
  │ SysTick fires                                      next SysTick │
```

### Slot Details

#### RX Slot (0 – 2 ms)

| Operation | Typical | Worst-Case |
|-----------|---------|------------|
| ETH DMA buffer read | 18 µs | 45 µs |
| IP header validate | 2 µs | 3 µs |
| UDP demux | 1 µs | 2 µs |
| PROFIsafe CRC1 verify | 7 µs | 8 µs |
| PROFIsafe CRC2 verify | 3 µs | 4 µs |
| Sequence number check | 1 µs | 2 µs |
| Toggle bit check | < 1 µs | 1 µs |
| Watchdog timestamp | < 1 µs | 1 µs |
| **Slot total** | **~33 µs** | **~66 µs** |

The RX slot has significant headroom. The 2 ms budget accounts for worst-case
Ethernet frame arrival time and potential re-receive after CRC error at the
MAC layer (the MAC drops bad frames, but we budget for the latency).

#### Process Slot (2 – 6 ms)

| Operation | Typical | Worst-Case |
|-----------|---------|------------|
| Safety data unpack | 4 µs | 6 µs |
| Application logic | 850 µs | 1.2 ms |
| Output computation | 120 µs | 180 µs |
| DPRAM write (own data) | 8 µs | 12 µs |
| DPRAM read (partner data) | 8 µs | 12 µs |
| Cross-comparison | 5 µs | 8 µs |
| Sequence hash verify | 3 µs | 5 µs |
| Output commit / passivate | 2 µs | 15 µs |
| **Slot total** | **~1.0 ms** | **~1.44 ms** |

The 4 ms process budget accommodates future application logic growth.
Current utilization is ~36% of slot budget.

#### TX Slot (6 – 8 ms)

| Operation | Typical | Worst-Case |
|-----------|---------|------------|
| PROFIsafe telegram build | 12 µs | 15 µs |
| CRC1 compute | 7 µs | 8 µs |
| CRC2 compute | 3 µs | 4 µs |
| UDP/IP header build | 3 µs | 5 µs |
| IP checksum compute | 2 µs | 3 µs |
| ETH DMA transmit | 15 µs | 35 µs |
| Heartbeat (every 500 ms) | 8 µs | 12 µs |
| **Slot total** | **~50 µs** | **~82 µs** |

#### Diagnostics Slot (8 – 9 ms)

| Operation | Typical | Worst-Case |
|-----------|---------|------------|
| RAM March-C (32 bytes) | 45 µs | 52 µs |
| ROM CRC-32 (4 KB) | 380 µs | 410 µs |
| CPU self-test | 85 µs | 120 µs |
| Stack canary check | 1 µs | 2 µs |
| Flow counter verify | 2 µs | 3 µs |
| Durchlaufzeit log | 5 µs | 8 µs |
| **Slot total** | **~518 µs** | **~595 µs** |

---

## Aggregate Timing

| Metric | Typical | Worst-Case |
|--------|---------|------------|
| Total execution per cycle | 1.60 ms | 2.18 ms |
| Cycle budget | 16.0 ms | 16.0 ms |
| Margin | 14.4 ms | 13.82 ms |
| CPU load | 10.0% | 13.6% |

> **Note:** The 8.7 ms worst-case figure in the README accounts for
> pathological Ethernet retransmissions and DMA stalls observed during
> 72-hour stress testing, not typical operation.

---

## Jitter Analysis

Measured over 100,000 consecutive cycles on hardware:

| Metric | Value |
|--------|-------|
| Mean cycle period | 16.0000 ms |
| Standard deviation | 3.2 µs |
| Min observed | 15.9988 ms |
| Max observed | 16.0012 ms |
| Peak-to-peak jitter | ±12 µs |

Jitter source: SysTick has inherent ±1 tick uncertainty (8.33 ns at 120 MHz),
but the dominant contributor is ETH DMA completing an in-progress transfer at
the moment SysTick fires. The DMA finishes its current burst (up to 4 beats
× 32 bits = 16 bytes) before the bus is available to the CPU.

---

## Durchlaufzeit Monitoring

Every cycle, the firmware records execution time using DWT:

```c
uint32_t t_start = DWT->CYCCNT;
/* ... entire cycle ... */
uint32_t t_end   = DWT->CYCCNT;
uint32_t dt_us   = (t_end - t_start) / (SystemCoreClock / 1000000U);
```

Thresholds:

| Condition | Action |
|-----------|--------|
| `dt_us` < 14000 (14 ms) | Normal operation |
| 14000 ≤ `dt_us` < 16000 | `DIAG_TIMING_WARNING` logged, counter incremented |
| `dt_us` ≥ 16000 (16 ms) | Immediate transition to `SCU_STATE_SAFE` |

Over the 72-hour soak test (16.2 million cycles), zero `DIAG_TIMING_WARNING`
events were recorded. The highest measured Durchlaufzeit was 8.7 ms.

---

## Clock Configuration

| Clock | Frequency | Source | Notes |
|-------|-----------|--------|-------|
| fSYS (CPU) | 120 MHz | PLL (12 MHz XTAL × 10) | Max rated speed |
| fPB (peripheral bus) | 120 MHz | = fSYS | No prescaler |
| SysTick | 120 MHz | = fCPU | 16 ms = 1,920,000 ticks |
| ETH MAC | 50 MHz | RMII REF_CLK from PHY | KSZ8031 provides |
| DWT CYCCNT | 120 MHz | = fCPU | Free-running, wraps at ~35.8 s |
