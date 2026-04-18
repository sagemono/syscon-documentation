# Complete Call Site Reference

Target: Mullion v1.0.0_k1 firmware

Total call sites: 62

## Error Code Format

```
Full 32-bit code: 0xA0_SS_CERR
  A0  = Fixed prefix (computed via unsigned wraparound)
  SS  = Boot step (BCD encoded)
  C   = Category nibble
  ERR = 12-bit error number
```

## Call Site Table

### Static Error Emitters (Category 0x1xxx - System Errors)

| # | Call Addr | Function | Error Code | Description |
|---|-----------|----------|------------|-------------|
| 1 | 0x12D3E | sub_12D26 | 0x1001 | RSX VRM power fail (alternate path). LDR R0,=0x1002; SUBS R0,#1. Disables interrupt 8, sends shutdown cmd 0x3E04000A. |
| 2 | 0x12D1C | sub_12D08 | 0x1002 | RSX VRM power fail (primary). Disables interrupt 8, sends shutdown cmd 0x3E04000A. NEC/TOKIN capacitor failure indicator. |
| 3 | 0x12CD6 | sub_12CC2 | 0x1103 | Thermal alert (Mullion-specific). Disables interrupt 10, sends shutdown cmd 0x3E040008. CELL temperature monitor. |
| 4 | 0x14A04 | sub_149B8 | 0x1200+zone | Thermal shutdown (first path). Dynamic: base 0x1200 + thermal zone index. Logs "[WMZONE] Thermal Shutdown". |
| 5 | 0x14A76 | sub_149B8 | 0x1200+zone | Thermal shutdown (second path). Same function, alternate code path for repeated thermal events. |
| 6 | 0x343F4 | sub_343A2 | 0x1301 | PLL lock lost. Monitors PLL lock status; when PLL transitions from locked to unlocked, logs "PLL lock -> unlock". |
| 7 | 0x12D90 | sub_12D84 | 0x14FF | Hardware checkstop. Disables interrupt 9, sends shutdown cmd 0x3E040008. CPU/GPU BGA, cache, or memory failure. |
| 8 | 0x12D7A | sub_12D70 | 0x15FF | Fatal unknown hardware error. Disables interrupt 11, sends shutdown cmd 0x3E04000A. |
| 9 | 0x102EA | sub_1029C | 0x1601 | CELL livelock detection. BE Attention handler checks status bits 31+24 (0x81000000). If both set, "[SSM] BE Livelock condition is detected !!". Recovery attempted via dword_2000C5C handler if registered. |
| 10 | 0x12CB8 | sub_12CA8 | 0x1701 | BE Attention error. Disables interrupt 12, sends shutdown cmd 0x3E05000B. Fatal CELL error during operation. |
| 11 | 0x12CE8 | sub_12CDC | 0x1802 | RSX initialization failure. Disables interrupt 13. RSX interrupt/init error. |
| 12 | 0x1F40C | sub_1F3FC | 0x1902 | RTC access failure. RTC subsystem communication error. |

#### Thermal Zone Mapping (for 0x1200+zone)

| Zone | Error Code | Name | Component |
|------|------------|------|-----------|
| 0 | 0x1200 | 1st BE Primary | CELL Processor |
| 1 | 0x1201 | RSX Primary | RSX GPU |
| 14 | 0x1214 | SB | Southbridge |

---

### Dynamic Error Emitters (Category 0x2xxx - Fatal Device Errors)

All functions in this category compute the error as `base + errlog_slot_for_ctx(context)`.

#### Device Write Errors (base 0x2000)

