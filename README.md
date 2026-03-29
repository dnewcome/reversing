# reversing

Reverse engineering notes and artifacts for audio and music gear.

## Targets

| Device | Manufacturer | Type | Directory |
|--------|-------------|------|-----------|
| CDJ-3000 | AlphaTheta (Pioneer DJ) | DJ media player | [`cdj-3000/`](cdj-3000/) |
| Octatrack DPS-1 | Elektron | Sampler / sequencer | [`octatrack/`](octatrack/) |

## CDJ-3000

Firmware v3.22 (`CDJ3Kv322.UPD`). The update file is an ISO 9660 filesystem encrypted
in-place with LUKS1 (AES-XTS-plain64, 512-bit key), followed by an unencrypted 15-byte
trailer. Hardware runs a Rockchip RK3399 SoC on newer revisions, Renesas on older ones.
Main blocker: obtaining the 64-byte binary AES key. See [`cdj-3000/PLAN.md`](cdj-3000/PLAN.md).

## Octatrack DPS-1

Firmware OS 1.40C. CPU is a Freescale ColdFire MCF54454; DSP is a Freescale DSP56721.
Firmware is encrypted in both the `.bin` (CF card) and `.syx` (MIDI SysEx) formats.
Main blocker: extracting the decryption key via BDM hardware debug port.
See [`octatrack/memory/`](octatrack/memory/) for findings.
