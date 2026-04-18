# Syscon Boot Sequence

Target: Mullion v1.0.0_k1 firmware

Derived from: all 27 calls to `set_boot_step` (0x6BB8) and `set_boot_step_raw_bcd` (0x1A0E8)

## Task Dispatcher

The power sequence runs as an RTOS task (`powseq_task_main` at 0x7EF6) that blocks on a message queue and dispatches to phase handlers based on the received command byte.

```
powseq_task_main (0x7EF6) - infinite loop
  while (1) {
      cmd = queue_receive(msg_queue);
      switch (cmd) {
          case 1: powseq_phase1_cold_boot       (0x779A)  // Full cold boot
          case 2: powseq_phase2_attention       (0x71F6)  // CELL attention handling
          case 3: powseq_phase3_clock_cfgring   (0x7218)  // Clock stepping + config ring
          case 4: powseq_phase4_training        (0x7236)  // FlexIO training sequence
          case 5: powseq_phase5_flash_handoff   (0x7254)  // Flash firmware + system running
          case 6: powseq_phase6_shutdown        (0x7D70)  // Power-off sequence
          case 7: powseq_phase7_warm_boot       (0x7EBE)  // Resume from sleep
      }
  }
```

Each phase wrapper sets a phase ID at byte offset +1 of the boot context structure before calling the actual work function. The SSM (System State Manager) posts commands to this queue to advance the boot.

## Cold Boot Flow (Phases 1 through 5)

### Phase 1: Cold Boot Entry

**Dispatcher**: `powseq_phase1_cold_boot` (0x779A)
**SSM command**: 1

```
powseq_phase1_cold_boot (0x779A)
  |
  +-- powseq_setup (0x7758)           // one-time HW init, "[POWSEQ] PowerSeq_Setup called."
  |     +-- sub_2CFA0                 // clock gen early init
  |     +-- sub_2CFA4                 // board-specific config
  |     +-- sub_2D09C                 // power domain setup
  |     +-- sub_2D094                 // misc HW init
  |     +-- set_boot_step_raw_bcd(0)  // clear step display
  |
  +-- sub_2DB30()                     // pre-boot hardware check
  |     fail => errlog 0x3000
  |
  +-- boot_steps_0_to_20 (0x74A4)     // main power-on sequence
  +-- boot_final_handoff (0x874C)     // hand off to OS boot
```

### Phase 1 Detail: boot_steps_0_to_20 (0x74A4)

This is the core power-on sequencer.