| # | Call Addr | Function | Error Pattern | Description |
|---|-----------|----------|---------------|-------------|
| 13 | 0x29FE6 | dev_write_reg_var | 0x2000+slot | SPI variable-length register write. Sends command byte 0x11 + address + data via FIFO. Used for CELL/RSX/SB SPI devices. |
| 14 | 0x2A3AE | sub_2A37A | 0x2000+slot | I2C register write with remap. 2-byte write (reg+value). Registers 3-8 remapped to 9-14 (offset +6). |
| 15 | 0x2A400 | sub_2A3BA | 0x2000+slot | I2C write with fixed subaddress 24 (0x18). Writes 1 byte. Both write and read errors emit 0x2000 (no distinction). Specific sensor/peripheral. |
| 16 | 0x2A454 | sub_2A412 | 0x2000+slot | Variable-width I2C register write. Lookup table at unk_40C04 determines 1-byte vs 2-byte (big-endian 16-bit) write. Sends 2 or 3 byte packet. |
| 17 | 0x2A4DA | sub_2A4B0 | 0x2000+slot | Variable-width I2C register read - address write phase failed. Sends register address byte before read; this is the write-phase error. |
| 18 | 0x2A98E | dev_write_reg8 | 0x2000+slot | Simple 8-bit I2C register write with RTOS semaphore. 2-byte packet (reg+value). Used by clock generators (slots 16-19). |
| 19 | 0x38608 | sub_385DE | 0x2000+slot | Clock generator I2C register write. 3-byte packet: reg + 0x01 (write cmd) + value. Used by ICS clock gen chips. |
| 20 | 0x38510 | sub_384E6 | 0x2000+slot | Generic I2C write with error logging. Thin wrapper around bus_xfer_write3_select_mode. |
| 21 | 0x386CA | sub_38694 | 0x2000+slot | **NOTE**: Despite address in 0x386xx, this xref is actually at the write-error path within what is primarily a read function (see comment). Actually this is sub_386D6 - generic I2C write. |
| 22 | 0x38700 | sub_386D6 | 0x2000+slot | Generic I2C write with error logging. Alternate bus write function. |
| 23 | 0x27F02 | hdmi_write_regs | 0x2000+slot | HDMI (SiI) register write. Dual I2C address selection via bit 0 of register. Debug: "[hdmiERROR] SiI chip Register Write Access Error ch.%d, DevID[%02X], Off[%02X] TID[%d] ECode[%08X]". Guarded by semaphore 0x41. |
| 24 | 0x18448 | dev_write_reg8_8 | 0x2000+slot | DVE/MultiAV 8-bit register write. |
| 25 | 0x38C1A | sub_38BDA | 0x2000+slot | Device init/command write. Writes 0x3024 then 0x3020 command words, waits 500ms (sub_38BA0). Timeout emits write error. |
| 26 | 0x3C270 | shadow16_updatebit_and_push_cmd6 | 0x2000+slot | SB shadow16 bit update + write. Maintains shadow copy of 16-bit register (offsets 14-15), updates specific bit, writes via command byte 0x06. |
| 27 | 0x3C2EE | shadow16_setbit_and_push | 0x2000+slot | SB shadow16 set bit + write. Sets bit in shadow register pair (offsets 10-11), writes 16-bit value via command byte 0x02. |
| 28 | 0x3C36C | shadow16_update_and_push | 0x2000+slot | SB shadow16 clear bit + write. Clears bit in shadow register pair (offsets 10-11), writes via command byte 0x02. Complement of setbit_and_push. |
| 29 | 0x3C4C4 | shadow16_write_pair | 0x2000+slot | SB shadow16 2-byte write. Updates shadow copy, writes 3-byte I2C command (register + 2 data bytes). |
| 30 | 0x3C694 | shadow16_write_single | 0x2000+slot | SB shadow16 1-byte write. Updates one shadow byte, writes 2-byte I2C command (register + 1 data byte). |

#### Device Read Errors (base 0x2100)

