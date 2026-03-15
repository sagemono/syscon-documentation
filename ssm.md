# SSM & Power Subsystem Analysis

## Overview

The System State Manager (SSM) is the central power orchestration subsystem inside the PS3 Syscon. It runs as RTOS task 4 and manages all power state transitions for the Cell Broadband Engine (BE), RSX, and supporting hardware. Every power-on, shutdown, restart, thermal kill, and error recovery flows through the SSM's state machine.

This writeup covers the SSM architecture, state machine, shell commands, the `syspowdown` command parameter space, and a newly discovered "zombie" power state caused by `syspowdown 0 1 0`.

**Source**: CXR713120-201GB (Mullion) rev:0B8E, firmware v1.0.0_k1. All addresses are ROM offsets in the Syscon firmware image.

---

## SSM Task Architecture

### Task Layout

The SSM runs in `ssm_task_main` (`0x10FC4`). On startup it:

1. Reads NVS offset 14592 for configuration
2. Creates the power sequencer task (RTOS task 10) and the WMM (thermal monitor) tasks
3. Enters its main loop, receiving 32-bit command codes from RTOS message queue 4

The main loop is:

```
while (1) {
    // Check for deferred command from previous iteration
    if (deferred_cmd != 0) {
        current_cmd = deferred_cmd;
        deferred_cmd = 0;
    } else {
        rtos_recv(queue_4, &current_cmd, WAIT_FOREVER);
    }

    // Phase 1: Dispatch based on top byte of command
    action = sub_10A98(&ssm_ctx);

    // Phase 2: State transition (updates major/sub state, fires callbacks)
    if (action != 100)
        sub_1D4F0(action, &ssm_ctx);

    // Phase 3: Post-transition processing (sends phase to power sequencer)
    if (action != 100)
        sub_10C96(action, &ssm_ctx);
}
```

### SSM Context Structure

The SSM context is a structure at `dword_2009244` (approx 48 bytes):

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0x00 | 4 | `cmd_code` | Current 32-bit command code |
| 0x04 | 4 | `next_cmd` | Queued command for next iteration |
| 0x08 | 4 | `pending_cmd` | Deferred command during bringup |
| 0x0C | 4 | `transition_flag` | Set to 1 during transition, 0 after |
| 0x10 | 1 | `state_major` | Current major state (0-8) |
| 0x11 | 1 | `state_sub` | Current sub-state |
| 0x14 | 4 | `wake_flag` | Wake source flag for bringup callback |
| 0x1C | 4 | `error_flag` | Non-zero when error condition present |
| 0x20 | 4 | `syspm_stat[0]` | Power management status word 0 |
| 0x24 | 4 | `syspm_stat[1]` | Power management status word 1 |
| 0x28 | 4 | `bringup_allowed` | 0 = bringup prevented, 1 = allowed |
| 0x2C | 4 | `force_restart` | Forces restart on next shutdown |

---

## SSM State Machine

### States

The SSM tracks its state as two nibbles: `XXYY` where XX is the major state and YY is the sub-state. This is what appears in UART logs as `[SSM] state: XXYY -> XXYY`.

| Major | Name | Description |
|-------|------|-------------|
| 0 | PowerOff | BE is completely off, standby mode |
| 1 | Bringup | Boot in progress, sub-states 01-06 track phases |
| 2 | Running (early) | BE attention received, post-boot init |
| 3 | Error (Bringup) | Boot failed during bringup (PowSeq Fail) |
| 4 | Running (full) | BE fully running, all subsystems up. Normal operating state. |
| 5 | Shutting Down | Shutdown in progress, power rails being torn down |
| 6 | PowerOff (Fatal) | Fatal error power-off, requires explicit clear |
| 7 | Error Recovery | Transitioning from error to shutdown |
| 8 | Fatal Recovery | Transitioning from fatal error to shutdown |

### Actions

Actions are the inputs that drive state transitions:

| Action | Name | Trigger |
|--------|------|---------|
| 0 | Bringup | Power button press, `bringup` command, restart from shutdown |
| 1 | Shutdown | Software shutdown request, `shutdown` command |
| 2 | Reset | `syspowdown 0 0 1`, warm reset without full power cycle |
| 3 | Fatal | Unrecoverable error (thermal, checkstop, etc.) |
| 4 | Success | Phase completed successfully |
| 5 | Fail | Phase failed |
| 6 | Error | Error during shutdown/recovery |
| 7 | Clear | Clear fatal error state |
| 99 | Forced | Forced shutdown (used in callbacks as wildcard match) |

### State Transition Table

Located at `byte_3FD94`, this is a 9x8 lookup table. Each row is a state, each column is an action. Value 0x0F means "invalid/no transition".