```
boot_load_board_config (0x870C)        // read NVS 0x3901 board flags
  bit 7 -> stru_2000BF0.magic_or_flags  (board feature A)
  bit 6 -> stru_2000BF0.entry_arg       (board feature B)
  bit 5 -> stru_2000BF0.entry_func      (board feature C)

STEP 0   set_boot_step(ctx, 0)
         power_12v_check()             // check 12V main rail
         fail => errlog 0x3001

STEP 1   set_boot_step(ctx, 1)
         Two paths based on (board_flags & 8):
           flag clear: sub_2BBB6(sb_context_slot)     // init SB SPI bus
                       sub_2BBB6(dword_200C9F4)       // init secondary SB
           flag set:   sub_2CE84()                    // alternate power path

         sub_2E7AE() -> sub_2E704() -> sub_2E70C()    // voltage regulator setup
         sub_2BBB6(cell_rsx_flexio)                   // init CELL/RSX SPI bus
         sub_2BBB6(dword_200C9AC)                     // init CELL secondary

         (board_flags & 8) set path:
           sub_2DC20()                                // DC-DC converter config
           sub_2DDC6() -> sub_2E8D8(0)                // power sequencing
           sub_2DE62() or sub_2DE48()                 // rail enable (depends on bit 0)
           sub_2E646(1)                               // power gate control
           sub_2E750()                                // voltage check
           sub_2BBB6(dword_200C9F8)                   // init power monitor bus
           sub_2DEFA() -> sub_2DEF2()                 // power rail verification

STEP 2   set_boot_step(ctx, 2)
         delay_ms(20)
         sub_2DC3A()                                  // SB half-speed transition
         sub_2DF26()                                  // post-transition check

STEP 3   set_boot_step(ctx, 3)
         boot_step_4_power_rail_enable (0x87C2)       // power rail enable sub-phase

STEP 4   set_boot_step(ctx, 4)     [inside boot_step_4_power_rail_enable]
         sub_2DF44()                                  // main power rail enable
         delay_ms(20)
         sub_2E76C()                                  // power rail status check
         sub_2DF5C()                                  // secondary power enable
         sub_2DF74()                                  // tertiary power (if board feature A)

         sub_2E75E()                                  // power stabilization check

STEP 5   set_boot_step(ctx, 5)
         sub_2DC8E() or sub_2DC30()                   // DC-DC converter fine-tune
         sub_2DF0E()                                  // power rail monitor

         delay_ms(20), delay_ms(11)
         sub_2E646(3)                                 // SB bus enable for power
         sub_2BB7A(dword_200C9F4)                     // release SB bus
         delay_ms(1)
         sub_2BB7A(sb_context_slot)                   // release SB primary
         sub_2E668()                                  // SB power validation
         sub_2DF8C() or sub_2DFFA()                   // VRM config (depends on bit 1)
         sub_2DBF0()                                  // final power validation

STEP 6   set_boot_step(ctx, 6)
         sub_2E06A()                                  // CELL power domain enable
         delay_ms(20)
         sub_2DE70()                                  // optional extra rail (if bit 0)

STEP 7   set_boot_step(ctx, 7)
         boot_steps_7_9_power_validation (0x7272)     // deep power validation

STEP 8   set_boot_step(ctx, 8)     [inside boot_steps_7_9_power_validation]
         sub_2E092() -> sub_2E78E()                   // power check
         delay_ms(20)
         sub_2DBD2()                                  // power stabilization
         sub_2E0EC() -> sub_2E10A() -> sub_2E796()    // multi-stage power verify
         delay_ms(20)
         sub_2E4AE()                                  // VRM final setup
         sub_2DC0E(0)                                 // DC-DC enable

STEP 9   set_boot_step(ctx, 9)     [inside boot_steps_7_9_power_validation]
         sub_2E8CC() -> sub_2E176()                   // interrupt setup
         sub_2E1B2(), sub_2E1C2()                     // conditional rail enables (features B, C)
         sub_2E79E()                                  // power monitor enable
         delay_ms(5)
         sub_2E1CA(), sub_2E19A()                     // phase 2 rail enables (feature B)
         sub_2E1BA()                                  // phase 2 rail enable (feature C)
         delay_ms(5)
         sub_2E1DA(), sub_2E1FA()                     // verify rail stability
         delay_ms(10)
         sub_2E7A6()                                  // final power check
         sub_2E202() -> sub_2BB7A(dword_200C9F8)      // release power monitor bus
         sub_2DB7C()                                  // CELL power-on trigger
           returns 0xB100000F => errlog 0x3003
           returns 0xB1000010 => errlog 0x3004
           returns 0xB1000002 => errlog 0x3002
           returns 0           => success, continue

         sub_2DC9E()                                  // XCG PLL configuration
         sub_2DDDE() -> sub_2DDF6()                   // clock generator final config

STEP 10  set_boot_step(ctx, 10)    [back in boot_steps_0_to_20]
         sub_2DE16() -> sub_2DE26() -> sub_2DE38(0)   // VID setup sequence
         delay_ms(4)
         sub_2E7BE() -> sub_2E708(0) -> sub_2E72E(0)  // power rail final check
         delay_ms(5)
         sub_2D666(), sub_2D674()                     // optional (if bit 1): FlexIO prep

         sub_2BB7A(dword_200C9AC)                     // release CELL secondary bus
         delay_ms(1)
         Psbd_SetBePll() (sub_2CEA4)                  // PLL config to CELL (NVS 0x3958)
         sub_2BB7A(cell_rsx_flexio)                   // release FlexIO bus

STEP 20  set_boot_step(ctx, 20)
         Phase 1 complete. SSM advances to Phase 2.
```

