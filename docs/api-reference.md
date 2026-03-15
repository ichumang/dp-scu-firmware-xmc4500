# API Reference

This document describes the public interfaces exposed by each firmware module.
Full function signatures are available in the corresponding header files.

---

## HAL — `src/hal/xmc4500_bsp.h`

### System Initialization

| Function | Description |
|----------|-------------|
| `bsp_init(void)` | Initialize clock tree, enable SysTick (16 ms), configure DWT, set up GPIO pin mux |
| `bsp_get_channel_id(void)` | Read P0.0 and return `CHANNEL_A` (0) or `CHANNEL_B` (1) |
| `bsp_get_tick_ms(void)` | Return monotonic millisecond counter (from SysTick) |
| `bsp_delay_us(uint32_t us)` | Busy-wait delay using DWT cycle counter |
| `bsp_reset(void)` | Trigger software reset via SCB->AIRCR |

### DWT Cycle Counter

| Function | Description |
|----------|-------------|
| `bsp_dwt_init(void)` | Enable DWT CYCCNT, reset to zero |
| `bsp_dwt_get_cycles(void)` | Return current `DWT->CYCCNT` value |
| `bsp_dwt_cycles_to_us(uint32_t cycles)` | Convert DWT cycles to microseconds at 120 MHz |

### GPIO

| Function | Description |
|----------|-------------|
| `gpio_set(gpio_port_t port, uint8_t pin)` | Drive pin high |
| `gpio_clear(gpio_port_t port, uint8_t pin)` | Drive pin low |
| `gpio_toggle(gpio_port_t port, uint8_t pin)` | Toggle pin state |
| `gpio_read(gpio_port_t port, uint8_t pin)` | Read pin level, return 0 or 1 |
| `gpio_set_safe(gpio_port_t port, uint8_t pin, uint8_t value)` | Set pin with readback verification, return `BSP_OK` or `BSP_ERR_READBACK` |

### UART

| Function | Description |
|----------|-------------|
| `uart_init(uint32_t baud)` | Configure USIC0_CH0 for TX/RX at given baud rate |
| `uart_putc(char c)` | Transmit single character (blocking) |
| `uart_puts(const char *s)` | Transmit null-terminated string |
| `uart_printf(const char *fmt, ...)` | Formatted output (subset: %d, %u, %x, %s, %c) |

### Watchdog

| Function | Description |
|----------|-------------|
| `wdt_init(uint32_t timeout_ms)` | Configure hardware WDT with given timeout |
| `wdt_kick(void)` | Service the watchdog (reset counter) |
| `wdt_disable(void)` | Disable WDT (only allowed in debug builds) |

---

## Ethernet — `src/eth/eth_mac.h`

### MAC Driver

| Function | Description |
|----------|-------------|
| `eth_mac_init(const eth_mac_config_t *cfg)` | Initialize ETH0 MAC, configure DMA descriptors, enable RMII |
| `eth_mac_receive(eth_frame_t *frame)` | Poll RX DMA ring, copy received frame. Returns `ETH_OK` or `ETH_NO_FRAME` |
| `eth_mac_transmit(const eth_frame_t *frame)` | Submit frame to TX DMA ring. Returns `ETH_OK` or `ETH_TX_FULL` |
| `eth_mac_get_link_status(void)` | Return `ETH_LINK_UP` or `ETH_LINK_DOWN` from PHY status register |
| `eth_mac_get_stats(eth_stats_t *stats)` | Read TX/RX counters, error counts, DMA overrun count |

### PHY Driver — `src/eth/phy_ksz8031.h`

| Function | Description |
|----------|-------------|
| `phy_init(void)` | Reset PHY via P2.6, wait for auto-negotiation, configure 100 Mbit FD |
| `phy_read_reg(uint8_t reg)` | MDIO read from PHY register |
| `phy_write_reg(uint8_t reg, uint16_t val)` | MDIO write to PHY register |
| `phy_get_link_speed(void)` | Return negotiated speed: 10 or 100 Mbit |

### UDP/IP — `src/eth/udp_ip.h`

| Function | Description |
|----------|-------------|
| `udp_ip_init(const udp_ip_config_t *cfg)` | Set local IP, MAC, gateway; initialize ARP table |
| `udp_ip_process(const eth_frame_t *frame)` | Demux received frame: ARP / ICMP / UDP. Returns port + payload for UDP |
| `udp_ip_send(uint32_t dst_ip, uint16_t dst_port, uint16_t src_port, const uint8_t *data, uint16_t len)` | Build and transmit a UDP packet |

---

## Communication — `src/comm/`

### PROFIsafe Codec — `profisafe_codec.h`

| Function | Description |
|----------|-------------|
| `profisafe_decode(const uint8_t *raw, uint16_t len, profisafe_telegram_t *out)` | Decode raw bytes into telegram struct, verify CRC1 + CRC2 + seqnr |
| `profisafe_encode(const profisafe_telegram_t *in, uint8_t *raw, uint16_t *len)` | Encode telegram struct to wire format with CRC1 + CRC2 |
| `profisafe_check_watchdog(uint32_t now_ms)` | Check if watchdog timeout has elapsed since last valid RX |

### Heartbeat — `heartbeat.h`

| Function | Description |
|----------|-------------|
| `heartbeat_init(uint32_t interval_ms)` | Set heartbeat interval (default: 500 ms) |
| `heartbeat_tick(uint32_t now_ms)` | If interval elapsed, queue heartbeat frame for TX |
| `heartbeat_process(const uint8_t *data, uint16_t len)` | Process received heartbeat from partner |
| `heartbeat_is_partner_alive(void)` | Return true if partner heartbeat received within 2× interval |

