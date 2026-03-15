# Safety Concept

## Applicable Standards

| Standard | Scope |
|----------|-------|
| IEC 61508:2010 | Functional safety, SIL 3 |
| IEC 62061 | Safety of machinery, SIL CL 3 |
| IEC 61784-3-3 | PROFIsafe communication profile |
| IEC 61158 | PROFINET IO / PROFIBUS DP |

---

## Safety Architecture: 1oo2 (One out of Two)

The DP-SCU implements a **1oo2 architecture** per IEC 61508-2, Table A.2.
Two independent channels (A and B) execute the same safety function on
separate processors and cross-compare results before any output is committed.

```
                    ┌───────────────┐     ┌───────────────┐
  Input ──────────► │  Channel A    │     │  Channel B    │ ◄────────── Input
  (same data)       │  (XMC4500)    │     │  (XMC4500)    │    (same data)
                    │               │     │               │
                    │  Safety Logic │     │  Safety Logic │
                    │               │     │               │
                    └───────┬───────┘     └───────┬───────┘
                            │                     │
                            │    DPRAM Compare    │
                            └──────────┬──────────┘
                                       │
                              ┌────────┴────────┐
                              │  Match?         │
                              ├─ YES ──► Output │
                              └─ NO  ──► SAFE   │
                                       └────────┘
```

### Failure Modes and Safe State

| Failure Mode | Detection | Response |
|--------------|-----------|----------|
| Channel A processor fault | CPU self-test, flow monitor | Channel B passivates alone → safe state |
| Channel B processor fault | CPU self-test, flow monitor | Channel A passivates alone → safe state |
| DPRAM corruption | Cross-compare mismatch | Both channels passivate → safe state |
| RAM bit-flip | March-C test | Affected channel passivates → partner follows |
| ROM corruption | CRC-32 mismatch | Affected channel passivates → partner follows |
| Ethernet link loss | No valid frame for 3 cycles (48 ms) | Both channels passivate → safe state |
| PROFIsafe timeout | Watchdog (100 ms, F-Host configured) | Passivation per IEC 61784-3-3 |
| PROFIsafe CRC error | CRC1/CRC2 mismatch | Telegram rejected, consecutive error counter |
| Sequence number error | Out-of-tolerance window | Passivation if 3 consecutive errors |
| Stack overflow | Canary pattern `0xDEADBEEF` corrupted | Immediate reset via WDT |
| Cycle overrun | Durchlaufzeit > 16 ms | Immediate safe state |
| Clock failure | Missing SysTick for > 50 ms | WDT reset |

### Safe State Definition

The safe state is: **all safety outputs driven to de-energized (0)**, all
PROFIsafe connections passivated, diagnostic counter incremented, and the
system remains in `SCU_STATE_SAFE` until a deliberate power cycle or remote
reset from the DP-MainUnit.

Safe state can only be exited by:
1. Power cycle (hardware reset)
2. Explicit `SCU_CMD_RESET` from DP-MainUnit (requires authenticated handshake)

---

## Diagnostic Coverage

### Hardware Diagnostics per IEC 61508-2, Table A.1

| Component | Test | DC | Period |
|-----------|------|----|--------|
| CPU / ALU | Arithmetic pattern test | 90% | Every cycle |
| Registers | Walking-1 pattern (R0–R12, SP, LR) | 90% | Every cycle |
| NVIC | Priority inversion test | 60% | Every cycle |
| RAM | March-C (full coverage in ~82 s) | 90% | 32 B / cycle |
| Flash | CRC-32 (full coverage in ~4 s) | 99% | 4 KB / cycle |
| Stack | Canary word check | 70% | Every cycle |
| Clock | SysTick period validation | 99% | Every cycle |
| Watchdog | Hardware WDT kick in idle slot | — | Every cycle |
| Ethernet PHY | Link status register poll | 80% | Every cycle |

Aggregate hardware diagnostic coverage: **> 90%** (meets SIL 3 requirement
for 1oo2 architecture per IEC 61508-2, Table 2: DC ≥ 90%).

### Software Diagnostics

| Mechanism | What It Detects | Response |
|-----------|-----------------|----------|
| Program flow monitoring | Skipped or corrupted function execution | Passivation |
| Sequence counter hash | Incorrect execution order | Passivation |
| Cross-channel comparison | Any divergence between Ch A and Ch B | Passivation |
| Durchlaufzeit monitoring | Timing anomalies, infinite loops | Safe state |
| Input plausibility | Sensor data out of physical range | Application-level fault |