### Phase 2: CELL Attention Handling

**Dispatcher**: `powseq_phase2_attention` (0x71F6)
**SSM command**: 2
**Work function**: `boot_steps_21_22_attention` (0x6E0E)

```
STEP 21  set_boot_step(ctx, 21)
         wait_cell_attention()                        // block until CELL signals attention
         fail => errlog 0x3010, "[POWERSEQ] Error : wait attention timeout.(SEQ1)"

         flexio_link_status_check()                   // check FlexIO bits 16/17
         fail => errlog 0x3013

         sub_2AD00(flexio, 0, &status)                // read CELL status register
         check status bits [10:11]:
           != 2 => errlog 0x3011                      // unexpected attention type
           == 2 => config ring requested, continue

STEP 22  set_boot_step(ctx, 22)
         sub_2B3A8()                                  // config ring write to CELL
         sub_2D8D6(0)                                 // config ring verify
         fail => errlog 0x3012
         delay_ms(20)

         Phase 2 complete. SSM advances to Phase 3.
```

### Phase 3: Clock Stepping + Config Ring

**Dispatcher**: `powseq_phase3_clock_cfgring` (0x7218)
**SSM command**: 3
**Work function**: `boot_steps_23_30_clock_cfgring` (0x8760)

```
STEP 23  set_boot_step(ctx, 23)
         sub_2D950(&v)                                // read current clock config
         sub_2DAAC()                                  // prepare clock stepping table
         sub_2DA92(v)                                 // step clock frequency gradually
                                                      //   uses byte_411E6 lookup (62 entries)
                                                      //   1ms delay between each step
         delay_ms(20)
         sub_2E628()                                  // verify clock stability
         fail => errlog 0x3020

         sub_2D3AE(byte_20072D8)                      // build config ring data (384 bytes)
         sub_2AE2E(flexio, byte_20072D8)              // send config ring to CELL via SPI

STEP 30  set_boot_step(ctx, 30)
         Phase 3 complete. SSM advances to Phase 4.
```

### Phase 4: FlexIO Training

**Dispatcher**: `powseq_phase4_training` (0x7236)
**SSM command**: 4
**Work function**: `boot_steps_31_60_training` (0x6EBC)

```
         wait_cell_attention()                        // wait for CELL ready signal
         fail => errlog 0x3030

STEP 31  set_boot_step(ctx, 31)
         sub_2AD00(flexio, 0, &status)                // read CELL config register
         check bits [21:20] == 1                      // CELL SPI mode validation
         fail => errlog 0x3031
         check bits [3:2] == 3                        // CELL SPI speed validation
         fail => errlog 0x3032

STEP 32  set_boot_step(ctx, 32)
         poll_rsx_vid_ready (0x6E94)                  // poll bit 23 up to 1000x
         fail => errlog 0x3033

         sub_2E4E2(0)                                 // RSX VID enable

STEP 40  set_boot_step(ctx, 40)
         sub_2E5EE()                                  // BitTraining (FlexIO calibration)
         fail => errlog 0x3034

         sub_2D582(1) -> sub_2CE96()                  // post-training checks

STEP 50  set_boot_step(ctx, 50)
         sub_2D55C()                                  // check if byte training needed
         if needed:
           sub_6BE8()                                 // BE-RS byte training
             polling loop (256x): read 0xF610, check >= 0xE6000000
             then wait for bit 27 in completion flag (256x)
             loop timeout => errlog 0x3035
             flag timeout => errlog 0x3036
             "[POWERSEQ] Error : BE-RS byte training failed."

           spi_write_register_64bit(flexio, 0xE200, 0x80000000)
           sub_2E490() -> sub_2D5B2(0)                // post-training setup
           sub_2D50A(0)                               // verify training results
           spi_write_register_64bit(flexio, 0xE200, 0xC0000000)
           read RSX VID response
           response == 0 => errlog 0x3037             // RSX VID setup failure

STEP 51  set_boot_step(ctx, 51)
         sub_6D3E()                                   // BE-SB byte training
         fail => errlog 0x3038, "[POWERSEQ] Error : BE-SB byte training failed."

         spi_write_register_64bit(flexio, 0xF200, 0x80000000)
         sub_2E448() -> sub_2E45A()                   // SB training finalize

         (board_flags & 8) set:
           sub_189F8(sb, 0x304, ctx[5])               // write SB config word 1
           sub_189F8(sb, 0x308, ctx[6])               // write SB config word 2
           sub_189F8(sb, 0x30C, ctx[7])               // write SB config word 3
           sub_189F8(sb, 0x310, ctx[8])               // write SB config word 4
         else:
           sub_2E2F2()                                // alternate SB config

         sub_2D682()                                  // SB init finalize
         sub_189F8(sb, 0x104, 0xC0000000)             // SB command: full-speed mode

STEP 52  set_boot_step(ctx, 52)
         sub_2D480()                                  // IO initialization
         spi_write_register_64bit(flexio, 0xF200, 0xC0000000)
         sub_189C8(sb, 144, &result)                  // read SB status
         result == 0 => errlog 0x3039                 // IO init failure
         sub_2E46C()                                  // IO finalize

STEP 60  set_boot_step(ctx, 60)
         Phase 4 complete. SSM advances to Phase 5.
```

