# CDJ-3000 Firmware Reversing Plan

## Goal
Reverse engineer the AlphaTheta CDJ-3000 firmware to understand hardware architecture,
memory maps, peripherals, and software platform.

## Target File
`CDJ3Kv322.UPD` — firmware v3.22 for CDJ-3000

---

## Current Findings

### File Format (fully understood)
Source: analysis of `CDJ3Kv322.UPD` + `github.com/emmaworley/cdj3k-root` Makefile

The `.UPD` file is an **ISO 9660 filesystem encrypted in-place with LUKS1**, followed by
a 15-byte unencrypted trailer:

```
[ ISO 9660 image ]  →  [ LUKS1 encrypt in-place ]  →  [ XDJ-RR{ver}\x00 ][ CRC32 LE ]
                        (32MB header space appended)     11 bytes            4 bytes
```

Build process (from Makefile):
1. `genisoimage` builds an ISO from `src/` scripts
2. 32 MB of zeros appended to make room for LUKS header
3. `cryptsetup reencrypt --encrypt --type luks1 --cipher aes-xts-plain64 --key-size 512 --key-file aes256.key`
4. CRC32 of the encrypted file computed (little-endian, 4 bytes)
5. Trailer appended: `XDJ-RR{ver}\x00` + CRC32

**CRC32 verified:** stored `0x3920bd43` == computed `0x3920bd43` ✓

### LUKS Header Parameters
| Field | Value |
|-------|-------|
| Format | LUKS v1 |
| Cipher | AES-XTS-plain64 |
| Key length | 512-bit (two AES-256 keys for XTS) |
| Key file | `aes256.key` (64 bytes binary) |
| Hash | SHA-256 PBKDF2 |
| PBKDF2 iterations | 1,347,367 |
| UUID | `9424e626-da90-4eb0-a72f-a1a32e8326e0` |
| Payload offset | Sector 4096 (byte offset 0x200000) |
| Encrypted payload | ~153 MB ISO 9660 filesystem |

### Hardware Variants
Two hardware revisions exist, distinguished by eMMC layout:

| Variant | SoC | eMMC device | Overlay partition |
|---------|-----|-------------|-------------------|
| Older | Renesas (unknown model) | `/dev/mmcblk0` | `/dev/mmcblk0p5` |
| Newer | **Rockchip RK3399** | `/dev/mmcblk1` | `/dev/mmcblk1p8` |

The ISO filename `CDJ3K-RK3399.iso` in the Makefile confirms the Rockchip variant.

### Device Internals (from update scripts)
- **Init system**: systemd (`systemctl`, `systemctl reboot`)
- **Display server**: `xserver-nodm` (X11 without display manager)
- **SSH daemons**: both `sshd.socket` (OpenSSH) and `dropbear` present but not started by default
- **Main app path**: `/home/root/pdj/EP{number}` (Pioneer DJ media player app)
- **Startup script**: `/home/root/scripts/apl_start.sh`
- **Runs as**: root (`/home/root/`)

### Update Process (3 phases)
The device finds `CDJ3Kv000.UPD` on USB, decrypts the LUKS container, mounts the ISO,
and calls `usb_update.sh` from the existing update daemon:

**Phase 0 — `usb_update.sh`** (runs from ISO via existing update daemon):
- Installs `phase1.sh` → `/dev/mmcblk{x}p{y}` overlay as `pdj.tar.gz/scripts/apl_start.sh`
- Installs `phase2.sh`, `payload.sh`, `.unroot.sh` into the tar
- Triggers reboot (Rockchip) or manual power cycle prompt (Renesas)

**Phase 1 — `phase1.sh`** (runs as `apl_start.sh` on next boot):
- Extracts tar, replaces itself with `phase2.sh` disguised as the Pioneer app binary
- Triggers reboot again

**Phase 2 — `phase2.sh`** (runs as Pioneer app on next boot):
- Prepends `payload.sh` to the real `apl_start.sh`
- Triggers final reboot

**Payload — `payload.sh`** (prepended to `apl_start.sh`, runs on every boot):
```bash
systemctl start sshd.socket
systemctl start dropbear
```

### Partition Layout (known)
- Rockchip model eMMC: `/dev/mmcblk1`, overlay at `p8`
- Renesas model eMMC: `/dev/mmcblk0`, overlay at `p5`
- Overlay partition stores `pdj.tar.gz` which is extracted over the running rootfs

---

## Phase 1 — Find the Decryption Key

**Critical blocker.** The key is a 64-byte binary file (`aes256.key`) stored somewhere
in the device's existing rootfs or secure storage. The `emmaworley/cdj3k-root` README
explicitly says to obtain it yourself from hardware.

### 1a. Hardware Key Extraction (highest priority)

- [ ] **UART serial console** — open the CDJ, find UART test pads on the PCB (commonly
  labeled TX/RX/GND near the SoC). Connect at 115200 8N1. Capture the full boot log.
  The update daemon that calls `cryptsetup --key-file` will log the key file path.
  If the console is writable, you have a root shell.
- [ ] **Intercept `aes256.key` path during update** — with UART open, trigger a firmware
  update and watch for the `cryptsetup reencrypt` or `luksOpen` call and its `--key-file`
  argument. Then read that file from the shell.
- [ ] **`dmsetup table --showkeys`** — once the LUKS container is open during an update,
  this command reveals the AES master key in hex from the running kernel.