| # | Call Addr | Function | Error Pattern | Description |
|---|-----------|----------|---------------|-------------|
| 31 | 0x29F40 | dev_read_reg_var | 0x2100+slot | SPI variable-length register read (first path). Sends read command via FIFO, reads response. |
| 32 | 0x29F9C | dev_read_reg_var | 0x2100+slot | SPI variable-length register read (second path). Same function, alternate error path. |
| 33 | 0x2A36E | sub_2A320 | 0x2100+slot | I2C indexed read. Write-then-read: sends 1-byte register address, reads 1 byte back. Standard I2C read pattern. |
| 34 | 0x2A536 | sub_2A4B0 | 0x2100+slot | Variable-width I2C register read - data read phase failed. Reads 1 or 2 bytes depending on unk_40C04 lookup, reassembles big-endian 16-bit. |
| 35 | 0x2AA80 | dev_read_reg8 | 0x2100+slot | Simple 8-bit I2C register read with RTOS semaphore. Write-then-read pattern (8448 = 0x2100). Used by clock generators (slots 16-19). |
| 36 | 0x2AC94 | sub_2AC5E | 0x2100+slot | CELL SPI status poll timeout. Polls device register up to a2 times checking bit 7 of high byte. If loop exhausts, emits read/poll timeout. |
| 37 | 0x385D2 | sub_3859C | 0x2100+slot | Clock generator I2C register read. Write-then-read: sends register number, reads 2 bytes, returns second byte (8448 = 0x2100). |
| 38 | 0x384DA | sub_384A4 | 0x2100+slot | Generic I2C read with error logging. |
| 39 | 0x386CA | sub_38694 | 0x2100+slot | Generic I2C read with error logging. Alternate bus read function. |
| 40 | 0x27E7A | hdmi_read_regs | 0x2100+slot | HDMI (SiI) register read. Dual I2C address selection. Debug: "[hdmiERROR] SiI chip Register Read Access Error ch.%d, DevID[%02X], Off[%02X] TID[%d] ECode[%08x]". Guarded by semaphore 0x41 + dword_20010CC enable flag. |
| 41 | 0x18414 | dev_read_reg8_after_index | 0x2100+slot | DVE/MultiAV 8-bit register read with index write. |
| 42 | 0x38C5A | sub_38C26 | 0x2100+slot | Device init/command read. Writes 0x3020, waits 500ms, reads 0x3024. Timeout emits read error. Read complement of sub_38BDA. |
| 43 | 0x3C3C8 | sub_3C37C (shadow16_read_pair) | 0x2100+slot | SB shadow16 2-byte read. Reads 16-bit value, updates shadow registers (offsets 8-15) based on register index via switch (reg 0->8,9; 1->8,9 swapped; 2->10,11; 3->10,11 swapped; 6->14,15; 7->14,15 swapped). |
| 44 | 0x3C51C | sub_3C4D4 (shadow16_bit_test) | 0x2100+slot | SB shadow16 bit test. Reads full 16-bit register 0, tests specific bit (0-15). Returns 1 if set, 0 if clear. |
| 45 | 0x3C59C | sub_3C554 (shadow16_read_single) | 0x2100+slot | SB shadow16 1-byte read. Reads 1 byte from register, updates corresponding shadow byte (switch maps reg 0->offset 8, 1->9, ... 7->15). |

#### Device Slot Mapping (for 0x2xxx errors)

| Slot | Context Address | I2C Addr | Device | Write Error | Read Error | Verify Error |
|------|-----------------|----------|--------|-------------|------------|--------------|
| 1 | 0x200C9A8 | SPI | CELL/RSX FlexIO | 0x2001 | 0x2101 | - |
| 2 | 0x200C9E4 | SPI | RSX | 0x2002 | 0x2102 | - |
| 3 | 0x200C9F0 | SPI | Southbridge | 0x2003 | 0x2103 | 0x2203 |
| 16 | (IC5001 ctx) | 0xD2 | Clock Gen IC5001 | 0x2010 | 0x2110 | - |
| 17 | unknown_device_slot_1 | 0xD8 | Clock Gen IC5003 | 0x2011 | 0x2111 | - |
| 18 | xdr_context_id_probable | 0xDE | Clock Gen IC5002 | 0x2012 | 0x2112 | - |
| 19 | unk_42074 | 0xDC | Clock Gen IC5004 (ICS9214) | 0x2013 | 0x2113 | - |
| 32 | g_hdmi_ctx (0x2001268) | I2C | HDMI Controller (SiI, IC2502) | 0x2020 | 0x2120 | - |
| 34 | 0x420A4 | 0x42 | DVE MultiAV (IC2406) | 0x2022 | 0x2122 | - |

#### Write Verify Errors (base 0x2200)

| # | Call Addr | Function | Error Pattern | Description |
|---|-----------|----------|---------------|-------------|
| 46 | 0x18DCA | sub_18D30 | 0x2200+slot | Southbridge write-with-verify. Writes in 128-byte chunks, reads back to compare. Retries via sub_188CA on mismatch. Emits error only after retry also fails. Only observed with slot 3 (SB) = 0x2203. |

---

### Static Error Emitters (Category 0x3xxx - Boot Errors)

