# reversing

Reverse engineering notes and artifacts for audio and music gear.

## Targets

| Device | Manufacturer | Type | Directory |
|--------|-------------|------|-----------|
| CDJ-3000 | AlphaTheta (Pioneer DJ) | DJ media player | [`cdj-3000/`](cdj-3000/) |
| Octatrack DPS-1 | Elektron | Sampler / sequencer | [`octatrack/`](octatrack/) |
| Fire | Akai | FL Studio controller / pad grid | *(future)* |

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

## Akai Fire

No firmware updates have ever been released publicly. The host-facing protocol is
well-documented (USB class-compliant MIDI + SysEx) via the
[SEGGER blog series](https://blog.segger.com/decoding-the-akai-fire-part-1/), but
firmware internals are unexplored.

Hardware notes:
- MCU: STM32 (specific model TBD — needs teardown/chip marking)
- A 10-pin Cortex debug connector is present inside the case ("unfinished engineering")
- STM32 bootloader mode: hold Step + Note + Drum + Perform on power-up

Likely approach: identify STM32 model from chip markings, use the Cortex debug header
to dump firmware with a J-Link or ST-Link. No encryption expected (no update mechanism
to protect against).