- [ ] **eMMC dump** — the running rootfs on eMMC is almost certainly unencrypted (only
  the update package is encrypted). Desolder or use an eMMC clip to read `mmcblk1`
  directly. Find `aes256.key` in the rootfs filesystem.
- [ ] **JTAG** — RK3399 JTAG pinout is well-documented. Use a debug probe to dump
  memory during an update and locate the 64-byte key.

### 1b. Leverage the Rooting Tool

The `emmaworley/cdj3k-root` project already has all the tooling to build a custom `.UPD`
once you have the key. If you can obtain a rooted CDJ (or root one via another method):

- [ ] SSH into a rooted CDJ and `cat` the `aes256.key` file directly
- [ ] Check if the key path is referenced in any update-related binary or script on the rootfs
- [ ] Look in `/etc/`, `/usr/share/`, `/home/root/` for the key file

### 1c. Try Known/Weak Passphrases

The key is a binary file, not a passphrase — but LUKS also supports passphrases in slot 0.
Given 1.3M PBKDF2 iterations, each attempt takes ~1-2 seconds on a CPU.

- [ ] Try: `pioneer`, `cdj3000`, `CDJ-3000`, `alphatheta`, `rekordbox`, `XDJ-RR`, UUID string, empty
- [ ] Compare LUKS UUIDs across multiple firmware versions — if UUID is static across
  versions, the key is a static device secret, not per-version

### 1d. Community

- [ ] Contact `emmaworley` / `_ichi_nichi_` on X — they clearly have the key
- [ ] Search for CDJ-3000 / XDJ-RR UART teardown threads; the Rockchip RK3399 is
  well-understood and UART is usually easy to find
- [ ] Look for other DJ hardware researchers who have dumped eMMC

---

## Phase 2 — Filesystem Analysis (after decryption)

Once the key is known, decrypt and mount on Linux:

```bash
# With key file:
sudo cryptsetup luksOpen --key-file aes256.key CDJ3Kv322.UPD cdj3k
sudo mount /dev/mapper/cdj3k /mnt/cdj3k -o ro

# The decrypted image is ISO 9660 — check what's inside:
isoinfo -R -l -i /dev/mapper/cdj3k
```

The ISO contains `src/` scripts (known) and a disk image `CDJ3K-RK3399.iso` which is
the actual rootfs image to be written to eMMC.

### 2a. OS / Platform
- [ ] Identify Linux kernel version from kernel image in ISO
- [ ] Check buildroot/Yocto artifacts for config
- [ ] Find device tree blob (`.dtb`) and decompile: `dtc -I dtb -O dts`
- [ ] Confirm RK3399 from DTB `compatible` string

### 2b. Memory Map (from device tree)
- [ ] DRAM base address and size (RK3399: typically 0x00000000, up to 4GB)
- [ ] Peripheral register ranges: USB, PCIe, I2C, SPI, UART, GPIO banks
- [ ] Audio subsystem: I2S controllers, codec I2C address
- [ ] Display: MIPI DSI or eDP controller, touchscreen I2C
- [ ] Map to RK3399 TRM (Technical Reference Manual — publicly available from Rockchip)

### 2c. Peripheral Inventory (from DTB + kernel modules)
- [ ] Audio codec (I2C address → identify chip)
- [ ] Jog wheel / encoder interface (SPI, dedicated MCU, or GPIO interrupt)
- [ ] USB host controllers (RK3399 has USB3 + USB2 OTG)
- [ ] Ethernet (Pro DJ Link) — GMAC or USB-Ethernet
- [ ] Display panel and touchscreen
- [ ] Storage: eMMC partitions, any SPI NOR flash

### 2d. Software Platform
- [ ] Confirm systemd, find unit files for Pioneer daemons
- [ ] Identify `/home/root/pdj/EP*` binary — disassemble to understand Pro DJ Link protocol
- [ ] Find audio pipeline (ALSA config, possibly JACK or custom DSP)
- [ ] Look for network services (Pro DJ Link uses UDP broadcasts on port 50000/50001)

### 2e. Partition Layout (of `CDJ3K-RK3399.iso` inner image)
- [ ] U-Boot or Rockchip miniloader in first partitions
- [ ] Kernel + DTB partition
- [ ] Root filesystem (ext4 or squashfs)
- [ ] Data / user partition
- [ ] Overlay partition (p8 — writable, where pdj.tar.gz is stored)

---

## Phase 3 — Hardware Correlation

- [ ] Find RK3399 UART0/2 pins on PCB (standard RK3399 debug UART is UART2)
- [ ] Confirm SoC marking on PCB (RK3399 is a large BGA chip)
- [ ] Identify companion chips: audio codec, USB hub, Ethernet PHY, display bridge
- [ ] Cross-reference device tree nodes with PCB ICs

---

## Notes

- **The key file is binary (64 bytes)**, not a text passphrase — `cryptsetup --key-file`
- **RK3399 TRM is public** — once we know peripheral addresses from DTB, full register
  documentation is available from Rockchip's developer site
- **`emmaworley/cdj3k-root`** is the most valuable existing resource; contacting the
  author is likely the fastest path to getting the key
- The Renesas variant (older hardware revision) may be easier to attack if it has
  a less locked-down boot environment
- SSH (OpenSSH + dropbear) is pre-installed on the device — once rooted, full remote
  access is available
