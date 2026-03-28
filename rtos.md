# RTOS Task & Cell Service Architecture

---

## RTOS Overview

The syscon runs a uITRON-based RTOS (NORTi or similar). Tasks communicate via message queues and semaphores. The firmware creates 20+ tasks during boot, each with a dedicated stack and priority level

---

## Task Map

All tasks created via `rtos_create_task(id, arg)`. SSM creates tasks 5 and 22 directly at init, then `system_bringup_create_tasks` creates the rest during POR

| Task ID | IDA Label | Entry Point | Name | Description |
|---------|-----------|-------------|------|-------------|
| 2 | RTOS_TASK_UISW | - | **UISW** | User interface switches (power/eject button input, debounce) |
| 3 | RTOS_TASK_UILED | - | **UILED** | LED driver (green/red LED state machine, blink patterns) |
| 4 | RTOS_TASK_COMMRECV | `sub_12BA2` | **COMMRECV** | BE-SC communication receiver. Receives raw SPI messages from Cell via Southbridge, parses protocol framing, routes to service tasks |
| 5 | (early WMM) | - | **WMM** | Thermal watchdog manager (created by SSM before bringup) |
| 6 | RTOS_TASK_FIRMUD | `firmware_update_task` (0xE20A) | **FIRMUD** | Firmware update handler. Receives encrypted firmware image, decrypts and writes it somewhere (flash?) (`decrypt_and_write_to_flash`) |
| 7 | RTOS_TASK_SB | `sub_6912` | **SB** | Southbridge interrupt handler. Monitors SB mailbox, dispatches registered interrupt handlers for SB events |
| 8 | RTOS_TASK_POWERSEQ | - | **POWSEQ** | Power sequencer. Executes bringup/shutdown rail sequences under SSM control |
| 9 | RTOS_TASK_SERV_DIAG | `sub_12A6C` | **SERV_DIAG** | Diagnostics service task. Handles diagnostic queries from Cell |
| 10 | RTOS_TASK_SERV_HDMI | - | **SERV_HDMI** | HDMI/AV output service. Controls SiI HDMI transmitter IC via I2C |
| 11 | RTOS_TASK_SERV_SECU | - | **SERV_SECU** | Security service |
| 14 | RTOS_TASK_IDLE | - | **IDLE** | Idle / background task |
| 15 | RTOS_TASK_SERV_MISC2 | - | **SERV_MISC2** | Second miscellaneous service task |
| 16 | RTOS_TASK_SERV_THERM | `sub_1401E` | **SERV_THERM** | Thermal monitoring service. Processes temperature readings, triggers alarms |
| 18 | RTOS_TASK_SERV_MISC | `sub_1428C` | **SERV_MISC** | **Cell service dispatcher** - The main Cell-facing service handler. Routes by SID to all 15 services (SDA, DEVPM, SETCFG, SYSPM, NVS, NOTIF, VERS, OSWDT, CONSOLE, PATCH, LS, STORAGE, etc.) |
| 19 | RTOS_TASK_WMM0 | - | **WMM0** | Thermal watchdog 0 (fast poll, 10ms interval, desc 13) |
| 21 | RTOS_TASK_WMM1 | - | **WMM1** | Thermal watchdog 1 (slow poll, 100ms interval, desc 14) |
| 22 | (unnamed) | - | **(init)** | Created by SSM at boot before bringup. Init/housekeeping |
| 24 | RTOS_TASK_INTRNOTIF | - | **INTRNOTIF** | Async interrupt notification delivery to Cell |
| 25 | RTOS_TASK_SIRCS | - | **SIRCS** | Sony SIRCS IR remote control receiver |
| - | (SSM itself) | `ssm_task_main` (0x10FC4) | **SSM** | System State Manager. Created by RTOS init before all others. Runs the central power state machine. Uses message queue 4 |
| - | (error logger) | `sub_1A416` | **ERRLOG** | Error log writer. Writes to NVS circular buffer 0x3700-0x37FF |
| - | (shell) | - | **SHELL** | UART command-line debug shell. Not a formal RTOS task, runs in interrupt/polling context |

Task creation order during bringup: SSM creates 22, 5 -> then `system_bringup_create_tasks` creates 8 -> 19 -> 21 -> 6 -> 4 -> 2 -> 14 -> 3 -> 7 -> 9 -> 10 -> 11 -> 16 -> 18 -> 15 -> 24 -> 25 -> releases startup barrier

---

## SSM (System State Manager) - Task 4