### Phase 5: Flash Firmware Handoff

**Dispatcher**: `powseq_phase5_flash_handoff` (0x7254)
**SSM command**: 5
**Work function**: `boot_steps_60_62_flash_handoff` (0x71A4)

```
         wait_cell_firmware_request (0x2D4D4)         // poll attention GPIO, 100x10ms
           waits for GPIO value == 1 (CELL requesting firmware)
         fail => errlog 0x3040

STEP 61  set_boot_step(ctx, 61)
         spi_write_register_64bit(flexio, 0, 0x800000)  // firmware data transfer

STEP 62  set_boot_step(ctx, 62)
         ctx->state = 6                               // mark boot complete

STEP 255 set_boot_step(ctx, 255)
         BCD encodes to 0x80 => "Power On State"
         Boot sequence complete. System running.
```

## Shutdown Flow (Phase 6)

**Dispatcher**: `powseq_phase6_shutdown` (0x7D70)
**SSM command**: 6

```
         powseq_setup() if not done                   // ensure HW init
         ctx->phase = 7
         set_boot_step_raw_bcd(0x90)                  // step display = 0x90 (shutdown)
         sub_2E206(0)                                 // begin shutdown sequence
         sub_2E47E() -> sub_2E4A2()                   // power domain teardown
         shutdown(ctx)                                // main shutdown handler
         ctx->phase = 0, ctx->state = 0               // reset state
         set_boot_step_raw_bcd(0)                     // clear step display
         sub_2E276()                                  // final power-down
```

## Warm Boot / Resume Flow (Phase 7)

**Dispatcher**: `powseq_phase7_warm_boot` (0x7EBE)
**SSM command**: 7

```
         ctx->phase = 1
         sub_2E206(1)                                 // begin resume sequence
         sub_2E47E() -> sub_2E4A2()                   // power domain restore

         boot_step_11_warm_resume (0x7DBE):
           sub_2BBB6(flexio, dword_200C9AC, sb, dword_200C9F4)   // reinit all SPI buses
           sub_2E7AE() -> sub_2E704() -> sub_2E70C()             // voltage regulator restart
           sub_2DE48()                                           // power rail re-enable
           sub_2E646(4)                                          // power gate resume
           sub_2E750()                                           // voltage check
           sub_2BBB6(dword_200C9F8)                              // power monitor bus (if feature C phase 2)
           delay_ms(1)
           sub_2E75E()                                           // power stabilization
           sub_2E646(3)                                          // SB bus resume
           sub_2BB7A(dword_200C9F4, sb_context_slot)             // release SB buses
           sub_2E668()                                           // SB power check
           sub_2BB7A(dword_200C9F8)                              // release power monitor (if feature C phase 2)
           sub_2E7BE() -> sub_2E708(0) -> sub_2E72E(0)           // power rail final check
           delay_ms(5)
           sub_2BB7A(dword_200C9AC)                              // release CELL secondary
           delay_ms(1)

STEP 11  set_boot_step(ctx, 11)
           Psbd_SetBePll() (sub_2CEA4)                           // PLL config to CELL
           sub_2BB7A(cell_rsx_flexio)                            // release FlexIO bus
           boot_final_handoff (0x874C)                           // hand off to OS
           ctx->phase = 2                                        // system resumed
```