### Diagnostic Frames — `diag_frame.h`

| Function | Description |
|----------|-------------|
| `diag_frame_build(diag_frame_t *frame, const scu_diag_t *diag)` | Assemble diagnostic frame from current counters |
| `diag_frame_process(const uint8_t *data, uint16_t len)` | Process incoming diagnostic request from DP-MainUnit |

---

## Safety Core — `src/safety/`

### Cross-Monitor — `cross_monitor.h`

| Function | Description |
|----------|-------------|
| `cross_monitor_init(channel_id_t id)` | Initialize DPRAM pointers based on channel ID |
| `cross_monitor_submit(const uint8_t *data, uint16_t len, uint32_t flow_hash)` | Write own output data + flow hash to DPRAM |
| `cross_monitor_compare(void)` | Read partner's DPRAM region, compare with own. Returns `XMON_MATCH` or `XMON_MISMATCH` |
| `cross_monitor_alive_tick(void)` | Increment own alive counter in DPRAM |
| `cross_monitor_is_partner_alive(void)` | Check if partner's alive counter has advanced |

### Safe State — `safe_state.h`

| Function | Description |
|----------|-------------|
| `safe_state_init(void)` | Register safe output pins, configure to de-energized on passivation |
| `safe_state_passivate(passivation_reason_t reason)` | Drive all outputs to safe state, log reason, set SAFE flag |
| `safe_state_is_active(void)` | Return true if system is currently in safe state |
| `safe_state_get_reason(void)` | Return the `passivation_reason_t` that triggered safe state |

### RAM Test — `ram_test.h`

| Function | Description |
|----------|-------------|
| `ram_test_init(void)` | Set start address, configure slice size (32 B) |
| `ram_test_run_slice(void)` | Execute March-C on next 32-byte slice. Returns `RAM_OK` or `RAM_FAULT` |
| `ram_test_get_progress(void)` | Return percentage of SRAM tested in current pass (0–100) |

### ROM Test — `rom_test.h`

| Function | Description |
|----------|-------------|
| `rom_test_init(void)` | Set flash start/end address, load reference CRC |
| `rom_test_run_slice(void)` | Compute CRC-32 over next 4 KB. Returns `ROM_OK`, `ROM_FAULT`, or `ROM_PASS_COMPLETE` |
| `rom_test_get_progress(void)` | Return percentage of flash checked in current pass |

### CPU Test — `cpu_test.h`

| Function | Description |
|----------|-------------|
| `cpu_test_alu(void)` | Run predetermined ALU pattern test. Returns `CPU_OK` or `CPU_FAULT` |
| `cpu_test_registers(void)` | Walking-1 pattern on R0–R12, SP, LR. Returns `CPU_OK` or `CPU_FAULT` |
| `cpu_test_nvic(void)` | NVIC priority test. Returns `CPU_OK` or `CPU_FAULT` |

### Flow Monitor — `flow_monitor.h`

| Function | Description |
|----------|-------------|
| `flow_monitor_reset(void)` | Reset sequence counter to zero (called at cycle start) |
| `flow_monitor_enter(uint16_t id)` | Increment counter by `id` (function entry) |
| `flow_monitor_exit(uint16_t id)` | Increment counter by `~id` (function exit) |
| `flow_monitor_verify(uint32_t expected)` | Compare accumulated counter against expected value |

---

## Application — `src/app/scu_state_machine.h`

### State Machine

| Function | Description |
|----------|-------------|
| `scu_sm_init(const scu_config_t *cfg)` | Initialize state machine with configuration, enter `SCU_STATE_INIT` |
| `scu_sm_run(void)` | Execute one cycle of the state machine (called from cycle manager) |
| `scu_sm_get_state(void)` | Return current `scu_state_t` |
| `scu_sm_get_diag(scu_diag_t *diag)` | Copy current diagnostic counters |
| `scu_sm_request_reset(void)` | Request transition from SAFE/ERROR to INIT (requires handshake) |

### Cycle Manager — `cycle_manager.h`

| Function | Description |
|----------|-------------|
| `cycle_manager_init(void)` | Register all slot callbacks, configure DWT timing |
| `cycle_manager_run(void)` | Execute one full 16 ms cycle: RX → Process → TX → Diag |
| `cycle_manager_get_durchlaufzeit_us(void)` | Return last cycle's execution time in microseconds |
| `cycle_manager_get_max_durchlaufzeit_us(void)` | Return worst-case execution time since power-on |

---

## Utilities — `src/util/`

### CRC — `crc.h`

| Function | Description |
|----------|-------------|
| `crc16_profisafe(const uint8_t *data, uint16_t len, uint16_t seed)` | CRC-16 with PROFIsafe polynomial 0x755B |
| `crc32_ieee(const uint8_t *data, uint32_t len, uint32_t seed)` | CRC-32 IEEE 802.3 for flash integrity |

### Safe String — `safe_string.h`

| Function | Description |
|----------|-------------|
| `safe_memcpy(void *dst, const void *src, uint32_t n, uint32_t dst_size)` | Bounded memcpy, returns bytes copied or 0 on overflow |
| `safe_memcmp(const void *a, const void *b, uint32_t n)` | Constant-time comparison, returns 0 for match |
| `safe_memset(void *dst, uint8_t val, uint32_t n, uint32_t dst_size)` | Bounded memset |
