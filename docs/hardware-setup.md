# Hardware Setup

## Development Hardware

### XMC4500 Relax Lite Kit V1

The primary development board for single-channel bring-up and Ethernet
validation. For dual-channel testing, two kits are connected via an external
DPRAM module (ISSI IS61WV10248EDBLL, 16 KB dual-port SRAM on a custom
breakout board).

| Parameter | Value |
|-----------|-------|
| MCU | XMC4500-F100x1024 (LQFP-100) |
| Core | ARM Cortex-M4F @ 120 MHz |
| Flash | 1024 KB |
| SRAM | 160 KB (64 KB DSRAM1 + 64 KB DSRAM2 + 32 KB PSRAM) |
| Ethernet | ETH0 with RMII, KSZ8031RNL PHY |
| Debug | J-Link OB (on-board, SWD) |
| Power | 5V USB or external 3.3V |
| LEDs | LED1 (P1.1), LED2 (P1.0) — active low |
| Buttons | Button1 (P1.14), Button2 (P1.15) — active low, internal pull-up |

---

## Pin Mapping

### Ethernet RMII (ETH0)

| Signal | Pin | Port/Alt | Direction | Notes |
|--------|-----|----------|-----------|-------|
| RMII_TXD0 | P2.8 | P2, ALT1 | Out | Transmit data bit 0 |
| RMII_TXD1 | P2.9 | P2, ALT1 | Out | Transmit data bit 1 |
| RMII_TX_EN | P2.5 | P2, ALT1 | Out | Transmit enable |
| RMII_RXD0 | P2.2 | P2, Input | In | Receive data bit 0 |
| RMII_RXD1 | P2.3 | P2, Input | In | Receive data bit 1 |
| RMII_CRS_DV | P15.9 | P15, Input | In | Carrier sense / data valid |
| RMII_REF_CLK | P15.8 | P15, Input | In | 50 MHz reference clock from PHY |
| MDIO | P2.0 | P2, ALT1 (hw ctrl) | Bidir | Management data I/O |
| MDC | P2.7 | P2, ALT1 | Out | Management data clock |
| PHY_RESET | P2.6 | P2, GPIO | Out | Active-low PHY hardware reset |

### UART Debug (USIC0 CH0)

| Signal | Pin | Port/Alt | Direction | Config |
|--------|-----|----------|-----------|--------|
| UART_TX | P1.5 | USIC0_CH0, DOUT0 | Out | 115200 baud, 8N1 |
| UART_RX | P1.4 | USIC0_CH0, DX0C | In | 115200 baud, 8N1 |

### Channel ID

| Signal | Pin | Port | Notes |
|--------|-----|------|-------|
| CHANNEL_ID | P0.0 | P0, Input | Low = Channel A, High = Channel B |
|  | | | External pull-down (Ch A) or pull-up (Ch B) |

### Status LEDs

| LED | Pin | Active | Purpose |
|-----|-----|--------|---------|
| LED1 (green) | P1.1 | Low | Heartbeat — toggles every 500 ms in RUN |
| LED2 (red) | P1.0 | Low | Safe state indicator — on during SAFE / ERROR |
| LED3 (ext) | P0.1 | High | Ethernet link-up indicator |

### DPRAM Interface (External Bus, dual-channel only)

On the production DP-SCU board, the DPRAM is memory-mapped via the External
Bus Unit (EBU). For the Relax Lite dev setup, the DPRAM breakout uses a
simplified bit-banged parallel interface on Port 0 and Port 3:

| Signal | Pins | Notes |
|--------|------|-------|
| DPRAM_ADDR[0:13] | P0.2–P0.11, P3.0–P3.3 | 14-bit address (16 KB) |
| DPRAM_DATA[0:7] | P3.4–P3.11 | 8-bit data bus |
| DPRAM_WE# | P0.12 | Write enable (active low) |
| DPRAM_OE# | P0.13 | Output enable (active low) |
| DPRAM_CE# | P0.14 | Chip enable (active low) |
| DPRAM_BUSY | P0.15 | Busy flag from dual-port arbitration |

---

## Wiring Diagram (Dev Setup)