## Complete Boot Step Summary

| Step | BCD | Function | Phase | Description |
|------|-----|----------|-------|-------------|
| 0 | 0x00 | boot_steps_0_to_20 | 1 | 12V power check |
| 1 | 0x01 | boot_steps_0_to_20 | 1 | Begin power sequencing, SPI bus init |
| 2 | 0x02 | boot_steps_0_to_20 | 1 | Post power-on delay, SB half-speed mode |
| 3 | 0x03 | boot_steps_0_to_20 | 1 | Power rail enable (enters boot_step_4) |
| 4 | 0x04 | boot_step_4_power_rail_enable | 1 | Main power rail enable + voltage check |
| 5 | 0x05 | boot_steps_0_to_20 | 1 | DC-DC converter fine tune, SB power validation |
| 6 | 0x06 | boot_steps_0_to_20 | 1 | CELL power domain enable |
| 7 | 0x07 | boot_steps_0_to_20 | 1 | Power validation entry (enters boot_steps_7_9) |
| 8 | 0x08 | boot_steps_7_9_power_validation | 1 | Deep power rail sequencing, VRM setup |
| 9 | 0x09 | boot_steps_7_9_power_validation | 1 | Interrupt setup, CELL power-on trigger, XCG PLL config |
| 10 | 0x10 | boot_steps_0_to_20 | 1 | VID setup, FlexIO prep |
| 11 | 0x11 | boot_step_11_warm_resume | 7 | Warm boot PLL config (resume only) |
| 20 | 0x20 | boot_steps_0_to_20 | 1 | PLL config complete, Phase 1 done |
| 21 | 0x21 | boot_steps_21_22_attention | 2 | CELL attention received, FlexIO link check |
| 22 | 0x22 | boot_steps_21_22_attention | 2 | Config ring write to CELL |
| 23 | 0x23 | boot_steps_23_30_clock_cfgring | 3 | Clock frequency stepping, config ring build |
| 30 | 0x30 | boot_steps_23_30_clock_cfgring | 3 | Config ring sent via SPI |
| 31 | 0x31 | boot_steps_31_60_training | 4 | CELL SPI init, config register validation |
| 32 | 0x32 | boot_steps_31_60_training | 4 | RSX VID setup |
| 40 | 0x40 | boot_steps_31_60_training | 4 | BitTraining (FlexIO calibration) |
| 50 | 0x50 | boot_steps_31_60_training | 4 | ByteTraining BE-RS (CELL to RSX) |
| 51 | 0x51 | boot_steps_31_60_training | 4 | ByteTraining BE-SB (CELL to Southbridge) |
| 52 | 0x52 | boot_steps_31_60_training | 4 | IO initialization, SB full-speed mode |
| 60 | 0x60 | boot_steps_31_60_training / flash_handoff | 4/5 | Flash firmware sequence begin |
| 61 | 0x61 | boot_steps_60_62_flash_handoff | 5 | Firmware data transfer to CELL |
| 62 | 0x62 | boot_steps_60_62_flash_handoff | 5 | Final SPI transfer complete |
| 255 | 0x80 | boot_steps_60_62_flash_handoff | 5 | System running (Power On State) |
| - | 0x90 | powseq_phase6_shutdown | 6 | Shutdown sequence |