Full analysis in [ssm.md](https://github.com/sagemono/syscon-documentation/blob/main/ssm.md). Key summary:

### State Machine

9 major states, 8 actions, transition table at ROM `byte_3FD94`:

```
Action:             0(bringup) 1(shutdown) 2(reset) 3(fatal) 4(ok) 5(fail) 6(error) 7(clear)
State 0 (Off):            1         -          -        8       -      -       -        -
State 1 (Boot):           -         -          -        -       2      4       3        -
State 2 (Run early):      1         5          -        7       -      -       -        -
State 3 (Err boot):       -         5          -        7       -      -       -        -
State 4 (Run full):       -         5          1        7       -      -       -        -
State 5 (Shutdown):       -         -          -        -       0      -       0        -
State 6 (Fatal off):      -         -          -        -       -      -       -        0
State 7 (Err recov):      -         -          -        -       6      -       6        -
State 8 (Fatal recov):    -         -          -        -       6      -       6        -
```

### SSM Command Codes (32-bit, top byte = action)

| Top Byte | Action | Example Code |
|----------|--------|-------------|
| 0x11 | Bringup (power on) | 0x11000000 |
| 0x12 | Shutdown | 0x12010000 (normal), 0x12020000 (from Cell) |
| 0x13 | Reset | 0x13020000 (panic reset) |
| 0x3F | Forced shutdown | 0x3F0200FE |
| 0x40 | Re-entry shutdown | 0x40000000 |

### Bringup Sub-States (UART log format: `[SSM] state: XXYY -> XXYY`)

```
0000 -> 0101 -> 0201 -> 0102 -> 0202 -> 0103 -> 0203 -> 0104 -> 0204 -> 0105 -> 0400
 off    ph1s    ph1c    ph2s    ph2c    ph3s    ph3c    ph4s    ph4c    ph5s    running
```

Failure at any phase: e.g. `0103 -> 0303` (PowSeq Fail at phase 3).

---

## Cell Service Dispatcher (`sub_1428C`)

This is the main Cell-facing communication handler. It runs as a task that loops forever, receiving messages from the Cell via the SPI/FlexIO mailbox. Each message contains a Service ID (SID) and payload. The dispatcher routes by SID, each handler processes the command and sends a response back to the Cell.

### Message Format

Messages arrive as `sc_msg_hdr` structures:
- `hdr.sid` - Service ID (routing key)
- `hdr.len` - Payload length (1, 2, 4, or 8 typically)
- `hdr + 12` - Payload data (sub-command byte + arguments)

### Complete Service Table

| SID | Hex | Source File | Service Name | Handler |
|-----|-----|------------|--------------|---------|
| 3 | 0x03 | serv_sda.c | SDA | `sub_1FF20` |
| 16 | 0x10 | - | DEVPM | `sub_21654` |
| 18 | 0x12 | serv_setconf.c | SETCFG | `sub_1EB54` |
| 19 | 0x13 | serv_syspm.c | SYSPM | `sub_1F88C` |
| 20 | 0x14 | - | NVS | `sub_21144` |
| 21 | 0x15 | - | SIRCS | `sub_23852` |
| 22 | 0x16 | serv_notif.c | NOTIF | `sub_218CA` |
| 23 | 0x17 | - | INTR_NOTIF | `sub_21E80` |
| 24 | 0x18 | - | VERS | `sub_20EEE` |
| 27 | 0x1B | - | LIVELOCK | `sub_21B60` |
| 28 | 0x1C | serv_oswdt.c | OSWDT | `sub_22FF4` |
| 32 | 0x20 | serv_cons.c | CONSOLE | `sub_20182` |
| 45 | 0x2D | serv_patch.c | PATCH | `sub_22E2C` |
| 64 | 0x40 | - | LS | `sub_2042E` |
| 80 | 0x50 | - | STORAGE | `sub_2330E` |

### Independent Service Tasks

Not all services route through the main dispatcher. These tasks have their own RTOS message loops and receive messages directly from COMMRECV:

| Task ID | Name | Entry Point | Source | Description |
|---------|------|-------------|--------|-------------|
| 9 | SERV_DIAG | `sub_12A6C` | - | Diagnostics service. Handles SID bytes 0 (query), 3, 4. Logs as `[SERV DIAG]`. |
| 10 | SERV_HDMI | (hdmi task) | - | HDMI/AV control. Manages SiI HDMI transmitter via I2C. Logs as `[hdmiERROR]`. Uses semaphore 0x41 for I2C bus exclusion. |
| 11 | SERV_SECU | (security task) | - | Security/authentication service. |
| 16 | SERV_THERM | `sub_1401E` | - | Thermal monitoring. Handles SID bytes 0 (temp query), 3 (threshold set), 4 (config). Logs as `[SERV THERM]`. |

---

## SID 3: SDA (System Data Area) - `serv_sda.c`

Raw EEPROM byte-level access service. Address range 0x48000-0x4FFFF.

| Cmd | Name | Description |
|-----|------|-------------|
| 0x00 | READ | Read arbitrary bytes from EEPROM. Max 2047 bytes. |
| 0x01 | WRITE | Write arbitrary bytes to EEPROM. |

**Access Control:**
- Blocks ALL access to HV area: 0x48100-0x487FF (returns error 3)
- Blocks WRITE access to QA token area: 0x48D20-0x48D3F
- Read/write validated against start+length to prevent spanning into protected regions

This is distinct from SID 20 (NVS) which uses region-based abstraction. SDA gives raw byte access with explicit protection zones.

---

## SID 16: DEVPM (Device Power Management)

Controls power states of GPU, ATA devices, and PCI bus.

| Cmd | Name | Handler | Description |
|-----|------|---------|-------------|
| 0x10 | GET_GPU_POWER_STATE | inline | Returns RSX power state (0=off, 1=standby, 2=on) |
| 0x11 | CONTROL_GPU_POWER_STATE | `sub_21338` | Set RSX power state |
| 0x20 | GET_ATA_DEVICES_POWER_STATE | inline | Returns HDD/BD power (0=off, 3=on) |
| 0x21 | CONTROL_ATA_DEVICES_POWER_STATE | `sub_2169C` | Set ATA power |
| 0x30 | GET_PCI_BUS_POWER_STATE | inline | Returns SB PCI bus state |
| 0x31 | CONTROL_PCI_BUS_POWER_STATE | `sub_213C4` | Set PCI power |

---

## SID 18: SETCFG (Set Configuration) - `serv_setconf.c`

The main hardware configuration service. Full command tree documented in eeprom_map.md. Handles:

### Command Dispatch (byte 0 of payload)

| Cmd | Sub | Name | Description |
|-----|-----|------|-------------|
| 0 | 0x00 | GET_XDR_CONFIG | Builds 112-byte XDR config block from ROM defaults + NVS overrides (`sub_1E548`) |
| 0 | 0x01 | SET_XDR_CONFIG | Write XDR config to lv0 (`sub_1E230`) |
| 0 | 0x02 | XDR_ASSERT/DEASSERT | Assert or deassert XDR reset lines |
| 0 | 0x10 | GET_CLOCK_FREQ | Read clock frequency from PLL register lookup table (`sub_1E430`) |
| 1 | 0x00 | (reserved) | Returns 260 bytes of zeros |
| 2 | 0x00 | GET_RSX_VID | Returns 3 bytes: [device_vid, nvs_vid, nvs_correction] (`sub_1EDB8`) |
| 2 | 0x10 | GET_CLOCK_FREQ_RSX | RSX clock frequency |
| 2 | 0x20 | GET_RSX_VID_FULL | Full RSX VID with 7-byte response |
| 2 | 0x22 | SET_RSX_VID | Write RSX VID to NVS 0x361D (validates 127-129) |
| 2 | 0x23 | GET_RSX_VARIANT | Read RSX hardware variant byte |
| 3 | 0x00 | GET_BE_CONFIG | Returns BE PLL enable config byte (NVS 0x3969) (`sub_1EBE6`) |
| 3 | 0x01 | GET_BE_CHIPID | Returns 22 bytes: 16-byte chip ID + 6-byte info (`sub_1E4F8`) |
| 3 | 0x02 | GET_BOARD_ID | Returns `byte_2000F5C` (board revision) |
| 3 | 0x03 | CONTROL_VRM | VRM mode control (enable/disable) |
| 3 | 0x10 | GET_CLOCK_FREQ_BE | BE clock frequency |
| 3 | 0x20 | GET_BE_VID | Returns 3 bytes: [device_vid, nvs_vid, nvs_correction] (`sub_1EC2E`) |
| 3 | 0x22 | SET_BE_VID | Write BE VID to NVS 0x361C (validates 127-129) |
| 3 | 0x23 | GET_BE_VARIANT | Read BE hardware variant byte |
| 4 | 0x10 | GET_CLOCK_FREQ_SB | SB clock frequency |
| 5 | 0x00 | CONTROL_HDMI_POWER | HDMI power on (0x00) / off (0x01) |
| 5 | 0x01 | SET_HDMI_MODE | HDMI output mode control (`sub_1EBB8`) |
| 6 | 0x00 | GET_BOARD_VERSION | Returns board version string from ROM (`sub_1E2AA`) |
| 0x22 | - | GET_DEVICE_CONFIG | Read NVS 0x3293 + 0x3294 (1+8 bytes device config) |
| 0x40 | - | GET_ERROR_LOG | Read error log entry by index (`sub_1A2D6`) |
| 0x41 | - | GET_POWER_STATS | Read power-on time, bringup/shutdown counts (`sub_1E642`) |

### Board Version Table (ROM 0x3FF88)

The SETCFG service checks the hardware variant byte (NVS 0x310E) against a table to determine which board is running:

| Key | [Board ID](https://www.psdevwiki.com/ps3/Platform_ID) | Notes |
|-----|----------|-------|
| 0xFFFFFFFF | "unknown" | Default / unrecognized |
| 0x30 | "Shr3.0" | Shreck 3.0 |
| 0x31 | "Shr3.1" | Shreck 3.1 |
| 0x33 | "Shr3.2" | Shreck 3.2 |
| 0x32 | "Shr3.2-4" | Shreck 3.2 variant |
| 0x34 | "Shr3.3" | Shreck 3.3 |
| 0x40 | "Shr4.0" | Shreck 4.0 |
| 0x10000010 | "Cok01" | Retail prototypes and onwards |

If the hardware variant doesn't match any entry, logs: `[SERV SETCFG] <<<WARNING>>> Board version does not match.`

---

## SID 19: SYSPM (System Power Management) - `serv_syspm.c`

The most complex service. Has two dispatch paths: mode 0 (request/response) and mode 16 (async notification).

### Mode 0 Commands (Request/Response via `sub_1F700`)

| Cmd | Name | Handler | Description |
|-----|------|---------|-------------|
| 0x00 | SHUTDOWN | `sub_1F05A(0,0)` | Clean shutdown. Builds SSM code 0x12020000. |
| 0x01 | RESET | `sub_1F05A(0,1)` | Warm reset. Builds SSM code 0x12020000 with restart flag. |
| 0x10 | GET_STATUS | `sub_1EF3A` | Returns 16-byte power status: [status, syspm_param(4B), wake_src(2B), shutdown_mode(2B), power_flags(2B)] |
| 0x11 | SET_SYSPM_PARAM | `sub_1F102` | Extended shutdown with wake parameter. Payload: [cmd, wake_hi, wake_lo, panic_flag]. |
| 0x12 | SET_WAKE_SOURCE | `sub_1F902` | Write 4-byte SYSPM parameter to NVS 0x3614 (mirrored). Validates alignment. |
| 0x13 | GET_SYSPM_PARAM | `sub_1F986` | Read current SYSPM parameter from NVS 0x3614. Returns [status(1B), value_BE(4B)]. |
| 0x20 | CONTROL_WAKE | `sub_1F9F6` | Configure wake sources. Sub-cmds: 0x20=set RTC alarm, 0x28=set wake mask. Supports wake types 1-3. |
| 0x23 | RTC_STATUS | `sub_1FB64` | Returns [status(1B), rtc_valid(1B), rtc_count_BE(2B)]. Checks RTC hardware and error count. |
| 0x30 | SET_RTC_EPOCH | `sub_1FBCC` | Write new RTC time to NVS 0x3618. Validates: new > current (err 3=past, 2=too close, 4=invalid). |
| 0x31 | GET_RTC_EPOCH | `sub_1FC9C` | Read current RTC time. Returns [status(1B), padding(3B), time_BE(4B)]. |
| 0x32 | CLEAR_RTC | `sub_1FC76` | Write 0xFFFFFFFF to NVS 0x3618 (invalidate stored RTC). |
| 0x33 | NOTIFY_RTC_ALARM | `sub_1F552` | Send RTC alarm notification to Cell. Uses stored comm tag from CONTROL_WAKE. |
| 0x40 | CONTROL_WATCHDOG | `sub_1FD02` | Enable/disable OS watchdog. 0=disable (clears byte_2000F62), 1=enable (stores comm tag for alarm delivery). |

### Mode 16 Commands (Async Button Events via `sub_1F49E`)

| Cmd | Name | Description |
|-----|------|-------------|
| 0x21 | POWER_BUTTON_PRESSED | Power button down event |
| 0x22 | POWER_BUTTON_RELEASED | Power button up event |
| 0x29 | RESET_BUTTON_PRESSED | Reset button down event |
| 0x2A | RESET_BUTTON_RELEASED | Reset button up event |
| 0x41 | ALARM_EVENT | RTC alarm fired (`sub_1F552`) |

Button events generate async notifications to the Cell via `msg_post(19, ...)`. The handler distinguishes press vs release and POWER vs RESET buttons.

### `sub_1F05A` - Core Power Dispatch

Translates parameters into SSM command codes:

| Input | SSM Code | Meaning |
|-------|----------|---------|
| Default | 0x12020000 | Normal shutdown from Cell |
| panic=1 | 0x13020000 | Panic reset |
| flag3=1 | 0x12020010 | Shutdown with flag |
| flag4=1 | 0x12020060 | Shutdown with extended flag |
| flag5=1 | 0x3F0200FE | Forced shutdown |

Restart bit: if wake parameter high or low byte is 1 or 2, sets bit 0 of the SSM code (restart flag).

---

## SID 20: NVS (Non-Volatile Storage)

Region-based EEPROM access for the Cell. Handler: `sub_20FC8`.

| Cmd | Name | Description |
|-----|------|-------------|
| 0x10 | WRITE | Write to NVS region. Payload: [cmd, region_id, offset, length, data...] |
| 0x20 | READ | Read from NVS region. Payload: [cmd, region_id, offset, length]. Response: [status, region_id, offset, length, data...] |

Region map (from decompiled base address calculations):

| Region ID | Hex | NVS Base | Name | Notes |
|-----------|-----|----------|------|-------|
| 0 | 0x00 | 0x7000 | BD (Bluray Drive) | 256 bytes max |
| 1 | 0x01 | 0x7100 | HV (HyperVisor) | 256 bytes max |
| 2 | 0x02 | 0x7200 | Token | 256 bytes max |
| 3 | 0x03 | 0x7300 | SysData | 256 bytes max |
| 16 | 0x10 | 0x2F00 | Industry | 256 bytes max |
| 32 | 0x20 | 0x3000 | Customer Service | 256 bytes max |
| 255 | 0xFF | RAM 0x200C210 | Direct RAM buffer | Read/write to syscon RAM directly |

Each region supports offset 0x00-0xFF. Offset + length validated to not exceed 0x100. Region 255 reads/writes directly to RAM address 0x200C210+offset without touching EEPROM.

---

## SID 21: SIRCS (Sony IR Remote Control) - `serv_sircs.c`

Full SIRCS infrared remote control support. The hardware is wired up on retail systems, the `RMC_IN` pad on the syscon connects to pin 3 of the CSW-001 front panel switch board and pin 17 on the CN4009 service connector (unconfirmed). The receiver components were populated on CEB prototypes but left unpopulated on retail boards, though the traces and pads remain. In 2020, [M4j0r confirmed](https://x.com/MinaRalwasser/status/1222606193883058176) that the SIRCS code path is present from CECHA through the CECHE models.

Wiring a standard TSOP38238 (38kHz IR receiver) to the RMC_IN pad and powering it from 3.3V_EVER supposedly enables full IR remote control of the console.

### Service Commands

Handler: `sub_239A0`. Source: `serv_sircs.c`.

| Mode | Cmd | Name | Handler | Payload | Description |
|------|-----|------|---------|---------|-------------|
| 0 | 0x10 | ENABLE/DISABLE | inline | [cmd, mode(1B)] | Enable IR receiver. Mode 0=disable (clear comm tag), 1=all-pass (forward every code), 2=repeat-suppress (filter consecutive duplicates). Stores comm tag for async event delivery. |
| 0 | 0x11 | SET_FILTER_TABLE | `sub_23A62` | [cmd, flag(1B), code0(3B), code1(3B), code2(3B), code3(3B)] (14B total) | Configure 4-entry IR code filter. flag=0 enables filtering, flag=1 disables. Each code is a 24-bit SIRCS value (big-endian). Non-matching codes are suppressed in mode 2. |
| 0 | 0x12 | READ_IR_DATA | `sub_23AC2` | [cmd] (1B) | Returns 13-byte response: 4x [code_high, code_mid, code_low] for the 4 configured button mapping codes (power, reset, eject, custom). |
| 0 | 0x13 | SET_BUTTON_MAP | `sub_2359C` | [cmd, power(3B), reset(3B), custom(3B), eject(3B)] (13B total) | Configure which IR codes map to physical console functions. Each code is 24-bit big-endian. 0xFFFFFF = disabled. Persists to NVS 0x3608-0x3613. |
| 16 | 0x20 | IR_EVENT_PUSH | `sub_23A2C` -> `sub_23704` | (async, 4B) | Async notification: [0x20, code_high, code_mid, code_low]. Sent to Cell when a valid IR code is received. |

### Hardware Interface

**SB Registers:**

| Address | Name | Description |
|---------|------|-------------|
| 0x3105900 | GPIO config | Write 0x5F to enable RMC_IN input, 0xCF to disable |
| 0x3109000 | SIRCS control | Bit 7: clear capture overflow |
| 0x3109008 | SIRCS status | Bit 0: capture data available, Bit 2: overflow |
| 0x3109014 | SIRCS capture data | 8-bit timer count representing pulse width |

The Southbridge contains a dedicated hardware timer/capture unit for the RMC_IN pin. It automatically measures pulse widths and stores them in a FIFO. The SIRCS task reads up to 10 capture values per interrupt event via `sub_16CB6`.

### SIRCS Protocol Decoder

The firmware implements a complete SIRCS decoder supporting all three standard variants.

**Supported Variants:**
- 12-bit SIRCS: 7-bit command + 5-bit device address
- 15-bit SIRCS: 7-bit command + 8-bit device address
- 20-bit SIRCS: 7-bit command + 13-bit extended device address

**Pulse Classification (`sub_16C80`):**

The raw timer capture value is compared against 5 threshold ranges (min/max pairs at context+7):

| Index | Meaning | Typical SIRCS Timing |
|-------|---------|---------------------|
| 0 | Error / unrecognized | - |
| 1 | Short pulse (logic 0) | ~0.6ms |
| 2 | Long pulse (logic 1) | ~1.2ms |
| 3 | Start bit (header) | ~2.4ms |
| 4 | Gap / end of frame | >2.4ms or timeout |
| 253 | Frame end marker | Internal: all bits received |
| 255 | Invalid / no data | - |

**Decoder State Machine (`sub_1BAE4`):**

```
State 0 (Idle)
    |-- Event 3 (start bit) --> State 1
    |-- anything else --> stay in State 0

State 1 (Start received, waiting for first data)
    |-- Event 4/253 (gap/end) --> State 0 (false start)
    |-- Event 1 or 2 (data bit) --> State 3

State 2 (Between bits, previous bit decoded)
    |-- Event 4 (gap) --> State 0 (abort)
    |-- Event 253 (frame end) --> validate bit count:
    |       if 12, 15, or 20 --> State 4 (COMPLETE)
    |       else --> State 0 (invalid frame)
    |-- Event 1 or 2 (next data bit) --> State 3

State 3 (Processing data bit)
    sub_1BB8C does the actual bit decode:
    |-- Event 1 (short pulse) --> accumulator unchanged (0 bit), bit_count++, --> State 2
    |-- Event 2 (long pulse) --> accumulator |= (1 << bit_count), bit_count++, --> State 2
    |-- Event 3 (start bit) --> State 0 (reset, new frame starting)

State 4 (Complete)
    Return 4 to caller, auto-reset to State 0
```

Bits are accumulated LSB-first into a 32-bit accumulator at context+8. The bit counter at context+4 tracks how many bits have been received. Only frames with exactly 12, 15, or 20 bits are accepted as valid SIRCS codes.

**Bit Reordering (`sub_1BABC`):**

After decoding, the raw accumulator value is reordered to separate command from device address:

```
final_code = (raw & 0x7F) | ((raw >> 7) << 8)
```

This extracts the low 7 bits as the SIRCS command byte and shifts the remaining bits (device address) into the upper bytes. The result is a 24-bit code stored big-endian as `[device_high, device_low, command]`.

For 12-bit SIRCS: `device_addr[4:0] << 8 | command[6:0]` (5-bit device + 7-bit command)
For 15-bit SIRCS: `device_addr[7:0] << 8 | command[6:0]` (8-bit device + 7-bit command)
For 20-bit SIRCS: `device_addr[12:0] << 8 | command[6:0]` (13-bit device + 7-bit command)

### Button Mapping (NVS Persistent)

Four 24-bit codes stored at NVS 0x3608-0x3613 define which IR codes trigger physical console functions. These are loaded at boot by `load_thresholds_from_nvs` (`sub_23702`) and can be reconfigured by the Cell via SID 21 cmd 0x13. Written to NVS via `sub_23984`

| NVS Offset | Size | Context | Function | Handler Called |
|------------|------|---------|----------|---------------|
| 0x3608 | 3B | word_2000FEA[5] | Power on/off | `sub_1F360(256)` -- identical to physical power button press |
| 0x360B | 3B | word_2000FEA[7] | Reset | `sub_1F026` -- identical to physical reset button press |
| 0x360E | 3B | word_2000FEA[11] | Eject | `sub_1E0FE(0)` -- identical to physical eject button press |
| 0x3611 | 3B | word_2000FEA[9] | Custom | No direct handler, forwarded to Cell via async notification only |

Value 0xFFFFFF (all bytes 0xFF) = disabled / not configured.

When a decoded IR code matches one of the power/reset/eject mappings, the syscon executes the corresponding hardware function directly, no Cell involvement needed. The console can be powered on from standby via IR, just like the physical power button.

### Code Filter Table (RAM)

A 4-entry filter table at `dword_200B710` controls which IR codes are forwarded to the Cell as async notifications. Configured via SID 21 cmd 0x11.

```
dword_200B710[0] = filter code 0 (24-bit, 0xFFFFFF = wildcard)
dword_200B714    = filter code 1
dword_200B718    = filter code 2
dword_200B71C    = filter code 3
```

The filter supports wildcard matching: if the low byte of a filter entry is 0xFF, it matches ANY command with the same device address (only the upper bytes are compared). This allows subscribing to all commands from a specific device.

### Notification Modes

| Mode | Name | Behavior |
|------|------|----------|
| 0 | Disabled | All decoded IR codes silently dropped |
| 1 | All-pass | Every decoded code forwarded to Cell. No repeat filtering. First reception of any code is always forwarded. |
| 2 | Repeat-suppress | Consecutive duplicate codes are filtered, only the first press is forwarded. A different code or 0xFFFFFF (key release / gap) resets the filter. |

### Key Repeat Handling

`sub_1BAD4` is the key repeat handler, called when event 20481 (0x5001) fires. It sends `0xFFFFFF` as the IR code, which the notification handler interprets as a "key released" event. This allows the Cell to distinguish between a held button and a new press.

### ROM Code Table

25 entries at ROM `0x40334` are copied to RAM `dword_200B720` during init (`sub_23C00`). These are 32-bit little-endian values, likely function pointers to per-button handler routines in the `0x15xxx` address range, NOT raw IR codes. The init function also applies a CRC-16 check and can patch the table from NVS if a valid override block exists.

### Task Architecture

```
Hardware: RMC_IN pin -> SB timer capture (0x3109000-0x3109014)
                |
Task 25 (SIRCS): sub_67B0 -- main loop
    |-- shell_sleep(-1)  (wait for SB interrupt)
    |-- sub_23BA0: enable RMC_IN GPIO (write 0x5F to SB 0x3105900)
    |-- loop:
    |     |-- sub_16CB6: read capture FIFO (up to 10 pulse widths per event)
    |     |-- event 0: sub_680E -> sub_1BAE4 (decode pulses)
    |     |     |-- state machine processes each pulse
    |     |     |-- on complete frame (state 4):
    |     |     |     |-- sub_1BABC: reorder bits -> 24-bit code
    |     |     |     |-- sub_237EC: package into message, post to service queue
    |     |-- event 20481 (0x5001): sub_1BAD4 -> send 0xFFFFFF (key repeat)
    |     |-- event 20480 (0x5000): sub_16C68 -> disable, return to sleep
                |
Task 18 (SERV_MISC): sub_1428C -> SID 21 dispatch
    |-- sub_23852: route by mode (0=request, 16=async push)
    |-- sub_239A0: process Cell commands (enable, filter, button map, read)
    |-- sub_23704: process decoded IR code
    |     |-- check against power/reset/eject mappings -> direct hardware action
    |     |-- check against filter table -> forward to Cell or suppress
    |     |-- sub_23652: send async notification via msg_post(0x15, ...)
```

---

## SID 22: NOTIF (Notification) - `serv_notif.c`

Hardware notification and control service.

| Cmd | Name | Handler | Description |
|-----|------|---------|-------------|
| 0x10 | QUERY_CUR_LED_STAT | inline | Returns 2 bytes: [green_mode, red_mode] |
| 0x11 | CONTROL_LED | `sub_11212`/`sub_111FC` | Set LED modes. Values 0-4 per LED, 0xFF=no change. Modes: 0=off, 1=on, 2=blink slow, 3=blink fast, 4=blink pattern |
| 0x18 | CONTROL_BD_HDD_LED | inline | Control BD/HDD activity LEDs. Values: 0=off, 1=on |
| 0x20 | RING_BUZZER | `sub_114A8` | Drive piezo buzzer. Args: [pattern, freq_BE(4B)], duration 1-32 repetitions |

---

## SID 23: INTR_NOTIF (Interrupt Notification)

Async event push from syscon to Cell. Two modes:

| Mode | Handler | Description |
|------|---------|-------------|
| 0 | `sub_21E9E` | Request/response: Cell queries for pending interrupts |
| 16 | `sub_21F0C` | Push: syscon pushes interrupt notification to Cell |

---

## SID 24: VERS (Version Service)

Returns version information for any service by SID.

| Cmd | Handler | Description |
|-----|---------|-------------|
| 0x01 | `sub_20DB8` | Takes target SID in payload. Returns 4 bytes: [status, target_sid, major_ver, minor_ver] |

Known version responses:

| Target SID | Major | Minor |
|-----------|-------|-------|
| 3 (SDA) | 1 | 0 |
| 16 (DEVPM) | 1 | 0 |
| 17 | 1 | 0 |
| 18 (SETCFG) | 1 | 10 |
| 19 (SYSPM) | 1 | 6 |
| 20 (NVS) | 5 | 1 |
| 21 (SIRCS) | 1 | 0 |
| 22 (NOTIF) | 2 | 1 |
| 24 (VERS) | 1 | 0 |
| 28 (OSWDT) | 1 | 0 |
| 32 (CONSOLE) | 2 | 1 |
| 48 (PATCH) | 5 | 0 |
| 64 (LS) | 1 | 0 |
| 80 (STORAGE) | 1 | 0 |

---

## SID 27: LIVELOCK (Bus Livelock Detection)

Cell BE bus livelock detection and resolution.

| Mode | Handler | Description |
|------|---------|-------------|
| 0 | `sub_21B82` | Configure livelock handler. Sub-cmd 0x10: 0=disable (clear comm tag), 1=enable (store comm tag for async notifications) |
| 16 | `sub_21BF4` | Livelock event notification push |

---

## SID 28: OSWDT (OS Watchdog Timer) - `serv_oswdt.c`

Monitors CellOS liveness. If the Cell stops sending heartbeats, the syscon forces a reset.

| Cmd | Name | Handler | Description |
|-----|------|---------|-------------|
| 0x10 | START_OSWDT | `sub_344F6` | Start watchdog with given timeout. Args: [period(1B), mode(1B)]. Mode: 0 or 1. |
| 0x11 | STOP_OSWDT | `sub_3450C` | Stop watchdog timer |
| 0x12 | CLEAR_OSWDT | `sub_344E0` | Heartbeat / keepalive. Cell must send periodically to prevent reset. |

---

## SID 32: CONSOLE - `serv_cons.c`

Debug bridge that lets the Cell inject commands into the syscon's internal shell parser remotely, without physical UART access.

> **Note:** The Cell does NOT connect to the syscon via UART. All Cell-to-syscon communication goes through the **Southbridge SPI bus** (BE-SC protocol). SID 32 receives characters over that SPI link, then feeds them into the same command parser that the hardware UART debug header uses. The function `sub_11814(1)` sets a `log_gate` flag to switch the shell into remote input mode.

Data path: `Cell -> Southbridge -> SPI bus -> COMMRECV (task 4) -> SERV_MISC (task 18) -> SID 32 -> sub_119D6 (shell char feed) -> command parser`

| Cmd | Name | Handler | Description |
|-----|------|---------|-------------|
| 0x00 | INJECT_CHARS | `sub_119D6` loop | Feed characters one by one into the shell parser. Accepts printable ASCII (0x20-0x7E), newline (0x0A), carriage return (0x0D). lv0/lv1 can execute any syscon shell command (r, w, eepcsum, etc.) this way. |
| 0x01 | SET_CONSOLE_MODE | `cmd_console_mode` | Switch shell operating mode |

---

## SID 45: PATCH - `serv_patch.c`

Firmware version query and runtime patching service.

| Cmd | Name | Handler | Description |
|-----|------|---------|-------------|
| 0x10 | GET_HW_REVISION | `parse_revision` | Returns 2-byte hardware revision |
| 0x11 | GET_FW_VERSION | `read_firmware_version` | Returns 8 bytes: [major(2B), minor(2B), revision(2B), patch(2B)] from NVS 0x2800 firmware header |
| 0x12 | GET_SECONDARY_FW_VERSION | `sub_2276C` | Returns secondary firmware version (same format) |
| 0x20 | APPLY_PATCH | `sub_22C4E` | Apply runtime firmware patch. Cell sends patch data, syscon applies it. Used during system software updates. |

---

## SID 64: LS (LabStation) - `serv_ls.c`

Factory/debug LabStation mode interface.

| Cmd | Name | Handler | Description |
|-----|------|---------|-------------|
| 0x00 | LS_INIT | `sub_1CD4A` | Initialize LabStation communication. Stores comm tag. Only accepted when NVS 0x3904 == 0x00 (LabStation mode enabled). |

---

## SID 80: STORAGE

BD/HDD storage subsystem interface.

| Mode | Cmd | Handler | Description |
|------|-----|---------|-------------|
| 0 | 0x20 | inline | READ_STORAGE - returns 2 bytes of storage state |
| 16 | - | `sub_234AC` | Async storage event notification |

---

## Error Logger Task (`sub_1A416`)

Runs as a dedicated task with its own message queue (queue 10). Writes error codes and timestamps to the NVS circular buffer.

### Error Log Layout

```
NVS 0x3700-0x377F: Error codes (32 entries x 4 bytes)
NVS 0x3780-0x37FF: Timestamps (32 entries x 4 bytes, J2000 format)
```

### Write Sequence

1. On startup: scans 0x3700-0x377F to find current write pointer (first entry matching the last-known error code marker)
2. Main loop: blocks on queue 10 waiting for error events
3. On error: reads RTC for timestamp, writes 4-byte error code to `0x3700 + write_idx`, writes 4-byte timestamp to `0x3780 + write_idx`
4. Advances write index by 4, wraps at 0x80 (32 entries)
5. Logs to UART: `[ERROR]: 0x%08x`

---

## BE-SC Communication Architecture

The Cell BE processor does not communicate with the syscon directly. The path is:

```
Cell BE (PPE/SPE)
    |
    | FlexIO / IOIF
    |
Southbridge (CXD2953GA / Starship)
    |
    | SPI bus (4-wire serial)
    |
Syscon (CXR713F / Mullion)
```

The Southbridge acts as a bridge (literally...). The Cell writes to SB MMIO registers, which the SB translates into SPI transactions to the syscon. The syscon's SB interrupt handler (task 7) monitors the SB mailbox for incoming messages.

On the syscon side, **COMMRECV (task 4)** receives raw SPI frames from the Southbridge, parses the BE-SC protocol framing (header, SID, length, payload, checksum), and routes the parsed message to the appropriate service task via RTOS message queues.

**SERV_MISC (task 18)** is the main service dispatcher (`sub_1428C`). It handles 15 service IDs covering hardware configuration, power management, storage, diagnostics, and debug.

Response path is the reverse: the service handler builds a response buffer, sends it back through the SPI bus to the Southbridge, which makes it available to the Cell via MMIO.

### Message Queues

| Queue | Owner | Writers | Purpose |
|-------|-------|---------|---------|
| 4 | SSM | Shell commands, SYSPM, button handlers | SSM command codes (32-bit) |
| 6 | Power semaphore | SYSPM, power functions | Power sequencer mutual exclusion |
| 10 | Error logger | Any task via `errlog_emit_event` | Error code delivery |
| 46 | Bringup | `bringup_plumb_queues` | Service initialization sync |
| 65 | RTOS semaphore 0x41 | HDMI task | HDMI I2C bus mutual exclusion |

### Cell-Initiated Request Flow

```
Cell BE -> SB MMIO write -> SB SPI TX -> Syscon SB IRQ (task 7)
    -> COMMRECV (task 4) parses BE-SC frame
    -> routes to SERV_MISC (task 18) by SID
    -> SID handler processes command
    -> response -> SPI TX -> SB -> Cell MMIO read
```

### Syscon-Initiated Async Push

```
HW event (button press, thermal alarm, livelock, RTC)
    -> ISR or monitor task detects event
    -> msg_post(sid, data, len, seq, comm_tag)
    -> SPI TX -> SB -> Cell interrupt -> Cell handler
```

The `comm_tag` is a connection identifier stored when the Cell first subscribes to a notification source (e.g., LIVELOCK enable, CONTROL_WATCHDOG enable, CONTROL_WAKE set). It routes the async notification back to the correct Cell-side handler.