```
             Action:  0(bringup) 1(shutdown) 2(reset) 3(fatal) 4(ok) 5(fail) 6(error) 7(clear)
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

Key transitions to note:

- **Bringup (0->1)**: Cold boot, starts power sequencer phases 1-5
- **Shutdown (4->5)**: Normal shutdown from full running state
- **Reset (4->1)**: Warm reset, skips full power cycle, goes directly back to bringup
- **Fatal (any->7/8)**: Any running state can be killed by fatal errors (thermal, checkstop)
- **Shutdown complete (5->0)**: Power rails torn down, system enters standby
- **Fatal clear (6->0)**: Only via explicit error clear (action 7)

### Bringup Sub-States

During bringup (major state 1), sub-states track which power sequencer phase is active:

```
0000 -> 0101 (phase 1 start)
0101 -> 0201 (phase 1 complete)
0201 -> 0102 (phase 2 start)
0102 -> 0202 (phase 2 complete)
0202 -> 0103 (phase 3 start)
0103 -> 0203 (phase 3 complete)
0203 -> 0104 (phase 4 start)
0104 -> 0204 (phase 4 complete)
0204 -> 0105 (phase 5 start)
0105 -> 0400 (phase 5 complete -> Running)
```

A failure at any phase transitions to state 03XX (Error), for example:
```
0103 -> 0303  (PowSeq Fail at phase 3)
```

---

## Command Code Dispatch

When the SSM receives a 32-bit command code, `sub_10A98` extracts the top byte to determine the action:

| Top Byte | Command | Behavior |
|----------|---------|----------|
| `0x11` | Bringup | Action 0. Only accepted from states 0 (off) or 2 (running-early). If `bringup_allowed` is 0, logs "Bringup is currently prevented" and ignores. |
| `0x12` | Shutdown | Action 1. Accepted from states 2, 3, 4. If currently in state 1 (booting) with bit 8 set and no pending 0x3F command, saves for later execution. Otherwise logs "Do nothing. (X State)". |
| `0x13` | Reset | Action 2. Only from state 4 AND only if RSX is fully powered (sub_6B64 returns 2). If RSX is not fully on, logs "Reset msg ignored because RSX Power state is not all ON." |
| `0x20` | Wake/standby | Internal handler `sub_11086` |
| `0x21` | (same as 0x20) | Same handler |
| `0x3E` | Error recovery | `sub_1085C` |
| `0x3F` | Fatal error | `sub_105E8` |
| `0x40` | Shutdown re-entry | `sub_1104C`. Only valid from state 6 (fatal off). |

---

## SSM Transition Callbacks

After every state transition, the SSM iterates through a callback table at `byte_3FDDC`. Each entry is 8 bytes:

```
byte[0]: from_state (high nibble=major, low nibble=sub, 0xE=wildcard)
byte[1]: to_state   (same encoding)
byte[2]: action     (0x63/99 = match any action)
byte[3]: padding
byte[4-7]: callback function pointer (Thumb)
```

The SSM fires every callback whose from/to/action pattern matches the current transition. 0xE in any nibble acts as a wildcard.

### Key Callbacks

| From | To | Action | Callback | Purpose |
|------|----|--------|----------|---------|
| EE->EE | any | 99 | `0x1D23A` | **Master cleanup**. When transitioning to state 0x00, calls `sub_1EFEC`, `sub_1F040`, `sub_1DE48`, `sub_1DFF0` to clear wake sources, reset button state machines, and clean up power state. |
| 40->50 | - | 1 | `0x1D27E` | Running->Shutdown transition. Sets up shutdown parameters. |
| EE->EE | - | 3 | `0x1D288` | Fatal action. Handles error logging and wake source setup for error-triggered shutdowns. |
| EE->00 | - | 99 | `0x1D2AE` | **PowerOff transition**. If bit 0 of SSM flags is NOT set, calls `sub_1EFA2` to clear wake source registers (`word_2000F64`=0, `word_2000F66`=0). Then stores the shutdown mode byte into `byte_2000F60` via `sub_1F00C`. |
| 00->11 | - | 0 | `0x1D328` | **Bringup from PowerOff**. Checks `wake_flag` at ctx+0x14; if non-zero, calls `sub_1EFCC(2)` to set wake source flag 2. |
| 00->11 | - | 0 | `0x1D052` | **Bringup setup**. Creates all service tasks (WMM, HDMI, thermal, security, BD, etc.) and initializes the power sequencer context. |

---

## Power Service Command Handler

The function `sub_1F700` is the SYSPM (System Power Management) service handler. It processes commands from the Cell/BE processor via the SB communication channel:

| Command ID | Name | Behavior |
|------------|------|----------|
| 0 (0x00) | Shutdown | Logs `[SERV SYSPM] SHUTDOWN CMD`, calls `sub_1F05A(0, 0)` |
| 1 (0x01) | Reset | Logs `[SERV SYSPM] RESET CMD`, calls `sub_1F05A(0, 1)` |
| 16 (0x10) | Get Powup Cause | Returns 16-byte power-up cause data |
| 17 (0x11) | Shutdown Extended | Parses wake_param from payload, calls `sub_1F102` -> `sub_1F05A` |
| 18 (0x12) | Set Wake Source | Calls `sub_1F902` |
| 19 (0x13) | (unknown) | Calls `sub_1F986` |
| 32 (0x20) | Standby | Calls `sub_1F9F6` |
| 35 (0x23) | (unknown) | Calls `sub_1FB64` |
| 40 (0x28) | (same as 0x20) | Calls `sub_1F9F6` |
| 48 (0x30) | (unknown) | Calls `sub_1FBCC` |
| 49 (0x31) | Get Clock Info | Returns clock configuration data |
| 50 (0x32) | (unknown) | Calls `sub_1FC76` |
| 51 (0x33) | (unknown) | Calls `sub_1FC9C` |
| 64 (0x40) | (unknown) | Calls `sub_1FD02` |

Command 17 (0x11) is the extended shutdown used by `syspowdown`. The payload length determines the mode: length=4 means 3 argument bytes follow the command byte, length=1 means simple shutdown.

---

## The `sub_1F05A` Power Dispatch Function

Address: `0x1F05A`

This is the core function that translates power parameters into SSM command codes. It takes:
- `a1`: 16-bit wake parameter (high byte = restart type, low byte = restart flag)
- `a2`: panic flag (from arg3 of syspowdown)
- `a3`, `a4`, `a5`: additional flags from other callers

### SSM Code Construction

The base SSM code is selected from the input flags:

| Condition | SSM Code | Hex |
|-----------|----------|-----|
| `a2` (panic) | 318898176 | `0x13020000` |
| `a3` (flag) | 302120976 | `0x12020010` |
| `a4` (flag) | 302121056 | `0x12020060` |
| `a5` (flag) | 1057095934 | `0x3F0200FE` |
| default | 302120960 | `0x12020000` |

Then the restart bit is applied:
- If `a1 >> 8` (high byte) is 1 or 2: `code |= 1` (restart flag)
- If `a1 & 0xFF` (low byte) is 1: `code |= 1` (restart flag)
- If `a1 >> 8` is any other non-zero value: command rejected silently

Before sending the command, the function:
1. Locks the power sequencer semaphore: `sub_31D1C(6, -1)`
2. Stores the wake parameter: `word_2000F64 = a1`
3. Stores and traces: `sub_1F8EE(a1)` -> writes `word_2000F66 = a1`
4. Releases the semaphore: `sub_3EF0C(6)`
5. Sends the command: `sub_10316(code)` -> enqueues to SSM task queue 4

---

## Shell Commands

### `bestat`

Handler: `sub_CEF4` (`0xCEF4`)

Takes no arguments. Queries the SSM state machine via `sub_1D1F2` and prints the current power state plus any error/status reason.

The function checks each possible state and prints:

| Value | State String |
|-------|-------------|
| 0 | (PowerOn State) |
| 1 | (PowerOff State) |
| 2 | (Busy State) |
| 3 | (Error State) |

Plus the error/status reason (from the sub-state byte):

| Sub | Reason String |
|-----|--------------|
| 0 | (newline only, no error) |
| 1,2,3,9 | (Power Fail) |
| 4,5,6 | (Thermal Alert) |
| 7 | (Trigger IN) |
| 8 | (CheckStop IN) |
| 10 | (RSX Interrupt) |
| 11 | (Attention BE) |
| 12 | (BE PLL Unlock) |
| 13 | (PowSeq Failed) |
| default | (Unknown Error) |

### `powupcause`

Handler: `sub_AE88` (`0xAE88`)

Takes no arguments. Requires BE to be running (`touch_first_rx_seen()` must return true). Calls `sub_1EF3A` to read the 16-byte power-up cause register and dumps it as hex:

```
Powup Cause: XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX
```

This tells you why the console last powered on (power button, bluetooth wake, RTC alarm, etc.).

### `bringup`

Handler: `sub_D07E` (`0xD07E`)

Takes no arguments. Checks whether BE is currently in a running state:
- **If running**: reads bringup mode from NVS, checks if mode 4 (restart) flag is set, clears it, and sends `0x11000000` to the SSM. This handles "restart while already on".
- **If not running**: sends `0x11000000` directly. Cold boot.

### `shutdown`

Handler: `sub_D0B0` (`0xD0B0`)

Takes no arguments. Checks a shutdown-in-progress flag via `sub_1D0F0`:
- **If already transitioning**: sends `0x40000000` (re-entry shutdown)
- **If in normal running state**: sends `0x12010000` (standard shutdown)

### `powersw`

Handler: `cmd_powersw` (`0xD0E4`)

Takes no arguments. Simulates a physical power button press. Calls `sub_1DE00(0x64)` which enters the power button state machine with a 100-unit debounce. The button handler (`sub_1DC3E`) is complex:

- Checks for double-press bugs (button already in pressed state)
- If BE is in bringup mode 1: builds a special command packet `{0x22, hi, lo}` and sends via `sub_1F1F4`
- If BE is in mode 0xFF (default): calls `sub_1DAE4` for standard power toggle behavior

After the button logic, falls through to common exit code at `0xD0AA`.

### `resetsw`

Handler: `sub_D0F0` (`0xD0F0`)

Simulates a physical reset button press. Calls `sub_1DF74(100)`.

### `ejectsw`

Handler: `sub_D0FC` (`0xD0FC`)

Simulates a physical eject button press. Calls `sub_1E0FE(1)`.

### `powerstate`

Handler: `sub_C582` (`0xC582`)

Takes no arguments. Dumps the current on/off status of every power domain:

```
ATA Power          : ON/OFF
PCI Power          : ON/OFF/AUX ON
RSX Power          : ON/OFF/GDDR ON
XDR Power          : ON/OFF
Eurus Power        : ON/OFF
SB Power           : ON/OFF
RSX Thermal Sensor : AVAILABLE/UNAVAILABLE
BE Thermal Sensor  : AVAILABLE/UNAVAILABLE
```

### `devpm`

Handler: `sub_C75E` (`0xC75E`)

Device Power Management. Directly toggles individual power rails:

```
devpm pci off/on         - PCI bus power
devpm ata off/on         - ATA (hard drive) power
devpm rsx off/on/gddron  - RSX core off, RSX core on, or core off but GDDR stays on
```

These bypass the SSM entirely and directly call power rail control functions. Useful for debugging but can leave the system in inconsistent states.

### `becount`

Handler: `sub_C20E` (`0xC20E`)

Reads power cycle statistics from NVS:
- NVS offset 13824 (2 bytes): bringup count
- NVS offset 13826 (2 bytes): shutdown count
- NVS offset 13828 (4 bytes): total power-on time in seconds

Output:
```
Bringup : X times
Shutdown: X times
Power-on: Xday XXhour XXmin XXsec
```

### `tshutdown`

Handler: `sub_AAEA` (`0xAAEA`)

Thermal shutdown temperature configuration. Subcommands:
- `tshutdown get <zone>` - read current threshold for thermal zone
- `tshutdown set <zone> <value>` - set threshold for thermal zone
- `tshutdown setini <zone>` - read factory default threshold
- `tshutdown getini <zone>` - get factory default from NVS

Zones with configurable thresholds: 0 (BE), 1, 3, 20, 21. Thresholds are stored in NVS at offsets 0x3492-0x34AA.

---

## `syspowdown` Command In Depth

Handler: `sub_AED0` (`0xAED0`)

### Synopsis

```
syspowdown <arg1> <arg2> <arg3>       (3 hex bytes, max 2 chars each)
syspowdown <wake_hi> <wake_lo>        (2 x 8 hex char words)
```

Requires BE to be running. If BE is off, prints: `The command is valid only when BE is running.`

### Form 1: Three-Argument Mode

Builds a 4-byte command packet: `{0x11, arg1, arg2, arg3}` and dispatches through `sub_1F102` -> `sub_1F05A`.

The wake parameter is constructed as: `wake_param = (arg1 << 8) | arg2`

The SSM command code construction:

| Condition | SSM Code | Effect |
|-----------|----------|--------|
| Base | `0x12020000` | Shutdown |
| arg3 == 1 | `0x13020000` | Panic/Reset |
| arg1 == 1 or 2 | `code |= 1` | Restart bit set |
| arg2 == 1 | `code |= 1` | Restart bit set |

The wake parameter is stored in `word_2000F64` before the command is sent.

### Form 2: Two-Argument Mode (Custom Wake Source)

Takes two 8-character hex arguments representing 32-bit wake source words:

```
syspowdown XXXXXXXX YYYYYYYY
```

This sets custom wake source registers:
- `dword_2000C64` = first argument (wake source high)
- `dword_2000C68` = second argument (wake source low)
- `dword_2000C60` = 1 (flag: use custom wake source)

Then dispatches a standard `{0x11, 0, 0, 0}` shutdown. When the SSM processes the shutdown (case 6 in `sub_10B72`), it checks `dword_2000C60`. If set, it writes the custom values directly into `syspm_stat[0]` and `syspm_stat[1]` instead of computing them from the normal wake source logic. This is the mechanism behind "Set Wake Up Source to Restart after Shutdown" from the SC communication protocol.

### Parameter Space (Verified on Hardware)

**Important: None of these commands trigger an automatic restart.** All of them shut down (or reset) and then require a manual `powersw` command or physical power button press to boot again. The `syspm_stat` value persists in NVS through the shutdown and into the next bringup, where it is communicated to the bootloader to tell XDR memory to retrain.

| Command | Packet | Wake Param | SSM Code | ctxt | syspm_stat | Behavior |
|---------|--------|------------|----------|------|------------|----------|
| `syspowdown 0 0 0` | `11 00 00 00` | `0x0000` | `0x12020000` | 00/00 | 00000000/00000000 | **Clean shutdown.** All power rails torn down. LED goes red. System enters standby. Next boot works normally. |
| `syspowdown 1 0 0` | `11 01 00 00` | `0x0100` | `0x12020001` | 01/00 | 00000001/00000000 | **Poisoned shutdown.** All rails torn down, syspm_stat=1 persists. Next boot results in GLOD (XDR link failure). See below. |
| `syspowdown 2 0 0` | `11 02 00 00` | `0x0200` | `0x12020001` | 02/00 | 00000000/00000000 | **Clean shutdown.** syspm_stat=0 because ctxt_byte1=2 fails the `== 1` check. Next boot works normally. |
| `syspowdown 0 0 1` | `11 00 00 01` | `0x0000` | `0x13020000` | - | - | **Warm reset.** State goes 0400->0106->0201->...->0400. No shutdown, no power cycle. BE re-bringup from phase 1 while power rails stay up. Immediate. |
| `syspowdown 0 1 0` | `11 00 01 00` | `0x0001` | `0x12020001` | 00/01 | 00000002/00000000 | **ZOMBIE STATE.** Incomplete shutdown, RSX stays warm. See below. |

### syspm_stat Computation

The syspm_stat value is computed in `sub_10B72` (case 6) during shutdown, from the wake parameter stored in `word_2000F64`:

```c
ctxt_byte1 = HIBYTE(word_2000F64);  // arg1 from syspowdown
ctxt_byte2 = LOBYTE(word_2000F64);  // arg2 from syspowdown