## Error Codes By Boot Step

| Step | Possible Errors | Meaning |
|------|----------------|---------|
| 0 | 0x3001 | 12V main rail failure |
| 7-9 | 0x3002, 0x3003, 0x3004 | Power validation failures (sub_2DB7C return codes) |
| pre-phase | 0x3000 | Pre-boot HW check failure (sub_2DB30) |
| 21 | 0x3010 | CELL attention timeout (SEQ1) |
| 21 | 0x3011 | Unexpected attention type (status bits [10:11] != 2) |
| 21 | 0x3013 | FlexIO link status failure (bits 16/17) |
| 22 | 0x3012 | Config ring write/verify failure |
| 23-30 | 0x3020 | Clock stability check failure after frequency step |
| 31 | 0x3030 | CELL attention timeout (pre-training) |
| 31 | 0x3031 | CELL config bits [21:20] != 1 |
| 31 | 0x3032 | CELL config bits [3:2] != 3 |
| 32 | 0x3033 | RSX VID ready timeout (bit 23 not set after 1000 polls) |
| 40 | 0x3034 | BitTraining (FlexIO calibration) failure |
| 50 | 0x3035 | BE-RS byte training loop timeout |
| 50 | 0x3036 | BE-RS byte training completion flag timeout |
| 50 | 0x3037 | RSX VID response invalid after training |
| 51 | 0x3038 | BE-SB byte training failure |
| 52 | 0x3039 | IO initialization failure (SB status == 0) |
| 60 | 0x3040 | CELL firmware request timeout (GPIO not asserted in 1s) |
| any | 0x2001/0x2101 | CELL SPI write/read error |
| any | 0x2003/0x2103 | Southbridge SPI write/read error |
| any | 0x1200+zone | Thermal shutdown |

## Functions Renamed in IDA

| Address | Name | Role |
|---------|------|------|
| 0x7EF6 | powseq_task_main | RTOS task, message loop dispatcher |
| 0x779A | powseq_phase1_cold_boot | Phase 1 entry: setup + steps 0-20 |
| 0x71F6 | powseq_phase2_attention | Phase 2 wrapper: attention handling |
| 0x7218 | powseq_phase3_clock_cfgring | Phase 3 wrapper: clock + config ring |
| 0x7236 | powseq_phase4_training | Phase 4 wrapper: FlexIO training |
| 0x7254 | powseq_phase5_flash_handoff | Phase 5 wrapper: firmware handoff |
| 0x7D70 | powseq_phase6_shutdown | Phase 6: shutdown/letup |
| 0x7EBE | powseq_phase7_warm_boot | Phase 7: resume from sleep |
| 0x74A4 | boot_steps_0_to_20 | Steps 0-20: power sequencing |
| 0x6E0E | boot_steps_21_22_attention | Steps 21-22: attention + config ring |
| 0x8760 | boot_steps_23_30_clock_cfgring | Steps 23-30: clock step + ring send |
| 0x6EBC | boot_steps_31_60_training | Steps 31-60: training + IO init |
| 0x71A4 | boot_steps_60_62_flash_handoff | Steps 60-62, 255: firmware + running |
| 0x7272 | boot_steps_7_9_power_validation | Steps 7-9: deep power check |
| 0x87C2 | boot_step_4_power_rail_enable | Step 4: rail enable sub-phase |
| 0x7DBE | boot_step_11_warm_resume | Step 11: warm boot PLL path |
| 0x870C | boot_load_board_config | Read NVS 0x3901 board flags |
| 0x874C | boot_final_handoff | Hand off to OS boot |
| 0x6E94 | poll_rsx_vid_ready | Poll RSX VID bit 23 (1000x) |
| 0x7758 | powseq_setup | One-time HW init |
| 0x2D4D4 | wait_cell_firmware_request | Poll attention GPIO (100x10ms) |
| 0x1A0E8 | set_boot_step_raw_bcd | Direct BCD write (no conversion) |