| # | Call Addr | Function | Error Code | Boot Step | Description |
|---|-----------|----------|------------|-----------|-------------|
| 47 | 0x077D0 | sub_779A | 0x3000 | Pre-boot | Pre-boot check failure. sub_2DB30() returned non-zero before Phase 1 entry. Boot sequence never starts. |
| 48 | 0x074C6 | sub_74A4 | 0x3001 | Early | 12V power check failure. sub_2E842 checks power status bits; sub_2DB46() non-zero after sub_2E7CE() returns zero. Power sequencing error. |
| 49 | 0x0747A | sub_7272 | 0x3002 | 7 | Power sequence validation failure. Step 7 power rail check. |
| 50 | 0x06E22 | sub_6E0E | 0x3010 | 21 | Post-attention check 1. LDR R0,=0x3035; SUBS R0,#0x25. First post-CELL-attention status validation. |
| 51 | 0x06E5A | sub_6E0E | 0x3011 | 21 | Post-attention check 2. Second status register validation after BE attention. |
| 52 | 0x06E84 | sub_6E0E | 0x3012 | 22 | Post-attention check 3. Third status check, CELL/RSX status word validation. |
| 53 | 0x2E908 | sub_2E8E4 | 0x3013 | varies | FlexIO link status failure. Checks bits 16+17 of CELL/RSX FlexIO status word. Bit 17 clear OR bit 16 set = error. |
| 54 | 0x0879A | sub_8760 | 0x3020 | varies | Clock/config ring setup failure. Called during phase 3 for clock configuration and config ring transmission. |
| 55 | 0x06F08 | sub_6EBC | 0x3031 | 31 | CELL SPI init - config bits [21:20] check. Expected value 1 in bits [21:20] of config register. |
| 56 | 0x06F22 | sub_6EBC | 0x3032 | 31 | CELL SPI init - config bits [3:2] check. Expected value 3 in bits [3:2] of config register. |
| 57 | 0x06FE2 | sub_6EBC | 0x3034 | 40 | BitTraining (FlexIO calibration) failure. LDR R0,=0x3033; ADDS R0,#1. FlexIO link calibration timeout. Primary GPU BGA failure indicator. |
| 58 | 0x06CA4 | sub_6BE8 | 0x3035 | 50 | ByteTraining BE-RS loop timeout. CELL-to-RSX byte training, polling loop exhausted. Logs "[POWERSEQ] Error : BE-RS byte training failed." |
| 59 | 0x06CDC | sub_6BE8 | 0x3036 | 50 | ByteTraining BE-RS completion flag timeout. Training threshold reached but completion flag (bit 27) never set. |
| 60 | 0x07096 | sub_6EBC | 0x3037 | varies | RSX VID setup failure. LDR R0,=0x3033; ADDS R0,#4. RSX voltage ID configuration failed. |
| 61 | 0x06DBE | sub_6D3E | 0x3038 | 51 | ByteTraining BE-SB failure. CELL-to-Southbridge byte training. LDR R0,=0x3035; ADDS R0,#3. |
| 62 | 0x07186 | sub_6EBC | 0x3039 | 52 | IO initialization failure. LDR R0,=0x3033; ADDS R0,#6. Southbridge IO init after training. |
| 63 | 0x071B6 | sub_71A4 | 0x3040 | 60 | Flash firmware sequence failure. MOVS R0,#0x3040. Flash-related boot step. |

**Note**: sub_2E842 (0x2E866) emits 0x3001 via sub_74A4, it's the power check called during the early boot path. The call at 0x074C6 is the actual emission site.

---

## Functions That Emit Multiple Error Codes

| Function | Address | Codes Emitted | Purpose |
|----------|---------|---------------|---------|
| sub_6BE8 | 0x6BE8 | 0x3035, 0x3036 | BE-RS byte training (loop timeout vs flag timeout) |
| sub_6E0E | 0x6E0E | 0x3010, 0x3011, 0x3012 | Post-attention triple status check |
| sub_6EBC | 0x6EBC | 0x3031, 0x3032, 0x3034, 0x3037, 0x3039 | Main training sequence (5 distinct failure points) |
| sub_149B8 | 0x149B8 | 0x1200+zone (x2) | Thermal shutdown (two code paths) |
| dev_read_reg_var | 0x29EBE | 0x2100+slot (x2) | SPI read (two error paths) |
| sub_2A4B0 | 0x2A4B0 | 0x2000+slot, 0x2100+slot | Variable-width read (write-phase vs read-phase error) |

---

## Call Sites By Function Category

### Boot Sequence (16 sites)

```
sub_779A       -> 0x3000  (pre-boot check)
sub_74A4       -> 0x3001  (12V power)
sub_7272       -> 0x3002  (power validation)
sub_6E0E       -> 0x3010, 0x3011, 0x3012  (post-attention checks)
sub_8760       -> 0x3020  (clock/config ring)
sub_6EBC       -> 0x3031, 0x3032, 0x3034, 0x3037, 0x3039  (training)
sub_6BE8       -> 0x3035, 0x3036  (BE-RS byte training)
sub_6D3E       -> 0x3038  (BE-SB byte training)
sub_71A4       -> 0x3040  (flash firmware)
```