v17 = (ctxt_byte1 == 1);   // EXACT match to 1 only
v16 = (ctxt_byte2 == 1);   // EXACT match to 1 only

syspm_stat[0] = (v16 ? 2 : 0) | (v17 ? 1 : 0);  // plus other bits from wake sources
```

This `== 1` check is why `syspowdown 2 0 0` produces syspm_stat=0 (ctxt_byte1=2 does not match) while `syspowdown 1 0 0` produces syspm_stat=1 (ctxt_byte1=1 matches).

---

## The GLOD State: `syspowdown 1 0 0`

### Symptoms

When `syspowdown 1 0 0` is issued from UART:

1. UART shows normal shutdown: state 0400 -> 0500 -> 0000 (PowerOff State)
2. LED goes red, system is fully off
3. All power rails torn down normally (syspm_stat bit 0 does not affect the shutdown handler's rail teardown path)
4. On next boot (via `powersw` or power button), syspm_stat=00000001 persists
5. Cell Boot Loader SE reports: `[ERROR]: 0xb0002001 (FATAL) XDR Link not initilized.`
6. Bootloader dumps diagnostic data: ITC_DUMP, PTC_DUMP, MIC_DUMP, XIO_DUMP, XDR_DUMP, IOW_DUMP (all from Cell, not syscon)
7. System enters GLOD: green LED on, no video output, unresponsive to power button
8. Must issue `shutdown` via UART shell to recover

### Root Cause Analysis

**Step 1: Shutdown is clean, but syspm_stat=1 poisons the next boot**

The shutdown itself completes normally. The syspm_stat=1 flag does NOT affect the syscon shutdown handler's rail teardown path (that's controlled by bit 1, not bit 0). All power rails are properly torn down.

However, syspm_stat=1 persists in RAM and is communicated to the Cell Boot Loader SE on the next bringup via `[SSM] Bringup mode : syspm_stat=00000001/00000000`.

**Step 2: Cell bootloader attempts warm XDR initialization**

The Cell Boot Loader SE reads the syspm_stat from syscon to determine its initialization strategy:

- `syspm_stat=0`: Cold boot. Full XDR memory training from scratch.
- `syspm_stat=1`: Warm/soft restart. Attempt to use previously calibrated XDR timing data, skip full training.

When syspm_stat=1, the bootloader assumes XDR memory controllers are in a warm state with valid calibration. But since the shutdown was actually a full cold shutdown (all rails dropped), the XDR hardware has no valid state. The XDR link initialization fails.

**Step 3: Bootloader dumps diagnostic data and hangs**

The bootloader reports `0xb0002001 (FATAL) XDR Link not initilized` and dumps all memory controller calibration data:

| Dump | Source | Content |
|------|--------|---------|
| ITC_DUMP | Cell Boot Loader SE | XDR I/O Timing Controller initial calibration (all zeros = never trained) |
| PTC_DUMP | Cell Boot Loader SE | XDR Pad Timing Controller(?) calibration (all zeros) |
| MIC_DUMP | Cell Boot Loader SE | Memory Interface Controller state (partial data from hardware registers) |
| XIO_DUMP | Cell Boot Loader SE | XDR I/O configuration per channel (two blocks for CH0/CH1) |
| XDR_DUMP | Cell Boot Loader SE | XDR channel training results |
| IOW_DUMP | Cell Boot Loader SE | I/O Window calibration timing per lane |

These dumps are NOT from syscon firmware. They are from the Cell processor's bootloader and are relayed to UART via the SB debug interface. The ITC and PTC dumps being all zeros confirms that XDR training never completed.

**Step 4: System hangs in GLOD**

After the fatal XDR error, the Cell bootloader halts. Syscon still considers the system to be in state 0400 (PowerOn). The green LED stays on, but there is no video output and the system does not respond to the physical power button. Recovery requires issuing `shutdown` via the UART shell.

### Why This is a Shell-Only Bug

In normal PS3 operation, `syspowdown 1 0 0` is never issued by the shell. The SC communication protocol equivalent (command 0x11 with payload byte 0x01 = Soft Restart) is always initiated by Cell/BE software, which performs proper XDR state preservation before requesting the restart. The sequence should be:

1. Cell software saves XDR calibration data to NVS
2. Cell software sends shutdown command 0x11/0x01 to syscon via SB
3. Syscon shuts down with syspm_stat=1
4. Syscon immediately re-bringups (the Cell side triggers auto-restart through the SC protocol, not through the shell command)
5. Cell bootloader sees syspm_stat=1, loads saved XDR calibration from NVS, performs warm init
6. XDR link comes up using saved calibration. Fast boot.

When issued from the syscon shell, steps 1 and 4 are skipped. The XDR calibration is never saved, and no auto-restart is triggered.

---

## The Zombie State: `syspowdown 0 1 0`

### Symptoms

When `syspowdown 0 1 0` is issued from UART:

1. UART shows shutdown sequence: state 0400 -> 0500 -> 0000
2. LED goes red (standby)
3. Fan continues running
4. RSX continues drawing ~1.5W (visible on power monitoring)
5. CELL drops to 0W (correctly powered off)
6. System reports "(PowerOff State)" but is NOT fully off
7. Pressing the power button from this state causes YLOD (PowSeq Fail at phase 3, state 0103 -> 0303)

![Image](https://github.com/sagemono/syscon-documentation/blob/main/images/powerchart.png?raw=true)

### Root Cause Analysis

**Step 1: Wake Parameter Sets syspm_stat Bit 1**

For `syspowdown 0 1 0`, the wake parameter is `0x0001` (arg1=0, arg2=1). During shutdown processing:

```c
ctxt_byte1 = HIBYTE(0x0001) = 0x00;
ctxt_byte2 = LOBYTE(0x0001) = 0x01;