---

## RAM Test: March-C

The March-C algorithm is a proven destructive memory test that detects:
- Stuck-at-0 and stuck-at-1 faults
- Transition faults
- Coupling faults (inversion, idempotent)

### Sliced Execution

Full March-C on 160 KB SRAM would take ~12 ms — too long for a single cycle.
Instead, the test is sliced:

- **32 bytes tested per cycle** (backup → test → restore)
- Address pointer advances each cycle
- Full SRAM coverage: 160 KB / 32 B = 5,000 cycles × 16 ms = **80 seconds**
- Meets IEC 61508 requirement for periodic memory test

### Algorithm (per 32-byte slice)

```
1. Backup slice to reserved area
2. ↑ (w0):   Write 0 ascending
3. ↑ (r0,w1): Read 0, write 1 ascending
4. ↑ (r1,w0): Read 1, write 0 ascending
5. ↓ (r0,w1): Read 0, write 1 descending
6. ↓ (r1,w0): Read 1, write 0 descending
7. ↓ (r0):   Read 0 descending
8. Restore slice from backup
```

If any read does not match the expected value → fault detected → passivation.

---

## ROM Test: CRC-32

Flash integrity is verified by computing CRC-32 over the firmware image and
comparing against the reference value stored at a fixed address
(`0x0803_2800`).

- **4 KB computed per cycle** (continuing from previous cycle's offset)
- Full 200 KB firmware image: 200 KB / 4 KB = 50 cycles × 16 ms ≈ **0.8 seconds**
- Reference CRC is written by the linker script at build time

---

## Program Flow Monitoring

Each safety-relevant function increments a shared sequence counter:

```c
void safety_function_a(void) {
    FLOW_ENTER(FLOW_ID_SAFETY_A);    /* counter += FLOW_ID_SAFETY_A */
    /* ... safety logic ... */
    FLOW_EXIT(FLOW_ID_SAFETY_A);     /* counter += ~FLOW_ID_SAFETY_A */
}
```

At the end of the Process slot, the accumulated counter is compared against
a compile-time-computed expected value. The complementary enter/exit pattern
ensures that:

- Skipping a function produces wrong total
- Executing a function twice produces wrong total
- Executing functions out of order produces wrong total

---

## PROFIsafe Safety Mechanisms

Per IEC 61784-3-3, the following mechanisms operate on every telegram:

| Mechanism | Purpose | Failure Mode Addressed |
|-----------|---------|------------------------|
| CRC1 (16-bit, poly 0x755B) | Safety data integrity | Bit errors, burst errors |
| CRC2 (8-bit, poly 0x1D) | Sequence integrity | Telegram replay, insertion |
| Consecutive number | Telegram ordering | Loss, repetition, delay |
| Watchdog (100 ms) | Connection monitoring | Link failure, partner crash |
| Toggle bit | Alive monitoring | Frozen partner |
| Codename (F_Par_CRC) | Connection authentication | Cross-wiring, mis-addressing |

Combined residual error probability: **< 2⁻²⁴** per IEC 61784-3-3.

---

## Failure Reaction Time

| Event | Detection Time | Reaction Time | Total |
|-------|---------------|---------------|-------|
| Cross-compare mismatch | Same cycle (< 16 ms) | Immediate passivation | < 16 ms |
| PROFIsafe CRC error | Same cycle (< 16 ms) | 3 consecutive → passivation | < 48 ms |
| PROFIsafe timeout | Watchdog (100 ms) | Immediate passivation | 100 ms |
| RAM fault | March-C detection (≤ 80 s) | Immediate passivation | ≤ 80 s |
| ROM fault | CRC-32 detection (≤ 0.8 s) | Immediate passivation | ≤ 0.8 s |
| Stack overflow | Every cycle (< 16 ms) | WDT reset | < 50 ms |

The maximum fault reaction time for communication errors is **100 ms**
(PROFIsafe watchdog). For internal faults, the worst case is the RAM test
period (~80 seconds), which is acceptable per the project safety analysis given
the 1oo2 architecture — a single-channel RAM fault is tolerated as long as
the partner channel remains healthy.
