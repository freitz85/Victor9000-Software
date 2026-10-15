# Victor 9000: serial bootstrap, bare-metal floppy access and disk formatting

This document collects the research behind three planned additions to Victor9000-Software. It is
meant to be complete on its own: it covers the hardware facts, the procedures, the design and the
open questions.

The three additions:

1. a bare-metal GCR floppy driver (read / write / format);
2. a **Format disk** function in `igc`;
3. a **serial bootstrap**. A Victor with no working boot disk is started through the ROM serial
   loader. A small client then formats a blank floppy and writes a bootable image onto it, all
   driven from the PC by `igcfs`.

> **Licensing rule.** Several sources below are Victor-copyrighted: the MS-DOS 3.1 BIOS listings,
> `FORMAT.COM` and the reference manuals. They were only *studied*. Everything here is written in our
> own words. Never commit vendor listings, disassembly, or code translated from them line by line.
> MAME and cerealbewt are open source, but here they serve only as references; check their licenses
> before reusing any code.
> vtg_image_util states no license, so it is reference only and no code may be reused.

---

## 1. Sources

| Source | Used for |
|---|---|
| [FozzTexx/Victor9000-ROM-source](https://github.com/FozzTexx/Victor9000-ROM-source) (`bt1base.asm`, `bt1info.doc`, `bt1fddvc.asm`, `bt1fdlbl.str`) | ROM serial loader, ROM floppy read path, boot label |
| [FozzTexx/cerealbewt](https://github.com/FozzTexx/cerealbewt) (`bootstrap.py`, README) | Host side of the ROM serial boot, cable |
| [ADTPro/adtpro](https://github.com/ADTPro/adtpro) | Model for the user workflow: bootstrap a client, then transfer images |
| [IntergalacticMicrosystems/vtg_image_util](https://github.com/IntergalacticMicrosystems/vtg_image_util) `format_docs/` | Victor MS-DOS filesystem layout |
| MAME `victor9k_fdc.cpp`, `victor9k_dsk.cpp` | FDC behaviour (VIAs, PCR modes, `WAIT` handshake), image geometry |
| Victor 9000 Hardware Reference Manual, disk chapter (sector format fig. 42, track format table 7) | Sector and track format |
| Victor MS-DOS 3.1 BIOS listings: `FSYNC`, `FDDV9000`, `FDIOREGS`, `SPEED`, `DIOV9000`, `DLABEL` | Write path, drive control, speed MCU, label, logical mapping, error codes |
| Victor `FORMAT.COM` (studied as a binary) | Track measurement, track write, verify |
| This repository: `igc`, `igcfs`, `viclibc2` | Serial driver, packet layer, image protocol |

---

## 2. What exists already in this repository

- **igc**
  - DOS two-pane file manager.
  - Reads and writes disk images through INT 25h/26h, so it needs DOS and an already formatted disk.
  - Image formats: `.IMG` and `.RLE`. The `.RLE` header is the magic `V9KRLE\x1a\x01` (the last byte
    is the format version), then a u32 raw size, then a u32 CRC-32.
- **igcfs** (`igc/tools/serialfs/`)
  - Python PC server using pyserial.
  - Supports MAME ptys and PyInstaller builds.
  - Disk-image exchange in `serialfs_proto.h`:

    | Command | Code | Notes |
    |---|---|---|
    | `SFS_CMD_IMG_OPEN` | `30h` | |
    | `SFS_CMD_IMG_INFO` | `31h` | |
    | `SFS_CMD_IMG_GET` | `32h` | |
    | `SFS_CMD_IMG_TRACK` | `33h` | 1024-byte fragments (`SFS_IMG_FRAG`), 7-byte header, RLE flag |
    | `SFS_CMD_IMG_CLOSE` | `34h` | carries the CRC-32 |
- **viclibc2**
  - Polled µPD7201/8253 serial driver (`serial_hw.c`).
  - Packet layer: SYNC `7Eh`, escape `7Dh`, CRC-16, ACK/NAK, 1-bit sequence number.
  - The FTX file protocol (`ftx_protocol.h`, commands `10h`–`23h`), RLE and CRC-32.
  - The serial and packet layers touch the hardware directly and use loop-count timeouts. They do
    **not** need DOS; only `ftx_dosfile.c` and `filetrx.c` use INT 21h.
  - Proven at **38400 8N1 over 3 wires**.
- **Command-number overlap:** `SFS_CMD_DELETE`/`SFS_CMD_MKDIR` (`1Ah`/`1Bh`) collide with
  `FTX_CMD_QUERY`/`FTX_CMD_HAVE`. Raise this upstream before assigning new numbers. The range
  `40h`–`4Fh` is currently unused by any protocol header.
- `igc` and `igcfs` must be changed together.

What is missing: formatting (all paths go through DOS on an already formatted disk), and any way to
run without a boot disk.

---

## 3. ROM serial boot

Source: `bt1base.asm` (serial download routine, around lines 595–725) and `bt1info.doc`.

### Trigger condition
- The routine is checked after the memory test.
- It is entered only if serial port A sees **DSR and RI asserted** (DCD is also wired in the
  reference cable).

### Transfer
- Exactly **512 bytes** are received at **1200 baud, 8 data bits, even parity, 1 stop bit (8E1)**.
- **Every byte is echoed** back to the host.
- The bytes are stored at **`0000:4000`**, and control passes to them via **`INT 255`**.
- Any receive error (parity, framing, overrun) aborts the download.

### Cable

| Host signal | Victor port A signal (DB-25 pin) |
|---|---|
| DTR | DSR (6), RI (22), DCD (8) |
| CTS | RTS (4) |
| TxD | RxD |
| RxD | TxD |
| GND | GND (7) |

The plain 3-wire igc cable **cannot** trigger the serial boot. Bootstrap needs this cable, or an
adapter that adds the DTR → DSR/RI/DCD and RTS → CTS links.

### Host procedure (cerealbewt `bootstrap.py`)
1. Open the port at 1200 8E1 and raise DTR.
2. Wait for CTS.
3. Send the 512 bytes one at a time, checking each echo.
4. If the first byte is not echoed, retry. The ROM may be re-arming: CTS drops and rises again.

### Serial hardware (for the stage-1 loader and the client)
- **µPD7201**: segment `E004h`. Channel A data at offset 0, channel B data at 1, channel A control at
  2, channel B control at 3.
- **8253 baud-rate generator**: segment `E002h`, clocked at 1.2288 MHz. The rate is 1.2288 MHz / 16 /
  divisor; 1200 baud uses divisor `41h`.
- **Clock-select VIA**: segment `E804h`.
- `viclibc2/serial_hw.c` already programs all three. The stage-1 loader only needs a minimal copy:
  switch to 8N1 and set the working baud rate.

---

## 4. Disk geometry

- GCR recording, constant linear velocity in **9 speed zones**.
- 512-byte sectors, 80 tracks per side, 96 tpi, about 8000 bpi.
- No index sensor.
- **SS = 1224 sectors (626,688 bytes); DS = 2391 sectors (1,224,192 bytes).** Images hold all side 0
  tracks first, then all side 1 tracks.

### Sectors per track, as (sectors, tracks)
- Side 0: (19,4) (18,12) (17,11) (16,11) (15,10) (14,12) (13,11) (12,9)
- Side 1: (18,8) (17,11) (16,11) (15,10) (14,12) (13,11) (12,12) (11,5)

### Zone lookup (BIOS)
- Zone upper track bounds: 3, 15, 26, 37, 47, 59, 70, 82, 87 for zones 0–8.
- Sectors per track for zones 0–8: 19, 18, 17, 16, 15, 14, 13, 12, 11.
- Side 1: add 8 to the track number before the lookup. So side 1 starts in zone 1 and ends in zone 8.
- Rotation period: from 237.9 ms (zone 0) down to 144.0 ms (zone 8).

### Logical-to-physical mapping
- MS-DOS 3.1 calls the mapping with interleave 0, so **image sector order equals physical order**.
  The interleave nibble in the label is read but not used for MS-DOS disks.
- Side 1 begins at logical sector 1224. Its physical track ID is `80h | track`, and it uses the zone
  of track + 8.
- BIOS multi-sector writes go out in the order 0, 2, 4, …, 1, 3, … within a track, followed by one
  verify pass. This is only a speed trick; it is not needed for correctness.

---

## 5. Disk label (sector 0)

The layout follows the MS-DOS 3.1 BIOS `DLABEL` definition and the ROM's `bt1fdlbl.str`. The BIOS
reads the first 128 bytes.

| Offset | Size | Field |
|---|---|---|
| 0 | 2 | label ID; `FF00h` = system (bootable) disk |
| 2 | 2 | load segment of the boot program |
| 4 | 2 | boot program length in paragraphs |
| 6 | 4 | entry point, IP:CS |
| 10 | 8 | part ID |
| 18 | 8 | serial number |
| 26 | 2 | sector size (512) |
| 28 | 2 | data start (often 0 on real disks) |
| 30 | 2 | boot start: first sector of the boot program |
| 32 | 2 | flags: bit 0 = double-sided; bits 15–12 = interleave |
| 34 | 1 | disk type: `00h` CP/M, `01h` MS-DOS, `10h` MS-DOS 3.1 |
| 35 | 3 | reserved |
| 38 | 18 | speed table |
| 56 | 15 | zone table |
| 71 | 15 | sectors-per-track table |
| 86 | … | further fields (not needed for formatting) |

### Behaviour
- **ROM boot:** if the label ID is `FF00h`, the ROM loads `paras` paragraphs starting at `boot_start`
  to the load segment, then jumps to the entry point via `INT 255`.
- **MS-DOS 3.1 BIOS:**
  - rejects the disk with `F5h` (bad label) unless the disk type is `10h`;
  - returns `F1h` if the label says double-sided but the drive's DS strap says single-sided;
  - keeps a CRC of the label to detect disk changes. A changed label makes the drive read-only until
    the disk is checked again.
- **Open question:** whether older BIOS versions accept other disk-type values.

---

## 6. Victor MS-DOS filesystem

Source: vtg_image_util `format_docs/`, which were checked there against real disks.

- There is **no IBM BPB**; the Victor label replaces it.
- FAT12 with **4 sectors per cluster** (2048 bytes).
- 2 FAT copies.
- 128-entry root directory (8 sectors).

| | Label | FAT 1 | FAT 2 | Root directory | Data |
|---|---|---|---|---|---|
| SS | 0 | 1 | 2 | 3–10 | 11 → |
| DS | 0 | 1–2 | 3–4 | 5–12 | 13 → |

Do **not** use vtg_image_util's blank-image creator as a reference. It:
- puts a JMP over the label at offset 0;
- leaves out the speed, zone and sectors-per-track tables;
- uses 2448 sectors for DS instead of 2391;
- counts clusters as sectors.

Its reader is still useful for checking our formatted disks and images.

Still to be confirmed from a disk freshly formatted by Victor `FORMAT`:
- the label tables;
- the IDs;
- `data_start`;
- the FAT media byte;
- the exact disk type value.

---

## 7. Floppy controller hardware

### VIAs and data port
- The drive interface is three 6522 VIAs in segment `E000h`, at offsets `80A0h`, `80C0h` and `80E0h`.
  Seen from segment `E800h`, the same VIAs are at offsets `A0h`, `C0h` and `E0h`.
- They are called **VIA-A**, **VIA-C** and **VIA-E** here.
- Register offsets are the standard 6522 ones:

  | Offset | Register |
  |---|---|
  | +0 | ORB |
  | +1 | ORA |
  | +2 | DDRB |
  | +3 | DDRA |
  | +4 … +9 | timers |
  | +0Bh | ACR |
  | +0Ch | PCR |
  | +0Dh | IFR |
  | +0Eh | IER |

- **Data port:** write to `80E0h`, read from `80E1h`.
- One GCR byte takes about **21.3 µs** (10 bit cells of about 2.13 µs).

### Byte handshake
- The byte-ready strobe is wired to the 8088 **`TEST`** pin.
- `WAIT` (`9Bh`) blocks until the controller needs or supplies the next byte.
- Every byte transfer is therefore a `WAIT` plus a `MOV` to or from the data port, with interrupts
  off. No polling is needed.

### Write mode: VIA-A PCR (`80ACh`)
- `0Eh` = sync mode. The bytes written (`FFh`) become sync marks.
- `0Ch` = normal GCR data mode.

### Write and erase current: VIA-C PCR (`80CCh`)

| Value | Write current | Erase current |
|---|---|---|
| `CCh` | on | on |
| `ECh` | on | off |
| `CEh` | off | on |
| `EEh` | off | off |

### Status
- `80C1h` bit 7 = sync detected (active low).
- `80CDh` (VIA-C IFR) bit 1 = GCR decode error.
- A sync mark of **6 or more bytes** raises the SYNC interrupt on the 8259 (segment `E000h`, offsets
  0 and 1). The 5-byte data sync is too short to raise it and has to be polled.

---

## 8. Drive control

Sources: the BIOS `FDDV9000`, `FDIOREGS` and `SPEED` modules.

### Per-drive control bytes (VIA-A)
- Drive 0 uses VIA-A **port A** (`E80A1h`). Drive 1 uses VIA-A **port B** (`E80A0h`).
- **Low nibble** = speed-zone number passed to the speed MCU. `0Fh` turns the motor off.
- **High nibble** = the four stepper phases.

### Stepping
1. Enable the drive's stepper by clearing its bit in VIA-C ORB (active low): bit 6 for drive 0,
   bit 7 for drive 1. Set the bit again to switch the stepper off when idle.
2. **Step in** (towards the spindle): rotate the phase pattern by one position.
   - Bits 7:6 move down to bits 5:4.
   - Bits 5:4, inverted, become bits 7:6.
3. **Step out** is the reverse rotation.
4. **Track 0** is reached when the phase pattern is `A0h` and the drive's track-0 sense bit in VIA-C
   ORA is low: bit 1 for drive 0, bit 3 for drive 1.
5. **Restore:** load phase `A0h`, assume track 100, then step out until the track-0 condition holds.
6. **Timing:** 3 ms per step, then 30 ms of head settle.

### Motor and speed
1. Write the zone number into the low nibble of the drive's control byte.
2. Wait for the drive's motor-ready bit in VIA-C ORB: bit 0 for drive 0, bit 1 for drive 1. The BIOS
   polls every 50 ms and gives up after 1.5 s.

### Select and status (VIA-C)

| Signal | Register | Bit |
|---|---|---|
| Drive select | ORA | bit 5 (0 or 1) |
| Head select | ORA | bit 4 (1 = side 1) |
| Activity LED | ORA | bit 0 (drive 0), bit 2 (drive 1) |
| Write protect | ORA | bit 6 (set = protected; selected drive) |
| Door open | ORB | bit 4 (drive 0), bit 3 (drive 1) |
| Double-sided strap | ORB | bit 5 (0 = side 2 available) |
| Motor ready | ORB | bit 0 (drive 0), bit 1 (drive 1) |
| Stepper enable | ORB | bit 6 (drive 0), bit 7 (drive 1); active low |

**Timers:** the VIA T1 timers count 1 µs ticks. The BIOS uses VIA-A T1 for drive 0 and VIA-C T1 for
drive 1.

### Initialisation done by the BIOS
1. Reset and load the speed MCU (next section).
2. Set up the VIAs:

   | VIA | ACR | PCR | IER |
   |---|---|---|---|
   | VIA-A | 0 | `0Ch` | door-sense interrupts enabled |
   | VIA-C | 0 | `EFh` | all off |
   | VIA-E | 1 (port A input latching) | `66h` | – |

### Speed MCU (8048) load protocol
The handshake uses three VIA-C ORB lines:
- bit 0 = `RDY0`, data strobe;
- bit 1 = `RDY1`, acknowledge;
- bit 2 = MCU reset.

Data goes out as nibbles: the low nibble on the drive-0 motor bits, the high nibble on the drive-1
motor bits.

1. Assert reset and make `RDY0`/`RDY1` outputs. Make the motor nibbles of both VIA-A ports inputs.
2. Release reset. Wait until the MCU pulls all motor nibbles low, then raise `RDY0`/`RDY1`.
3. Wait until all motor nibbles are high. Then make the motor nibbles outputs and `RDY1` an input.
4. Send 19 bytes: a count byte of 15, then the 18-byte speed table. For each byte:
   1. Place the nibbles.
   2. Toggle `RDY0`.
   3. Wait for `RDY1` to go low.
   4. Toggle `RDY0` back.
5. Finish:
   1. Set both motor nibbles to `0Fh` (motors off).
   2. Toggle `RDY0`.
   3. Wait for the acknowledge.
   4. Turn `RDY0` back into an input.

**Default speed table** of the MS-DOS 3.1 BIOS. The first three bytes are a header; the rest are
per-zone speed values:

```
04 11 01  3E 49 53 5D 67 71 7B 86  8A 8A 8A 8A 8A 8A 8A
```

The MS-DOS 3.1 BIOS always loads this table at start-up and ignores the label's `speed` field.

A bare-metal client started through the ROM serial boot should **load the speed MCU itself**. It is
not known whether the ROM leaves it in a usable state.

---

## 9. Sector layout

| Field | Bytes | Content |
|---|---|---|
| Header sync | 15 | `FFh` written in sync mode |
| Header ID | 1 | `07h` |
| Track ID | 1 | track 0–79; **bit 7 set for side 1** |
| Sector ID | 1 | 0 … spt−1 |
| Header checksum | 1 | (track ID + sector ID) mod 256 |
| Gap 1 | 5 | `00h` |
| Data sync | 5 | `FFh` in sync mode |
| Data ID | 1 | `08h` |
| Data | 512 | |
| Data checksum | 2 | 16-bit sum of the 512 data bytes, low byte first |
| Gap 2 | n | `00h`; *n* is computed per zone during the format |

Sector IDs are written in **physical order 0, 1, 2, …** with no physical interleave.

## 10. Updating one sector (BIOS write path)

1. **Find the header.** Wait for the SYNC interrupt, then check the `07h` ID, track, sector, checksum
   and the GCR error bit.
2. Let a few gap-1 bytes pass, so the new data field starts at a fixed distance from the header.
3. Write one `FFh`, switch to sync mode, turn on write current, then write 5 sync bytes.
4. Switch to data mode and write `08h` and the 512 data bytes. Turn on erase current after about
   20 bytes.
5. Write the 2 checksum bytes and 2 bytes of `00h`, then turn write current off.
6. About 620 µs (≈ 31 byte times) later, turn erase current off (trim erase tail).
7. Verify by reading the sector back and comparing.

## 11. Formatting one track (FORMAT.COM procedure)

There is no index sensor. A track is written as one continuous stream lasting a little more than one
revolution: a long lead-in, then all sectors. The end of the stream overwrites part of the lead-in,
which is harmless because the lead-in is only filler.

### 11.1 Measure the track
Do this once per speed zone, whenever the zone changes, after the motor speed has been set for the
zone.

1. With interrupts off, turn on write and erase current and switch to data mode.
2. Write about 15,000 filler bytes of `5Eh` to settle the drive.
3. Write 200 blocks, each made of 4 sync bytes and 200 bytes of `5Eh`. The short syncs keep the PLL
   locked but are too short to raise an interrupt.
4. Write one 15-byte sync mark, then turn write and erase current off.
5. Wait briefly. Hook the SYNC interrupt and count byte-ready strobes from one SYNC interrupt to the
   next, which is one revolution.
   - The counter starts at a bias of `73h` (115) to make up for the detection latency.
   - If no second interrupt arrives within about 20,000 bytes, the measurement fails.
6. Compute gap 2 from the measured `bytes_per_rev`:

   ```
   usable = bytes_per_rev - bytes_per_rev / 100      (1 % speed margin)
   gap    = usable / spt - 546                        (546 = fixed bytes per sector)
   ```

   `gap` must be between 21 and 121; otherwise the format fails ("drive speed out of range"). The
   value actually written as gap 2 is `gap + 2`.

### 11.2 Write the track
1. With interrupts off, turn on write and erase current (`CCh`) and switch to data mode.
2. Write a 2,000-byte lead-in of `00h`.
3. For each sector *s* = 0 … spt−1, write:
   - 15 sync bytes;
   - `07`, track, *s*, checksum;
   - 5 × `00`;
   - 5 sync bytes;
   - `08`, 512 filler bytes, 2 checksum bytes;
   - gap 2 × `00`.

   `FORMAT.COM` writes whatever is in its buffer as filler. Our driver should use a fixed pattern
   (for example `E5h`) or the final filesystem contents.
4. Turn write and erase current off (`EEh`).

### 11.3 Verify the track
1. After a short delay, read every sector in the order its header comes by.
2. Check the header ID, track, sector and header checksum, then the five `00` gap bytes, then the GCR
   error bit.
3. Wait for the data sync. Check `08h`, the data against the source, the checksum, the gap-2 bytes
   and the GCR error bit.
4. Retry the whole track up to 10 times before reporting an error.

### 11.4 Overall order
1. Format side 0, tracks 0–79.
2. If the disk is double-sided, format side 1, tracks 0–79, with track ID = `80h | track`.
3. Step the head after each track.
4. Finally write the label, both FATs and the root directory (section 6).

### 11.5 Still to confirm on real disks
- Real gap and lead-in sizes.
- What `FORMAT` puts in the label.

Applesauce flux captures of freshly formatted disks will settle both (section 16).

---

## 12. Error codes (BIOS floppy status)

| Code | Meaning | Code | Meaning |
|---|---|---|---|
| `11h` | bad sync | `F1h` | side 2 not available |
| `12h` | bad header ID | `F2h` | step error |
| `13h` | header checksum | `F3h` | buffer lost |
| `14h` | header GCR error | `F4h` | read-only (label changed) |
| `15h` | track mismatch | `F5h` | bad label |
| `16h` | sector mismatch | `F6h` | no jobs |
| `17h` | bad job code | `F7h` | bad track |
| `21h` | bad data ID | `F8h` | bad drive |
| `22h` | data checksum | `F9h` | bad job |
| `23h` | data GCR error | `FAh` | door open |
| `24h` | sync timeout | `FBh` | motor timeout |
| `31h` | verify: ID | `FCh` | write protected |
| `32h` | verify: GCR | `FDh` | seek error |
| `34h` | verify: compare | `FEh` | job timeout |
| `38h` | verify: checksum | `FFh` | no sync |

`FORMAT.COM` uses its own verify codes, `41h`–`4Dh`.

The new driver should reuse these values, so that results can be compared with DOS and the BIOS and
igcfs can log them.

---

## 13. Proposed architecture

```
viclibc2/
  floppy/                    NEW  bare-metal GCR driver (Open Watcom C + small asm byte loops)
    include/v9kfdc.h              fdc_init, speed-MCU load, seek, fdc_read_sector/track,
                                  fdc_write_sector/track, fdc_format_track, error codes
    src/fdc.c                     drive control: select, motor, stepping, zones
    src/fdc_gcr.asm               WAIT-based byte loops, interrupts off
    test/fdctest.c                DOS test harness; compares results with INT 25h
igc/
  src/format.c               NEW  "Format disk" in the F6 menu (SS/DS); lays down the tracks with
                                  v9kfdc, then writes label + empty FATs + root directory
  tools/serialfs/
    igcfs.py                 EXT  PC-initiated commands, --bootstrap mode
    v9kboot.py               NEW  stage 0 upload (1200 8E1, DTR/CTS, per-byte echo) + client upload
    v9kimage.py              EXT  build an empty Victor DOS disk image / label (same layout as format)
bootstrap/                   NEW
  boot1.asm                       512-byte stage-1 loader, org 0000:4000
  client/                         bare-metal client: own startup (no DOS, no C runtime), viclibc2
                                  serial + packet + RLE/CRC-32, v9kfdc, command loop
```

### Driver design notes
- The `WAIT` handshake keeps the byte timing in hardware. Even so, the inner byte loops must be small
  assembly routines, called from C and run with interrupts disabled. Plain C is too slow and fragile
  there.
- Track measurement (section 11.1) is essential. A fixed gap only works on drives running at nominal
  speed.
- The driver must not call DOS or the BIOS. It is used both under DOS (by igc) and bare-metal (by the
  client).
- Under DOS, igc must refuse to format the drive it runs from. It must also make DOS re-read the
  label afterwards, because of the label-change read-only state.

---

## 14. Protocol additions and bootstrap flow

### New PC → Victor commands
These ride on the existing packet layer. The command range, e.g. `40h`–`4Fh`, must be agreed with
upstream first.

| Command | Purpose |
|---|---|
| `HELLO` / `INFO` | client version, RAM, drives, SS/DS capability |
| `DISK_FORMAT(drive, sides)` | format with per-track progress and status |
| `DISK_READ(drive)` | read a disk; reuses the `IMG_TRACK` fragment format and the CRC-32 close |
| `DISK_WRITE(drive)` | write a disk; reuses the `IMG_TRACK` fragment format and the CRC-32 close |
| `DISK_VERIFY(drive)` | read back and compare |
| `REBOOT` | restart the machine |

Every track reports a status using the codes from section 12. This lets igcfs retry and log bad
sectors.

### Bootstrap flow
1. **Stage 0 (PC):** `igcfs --bootstrap --dev /dev/ttyUSB0 …`
   1. Open the port at 1200 8E1 and raise DTR.
   2. Wait for CTS.
   3. Send `boot1.bin` (512 bytes) with a per-byte echo check, retrying as described in section 3.
2. **Stage 1 (`boot1`, at 0000:4000):**
   1. Switch the 7201 to 8N1 and the 8253 to the working baud rate.
   2. Show status in video RAM.
   3. Receive the client (length + CRC), relocate it to a safe segment, and jump to it.
3. **Client:**
   1. Load the speed MCU and initialise the VIAs.
   2. Send `HELLO`.
   3. igcfs runs the requested job, for example
      `igcfs --bootstrap --format B --write-image MSDOS.IMG --verify --reboot`.
4. **Optional rescue disk:** build a bootable floppy image that contains the client (valid label,
   load segment and entry point). MAME can then boot it without the ROM serial path, and users can
   keep a rescue disk.

---

## 15. Work plan

| ID | Task | Depends on |
|----|------|------------|
| fork-setup | Toolchain (Open Watcom v2, NASM, MAME); build and run the existing tests; igcfs against MAME | – |
| upstream-issue | Agree the approach, command numbers and directory layout with upstream | – |
| research-fdc | FDC write/format research (this document); confirm with Applesauce captures | – |
| fdc-read | Driver: select, motor, speed MCU, seek, sector/track read with error codes; check against INT 25h | fork-setup, research-fdc |
| fdc-write | Driver: sector/track write + read-back verify | fdc-read |
| fdc-format | Driver: per-zone measure + track format + verify | fdc-write |
| dos-fs-layout | Empty Victor DOS filesystem (label, FATs, root directory) in C and `v9kimage.py`; check with vtg_image_util, igc, Victor DOS | fork-setup |
| igc-format | F6 "Format disk" (drive, SS/DS, confirm, progress, verify); docs | fdc-format, dos-fs-layout |
| boot1 | 512-byte stage-1 loader | fork-setup |
| host-bootstrap | `v9kboot.py`, `igcfs --bootstrap` | boot1 |
| client-core | Bare-metal startup, serial/packet, command loop, `HELLO`/`INFO`, `REBOOT`, status in video RAM | fork-setup, boot1 |
| igcfs-pc-commands | Format/read/write/verify commands in igcfs + client, retries, bad-sector log | client-core, fdc-format, upstream-issue, host-bootstrap |
| rescue-disk | Bootable client image and rescue workflow | igcfs-pc-commands, dos-fs-layout |
| mame-testing | MAME tests for format, read/write and the rescue image; stage 0 on real hardware | igc-format, rescue-disk |
| docs-upstream | Cable diagram (bootstrap vs. 3-wire), USB-adapter 8E1 notes, usage. Upstream PRs in this order: driver → igc format → bootstrap | mame-testing |

### Risks
- **GCR write/format timing.** The procedure is known, but the byte loops must be exact. Test in MAME
  first (it emulates the same VIA, PCR and `WAIT` behaviour), then on real hardware with scratch
  disks.
- **Upstream churn.** The repository is very active, and igc and igcfs must stay in sync. Use small
  PRs, agree the protocol numbers first, and rebase often.
- **Client without DOS or C runtime.** It needs its own startup, `-zl` (no default libraries) and no
  library calls. Watch the size the boot1 loader has to handle.
- **MAME and the ROM serial boot.** MAME may not reproduce the DSR/RI/DCD gating. Test stage 0 on real
  hardware, and everything after it in MAME via the rescue image.
- **USB-serial adapters.** Some mishandle 8E1 or the control lines. Document adapters known to work.

---

## 16. Open questions and data still needed

1. **Applesauce flux captures** (`.a2r` plus the decoded sector image) of:
   - a single-sided blank disk freshly formatted by Victor `FORMAT`;
   - a double-sided blank disk freshly formatted by Victor `FORMAT`;
   - a bootable MS-DOS system disk.

   These will confirm the real gap-2 and lead-in sizes, the label contents (speed, zone and
   sectors-per-track tables, IDs, `data_start`), the FAT media byte and the disk type.
2. **Which `FORMAT` version** (and DOS/BIOS version) produced the reference disks. `V20FORM.EXE` also
   exists and has not been examined.
3. **Technical Reference Manual pages** on:
   - the label speed, zone and sectors-per-track tables;
   - the units of the speed values;
   - the boot ROM and serial boot.
4. Whether **older BIOS versions** accept disk types other than `10h`.
5. Whether the ROM leaves the **speed MCU** loaded after a serial boot. The plan assumes it does not,
   so the client loads it.