v17 = (ctxt_byte1 == 1);  // false
v16 = (ctxt_byte2 == 1);  // TRUE

syspm_stat[0] = (v16 ? 2 : 0) | (v17 ? 1 : 0) = 0x00000002;
```

**Step 2: Shutdown Handler Skips RSX Power Teardown**

The main shutdown function at `sub_77E2` receives a context structure whose flags word (`ctx[3]`) determines which power domains to tear down. When `ctx[3]` has bit 1 set (derived from syspm_stat=2), the function takes an alternate "warm" path:

- Calls `sub_2D582(0)` instead of normal XDR shutdown
- Skips `sub_2E7AE` (voltage regulator shutdown)
- Calls `sub_2D674`/`sub_2D666` (partial RSX shutdown) instead of full teardown
- Skips `sub_2DC12`, `sub_2E4C8` (SB bus shutdown)
- Skips `sub_2E152` (final power rail disable)
- Calls `sub_2DA78`/`sub_2E038` instead of `sub_2DC02`/`sub_2DFE6`

This leaves RSX in a partially-powered state where the core is stopped but VDDIO/GDDR rails remain energized, explaining the ~1.5W residual draw and the fan continuing to run.

**Step 3: YLOD on Next Boot**

When the user presses the power button to boot from this zombie state:

1. SSM transitions 0000 -> 0101 (bringup starts)
2. The stale `syspm_stat=00000002` persists and is printed: `[SSM] Bringup mode : syspm_stat=00000002/00000000`
3. At phase 3 (clock/PLL configuration), the boot sequence attempts to configure clocks on an RSX that has been sitting in an intermediate power state
4. PLL or attention handshake fails: state transitions 0103 -> 0303 (PowSeq Fail)
5. Error recovery: 0303 -> 0700 -> 0600 (Fatal PowerOff with YLOD)

### Power Consumption Graph

From real hardware measurement during `syspowdown 0 0 0`:

```
Time 0s:    System power on. CELL ramps to ~65W, RSX to ~30W, Total ~95W.
Time ~25s:  In XMB. CELL ~65W, RSX ~30W stable.
Time ~28s:  syspowdown 0 0 0 issued.
            CELL drops to 0W over ~1s.
            RSX drops to ~1.5W, then to 0W after ~2s more.
            Total reaches 0W (standby).
            RSX at 1.5W forever = GDDR/VDDIO rail energized.