```
                                    Ethernet Switch
  ┌─────────────┐                  ┌──────────────┐                  ┌─────────────┐
  │ PC / Debug  │ ──── ETH ─────► │              │ ◄──── ETH ────── │ DP-MainUnit │
  │ (Wireshark) │                  │              │                  │ (simulator) │
  └─────────────┘                  └───┬──────┬───┘                  └─────────────┘
                                       │      │
                                  ETH  │      │  ETH
                                       │      │
                               ┌───────┴──┐ ┌─┴────────┐
                               │ XMC4500  │ │ XMC4500  │
                               │ Ch A     │ │ Ch B     │
                               │          │ │          │
                               │ P0.0=GND │ │ P0.0=3V3 │
                               └────┬─────┘ └─────┬────┘
                                    │   DPRAM      │
                                    │  breakout    │
                                    └──────┬───────┘
                                           │
                                    ┌──────┴──────┐
                                    │ IS61WV10248 │
                                    │ 16KB DPRAM  │
                                    └─────────────┘
```

### Connections Checklist

1. **Channel A** — XMC4500 Relax Lite #1, P0.0 pulled to GND via 10K resistor
2. **Channel B** — XMC4500 Relax Lite #2, P0.0 pulled to 3.3V via 10K resistor
3. **Ethernet** — Both boards connected to same switch, PC with Wireshark on third port
4. **DPRAM** — Breakout board wired to Port 0 / Port 3 per table above
5. **UART** — Channel A UART (P1.5/P1.4) to USB-serial adapter for RTT fallback
6. **Power** — Both boards powered via USB; DPRAM breakout powered from Channel A 3.3V rail

---

## Debug Setup

### J-Link + SEGGER RTT

The on-board J-Link supports SEGGER Real-Time Transfer (RTT) for zero-overhead
printf-style debug output. RTT is preferred over UART because it does not
consume cycle time.

```bash
# Start J-Link GDB Server
JLinkGDBServerCLExe -device XMC4500-1024 -if SWD -speed 4000

# In another terminal: start RTT client
JLinkRTTClient

# Or use RTT Viewer for graphical output
JLinkRTTViewerExe
```

### Flash Programming

```bash
# Via J-Link Commander
JLinkExe -device XMC4500-1024 -if SWD -speed 4000 -autoconnect 1 << EOF
r
h
loadbin build/dp_scu_firmware.bin, 0x08000000
r
g
q
EOF
```

### GDB Debugging

```bash
# Start GDB server (see above), then:
arm-none-eabi-gdb build/dp_scu_firmware.elf
(gdb) target remote localhost:2331
(gdb) monitor reset
(gdb) load
(gdb) break main
(gdb) continue
```

---

## Wireshark Capture Setup

For validating PROFIsafe telegrams on the wire:

1. Connect PC to the same Ethernet switch as both XMC4500 boards
2. Configure switch port mirroring (or use a hub for passive capture)
3. Apply Wireshark display filter: `udp.port == 0x9876 || udp.port == 0x9877`
4. Use the custom Lua dissector (`tools/profisafe_dissector.lua`) for decoded view

### Expected Packet Rate

| Frame Type | Port | Interval | Size (ETH) |
|------------|------|----------|------------|
| PROFIsafe telegram | 0x9876 | 16 ms | 60–137 B |
| Heartbeat | 0x9877 | 500 ms | 60 B |
| Diagnostic | 0x9878 | On demand | 60–256 B |

---

## Clock Configuration (System Control Unit)

The XMC4500 clock tree is configured in `xmc4500_bsp_init()`:

| Clock | Source | Frequency |
|-------|--------|-----------|
| fOSC | External crystal | 12 MHz |
| fPLL | fOSC × 10 | 120 MHz |
| fSYS | fPLL | 120 MHz |
| fCPU | fSYS / 1 | 120 MHz |
| fPB | fSYS / 1 | 120 MHz |
| fCCU | fSYS / 1 | 120 MHz |
| fETH | External (RMII REF_CLK) | 50 MHz |

The PLL lock is verified before the firmware proceeds past `SCU_STATE_INIT`.
If the PLL does not lock within 10 ms, the system enters `SCU_STATE_ERROR`.
