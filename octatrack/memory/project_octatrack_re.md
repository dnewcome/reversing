---
name: Octatrack RE project state
description: Hardware details, firmware format findings, and RE approach for Elektron Octatrack DPS-1 OS 1.40C
type: project
---

# Elektron Octatrack DPS-1 Reverse Engineering

## Hardware (confirmed via teardown research)
- **CPU**: Freescale ColdFire MCF54454 (32-bit, 266 MHz, ~410 MIPS, ColdFire V4e / 68k descendant)
- **DSP**: Freescale DSP56721AG (24-bit, 200 MHz, dual DSP56300 cores, ~810 MIPS total)
- **RAM**: ~80 MB DDR266
- **Debug port**: BDM (Background Debug Mode), 10-pin header — NOT JTAG
- **Storage**: Compact Flash card (user-accessible)
- SysEx device ID: 0x05; internal product code: "ELEK017"

## Firmware files
- `OCTATRACK_OS1.40C.bin` — CF card update format, starts with "ELUP" magic (4 bytes) + encrypted body (entropy 7.998, ~AES-level)
- `OCTATRACK_OS1.40C.syx` — MIDI SysEx format, 7460 packets (88 bytes each), 7-bit MIDI encoded, encrypted payload (entropy 7.88 after decode)

## SysEx packet format (decoded)
- Header: `f0 00 20 3c 05 00 [cmd]` (Elektron mfr ID 00:20:3c, device 05, cmd 7e=data, 7f=end)
- Each 88-byte packet → 80 bytes payload → 7-to-8 bit decode → 70 bytes
- First 7 decoded bytes per packet = metadata (sequence/address counter)
- Remaining 63 bytes = encrypted firmware payload
- 7459 data packets × 63 bytes ≈ 469,917 bytes ≈ BIN payload size
- First packet (cmd 0x7e, subcmd 0x0d) = plaintext header with "ELEK017", "8     1", "1.40C" strings

## Encryption
- Both .bin and decoded .syx payload are encrypted (not just compressed)
- No known public decryption tools
- Unofficial Machinedrum firmware (Yatao Li, Justin Valer) exists but tools were never published
- Key must be stored in device bootloader/ROM or BDM-accessible flash

## Attack vectors
1. **BDM hardware extraction** — dump bootloader from physical device using BDM dongle
2. **Flash chip desoldering** — read NOR/NAND flash directly
3. **Differential analysis** — compare multiple firmware versions for patterns
4. **Community collaboration** — Machinedrum RE team (elektronauts.com) kept tools private

## Useful references
- elektronauts.com - "Octatrack CPU chip model?" thread
- elektronauts.com - "Machinedrum SPS1-UW X.04 released unofficial" thread
- github.com/dagargo/elektroid (modern Elektron SysEx, different protocol)
- NXP MCF54455 datasheet (same family as MCF54454)
- Freescale DSP56721 datasheet

**Why:** User wants to emulate the hardware and make firmware modifications.
**How to apply:** Focus RE efforts on BDM hardware extraction or community collaboration rather than software-only approaches. ColdFire V4e ISA is well-supported in Ghidra if decrypted firmware is obtained.