```

In the zombie state (`syspowdown 0 1 0`), the RSX never drops below ~1.5W because the GDDR/VDDIO rails are intentionally kept energized by the partial shutdown path.

### Comparison: All `syspowdown` Modes (Verified on Hardware)

| Mode | State Flow | syspm_stat | Rails After Shutdown | Next Boot Result |
|------|-----------|------------|---------------------|-----------------|
| `0 0 0` | 0400->0500->0000 | 00000000 | All off | **OK** - Clean cold boot |
| `1 0 0` | 0400->0500->0000 | 00000001 | All off | **GLOD** - XDR link failure. Bootloader thinks warm boot but XDR is cold. |
| `2 0 0` | 0400->0500->0000 | 00000000 | All off | **OK** - Clean cold boot (ctxt_byte1=2 fails ==1 check) |
| `0 0 1` | 0400->0106->...->0400 | (no shutdown) | Never down | **OK** - Immediate warm reset, rails stay up |
| `0 1 0` | 0400->0500->0000 (zombie) | 00000002 | RSX warm (~1.5W) | **YLOD** - RSX in bad state, PowSeq Fail at phase 3 |

### Why `0 1 0` is Different From `1 0 0`

Both are broken when used from the shell, but in different ways:

- `1 0 0` sets syspm_stat bit 0 (=1). This does NOT affect the syscon shutdown handler's rail teardown. All rails go down cleanly. The damage is done on the NEXT boot: the Cell bootloader sees syspm_stat=1 and attempts warm XDR init on cold hardware. Result: GLOD.

- `0 1 0` sets syspm_stat bit 1 (=2). This DOES affect the syscon shutdown handler. The shutdown takes the warm path, leaving RSX/GDDR rails energized. The system enters a zombie state where it's "off" but RSX is still drawing power. On next boot, RSX is in an undefined intermediate state. Result: YLOD.

### Why `2 0 0` Works But `1 0 0` Doesn't

The syspm_stat computation uses exact equality checks:

```c
v17 = (ctxt_byte1 == 1);  // only matches 1, not 2
```

- `syspowdown 1 0 0`: ctxt_byte1 = 0x01, matches `== 1`, sets syspm_stat bit 0. Broken.
- `syspowdown 2 0 0`: ctxt_byte1 = 0x02, does NOT match `== 1`, syspm_stat stays 0. Works fine.

This is likely intentional from the firmware perspective: value 1 means "soft restart with warm state", value 2 means "hard restart with full reinit". But without the Cell-side coordination (saving XDR calibration data before shutdown and triggering auto-restart after), the soft restart path fails.

---

## Wake Source Registers

The Syscon maintains several registers for tracking wake/restart state:

| Address | Name | Purpose |
|---------|------|---------|
| `byte_2000F60` | shutdown_mode | Shutdown mode byte (set by PowerOff callback) |
| `word_2000F64` | wake_param | 16-bit wake parameter from `sub_1F05A`. Hi byte = restart type, lo byte = restart flag. |
| `word_2000F66` | wake_param_copy | Copy of wake_param (set by `sub_1F8EE`) |
| `word_2000F6A[1]` | wake_src_flags | Bitfield of wake source flags (OR'd in by `sub_1EFCC`) |
| `dword_2000C60` | custom_wake_flag | When 1, use custom wake source values |
| `dword_2000C64` | custom_syspm_0 | Custom syspm_stat[0] value |
| `dword_2000C68` | custom_syspm_1 | Custom syspm_stat[1] value |

The `syspm_stat` values are stored in the SSM context at offsets 0x20 and 0x24 (memory addresses `MEMORY[0x2006804][2697]` and `[2698]`). These persist across the shutdown and are printed in UART logs:

```
[SSM] Bringup mode : syspm_stat=XXXXXXXX/XXXXXXXX
[SSM] Shutdown mode : syspm_stat=XXXXXXXX/XXXXXXXX
```

### syspm_stat Bit Meanings

| Bit | Value | Meaning | Set When | Syscon Effect | Cell Bootloader Effect |
|-----|-------|---------|----------|--------------|----------------------|
| 0 | 0x01 | Soft restart | ctxt_byte1 == 1 | None on shutdown | Attempts warm XDR init (fails if XDR was cold-shutdown) |
| 1 | 0x02 | Keep-warm flag | ctxt_byte2 == 1 | Shutdown skips RSX/XDR rail teardown | Attempts warm XDR init (fails because RSX in intermediate state) |
| 2 | 0x04 | Wake source pending | Various wake conditions | (varies) | (varies) |
| 3 | 0x08 | Restart from error | Error-triggered restart | (varies) | (varies) |

---

## Shutdown Handler: `sub_77E2`

Address: `0x77E2`

This is the main power rail teardown function called during powseq phase 6. It takes the power sequencer context and tears down power domains in a specific order based on the flags in `ctx[3]` and `ctx[4]`.

### Shutdown Sequence (Normal Path, `ctx[3] & 2 == 0`)

1. Save SB interrupt masks via `sub_189C8` on SB registers 772/776/780/784
2. Mask interrupts by ANDing with `0xAFFFFFFF` via `sub_189F8`
3. Disable SB register 28736
4. Acquire FlexIO and RSX SPI bus locks
5. Shut down SB context and power monitor buses
6. Disable power gate 4 via `sub_2E646(4)`
7. Shutdown voltage regulators: `sub_2E7AE`, `sub_2E704`, `sub_2E70C`
8. Disable power monitor bus
9. Main power rail disable: `sub_2DE48`
10. Shutdown DC/DC converters: `sub_2DDC6`, `sub_2DC16`
11. Shutdown HDMI/AV: `sub_2E8D8`
12. Disable XDR power: `sub_2E1A6`, `sub_2E1BE`, `sub_2E1B6`, `sub_2E1C6`
13. Additional shutdown: `sub_2E7AA`, `sub_2E17A`, `sub_2E7A2`
14. SB bus shutdown: `sub_2DC12`, `sub_2E4C8`
15. Final rail disable: `sub_2E152`
16. BE/CELL power down: `sub_2E0FC`
17. RSX power down: `sub_2E79A`, `sub_2E0F8`, `sub_2E14E`, `sub_2DB20`
18. Core voltage disable: `sub_2DBE4`, `sub_2E0E0`, `sub_2E792`
19. Final power gate: `sub_2E080`
20. Master power off: `sub_2DC02`, `sub_2DFE6`

### Shutdown Sequence (Warm Path, `ctx[3] & 2 != 0`)

When bit 1 is set, the function takes alternate paths at several decision points:

- Step 3: Calls `sub_2D582(0)` instead of normal bus shutdown
- Steps 7-8: Calls `sub_2D674` + `sub_2D666` instead of `sub_2E7AE`
- Steps 14-15: Skips SB bus and final rail shutdown entirely
- Step 20: Calls `sub_2DA78` + `sub_2E038` instead of `sub_2DC02` + `sub_2DFE6`

This leaves RSX VDDIO and GDDR rails energized for fast restart.

### Flags in `ctx[3]` and `ctx[4]`

| Bit in ctx[3] | Meaning |
|---------------|---------|
| 0 (& 1) | Warm restart mode (affects some rail paths) |
| 1 (& 2) | Keep RSX warm / partial shutdown |
| 2 (& 4) | Skip some late-stage teardown |
| 3 (& 8) | SB warm mode (preserve SB context, skip interrupt teardown) |

| Bit in ctx[4] | Meaning |
|---------------|---------|
| 0 (& 1) | Skip XDR power stage 3 |
| 1 (& 2) | Skip XDR power stage 1 |
| 2 (& 4) | Skip XDR power stage 4 |
| 3 (& 8) | Skip XDR power stage 2 |
| 4 (& 16) | Skip final power rail in warm mode |

---

---

## Appendix: Complete Shell Command Table (Power-Related Subset)

| Address | Command | Handler | Flags | Category |
|---------|---------|---------|-------|----------|
| 0x3F5C4 | `poll` | `cmd_poll` | 0xCDD | SB communication |
| 0x3F5D0 | `recv` | `cmd_recv` | 0xCDD | SB communication |
| 0x3F5DC | `send` | `cmd_send` | 0xCDD | SB communication |
| 0x3F57C | `powerstate` | `sub_C582` | 0xCDD | Power status |
| 0x3F588 | `devpm` | `sub_C75E` | 0xCDD | Device power mgmt |
| 0x3F558 | `becount` | `sub_C20E` | 0xCDD | Power statistics |
| 0x3F60C | `bestat` | `sub_CEF4` | 0xFFD | SSM state query |
| 0x3F618 | `bringup` | `sub_D07E` | 0xFFD | Power on BE |
| 0x3F624 | `shutdown` | `sub_D0B0` | 0xFFD | Power off BE |
| 0x3F630 | `powersw` | `cmd_powersw` | 0xFFD | Simulate power button |
| 0x3F63C | `resetsw` | `sub_D0F0` | 0xFFC | Simulate reset button |
| 0x3F648 | `ejectsw` | `sub_D0FC` | 0xFFD | Simulate eject button |
| 0x3F4D4 | `powupcause` | `sub_AE88` | 0xCDD | Read power-up cause |
| 0x3F4E0 | `syspowdown` | `sub_AED0` | 0xCDD | Extended power control |
| 0x3F3FC | `tshutdown` | `sub_AAEA` | 0xCDD | Thermal shutdown config |
| 0x3F51C | `tshutdowntime` | `sub_C0E8` | 0xCDD | Thermal shutdown timing |
| 0x3F54C | `thermfatalmode` | `sub_C1CC` | 0xCDD | Thermal fatal mode |

Note: Flags field `0xFFD` vs `0xCDD` affects command availability in different authentication states. Commands with `0xFFD` flags (bestat, bringup, shutdown, powersw, ejectsw) are available at a higher privilege level than those with `0xCDD`.