### Runtime Hardware Monitors (12 sites)

```
sub_12D26      -> 0x1001  (RSX VRM alt)
sub_12D08      -> 0x1002  (RSX VRM)
sub_12CC2      -> 0x1103  (thermal alert)
sub_149B8      -> 0x1200+zone (x2)  (thermal shutdown)
sub_343A2      -> 0x1301  (PLL unlock)
sub_12D84      -> 0x14FF  (checkstop)
sub_12D70      -> 0x15FF  (fatal unknown)
sub_1029C      -> 0x1601  (livelock)
sub_12CA8      -> 0x1701  (BE attention)
sub_12CDC      -> 0x1802  (RSX init)
sub_1F3FC      -> 0x1902  (RTC)
```

### SPI Device Access (5 sites)

```
dev_read_reg_var   -> 0x2100+slot (x2)
dev_write_reg_var  -> 0x2000+slot
sub_2AC5E          -> 0x2100+slot  (poll timeout)
sub_2E8E4          -> 0x3013  (FlexIO link status, static)
```

### I2C Device Access - Generic (12 sites)

```
sub_384A4      -> 0x2100+slot  (read)
sub_384E6      -> 0x2000+slot  (write)
sub_3859C      -> 0x2100+slot  (clock gen read)
sub_385DE      -> 0x2000+slot  (clock gen write)
sub_38694      -> 0x2100+slot  (read alt)
sub_386D6      -> 0x2000+slot  (write alt)
sub_2A320      -> 0x2100+slot  (indexed read)
sub_2A37A      -> 0x2000+slot  (remapped write)
sub_2A3BA      -> 0x2000+slot  (fixed-addr write)
sub_2A412      -> 0x2000+slot  (variable-width write)
sub_2A4B0      -> 0x2000+slot, 0x2100+slot  (variable-width read)
```

### I2C Device Access - Semaphore-Protected (4 sites)

```
dev_write_reg8     -> 0x2000+slot  (8-bit write + mutex)
dev_read_reg8      -> 0x2100+slot  (8-bit read + mutex)
dev_write_reg8_8   -> 0x2000+slot  (DVE write)
dev_read_reg8_after_index -> 0x2100+slot  (DVE read)
```

### HDMI (2 sites)

```
hdmi_write_regs    -> 0x2000+slot  (SiI chip write)
hdmi_read_regs     -> 0x2100+slot  (SiI chip read)
```

### Southbridge Shadow16 (8 sites)

```
shadow16_updatebit_and_push_cmd6  -> 0x2000+slot  (bit update + write reg 6)
shadow16_setbit_and_push          -> 0x2000+slot  (set bit + write reg 2)
shadow16_update_and_push          -> 0x2000+slot  (clear bit + write reg 2)
shadow16_write_pair               -> 0x2000+slot  (2-byte write)
shadow16_write_single             -> 0x2000+slot  (1-byte write)
sub_3C37C (shadow16_read_pair)    -> 0x2100+slot  (2-byte read)
sub_3C4D4 (shadow16_bit_test)     -> 0x2100+slot  (bit test)
sub_3C554 (shadow16_read_single)  -> 0x2100+slot  (1-byte read)
```

### Device Init (2 sites)

```
sub_38BDA      -> 0x2000+slot  (init write sequence + timeout)
sub_38C26      -> 0x2100+slot  (init read sequence + timeout)
```

### Write Verify (1 site)

```
sub_18D30      -> 0x2200+slot  (write + readback compare + retry)
```

---

## Possible Error Codes By Device

| Device | Write (0x20xx) | Read (0x21xx) | Poll (0x21xx) | Verify (0x22xx) |
|--------|----------------|---------------|---------------|-----------------|
| CELL/FlexIO (slot 1) | 0x2001 | 0x2101 | 0x2101 | - |
| RSX (slot 2) | 0x2002 | 0x2102 | - | - |
| Southbridge (slot 3) | 0x2003 | 0x2103 | - | 0x2203 |
| Clock Gen IC5001 (slot 16) | 0x2010 | 0x2110 | - | - |
| Clock Gen IC5003 (slot 17) | 0x2011 | 0x2111 | - | - |
| Clock Gen IC5002 (slot 18) | 0x2012 | 0x2112 | - | - |
| Clock Gen IC5004 (slot 19) | 0x2013 | 0x2113 | - | - |
| HDMI SiI (slot 32) | 0x2020 | 0x2120 | - | - |
| DVE MultiAV (slot 34) | 0x2022 | 0x2122 | - | - |
