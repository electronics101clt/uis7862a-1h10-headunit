# Board identity, debug pads, and the recovery route

Added 2026-10-21, from photographs of the large screen's board taken after it
boot-looped. Identity below is **proven** — read off the board and its QR label.
The recovery procedures are **evidence**: documented by the TS10 community,
not yet performed on this unit.

## Identity

QR label on the board, transcribed:

```
YM*E1208*12.AW4C-0432-47*TS18/TS10M-A125M-V16/V1.0/AW4B-0432-4A/TS10S.2.1_20240928.091908*00499
```

Silver label, printed:

```
E1208C100 / V20241119003 / AW4C-0432-47
TS10S 432 1FT / MIPI/AHD/770 / DSP32107/QD / YA00499
```

| Field | Value |
|---|---|
| Carrier board | **AW4C-0432-47** |
| Core module (SoM) | **AW4B-0432-4A** |
| Panel/base board | **Toparea TS18/TS10M-A125M-V16**, rev V1.0, D/C `M31C` |
| PCB silkscreen | `YM-E-1205-2` |
| Firmware | **TS10S.2.1_20240928.091908** |
| Unit serial | `00499` / `YA00499`, build date `V20241119003` |
| Display / camera | MIPI panel, AHD camera input |
| Audio | DSP32107 + YD1821B amplifier |
| Bluetooth | Realtek RTL8761BTV |

This is a **TS10S** (UIS7862A). The silkscreen reads `TS18/TS10M` because the
board is shared across both product lines — the *firmware* string is what
identifies it. **Flashing TS18 firmware onto this board is an unrecoverable
brick.** Match `TS10S` exactly.

## Debug pads

Four round test points in a vertical column, below the CVBS connector, near the
RTL8761BTV. Silkscreened top to bottom:

```
GND
KEY0
TX     (silkscreen arrow points at the pad)
RX
```

`1AM`-marked SOT-23 parts sit in series on TX/RX. **Measure the idle voltage on
TX before connecting anything** — this board carries a `1V8` rail and Unisoc
consoles are commonly 1.8V, not 3.3V. 115200 8N1.

A separate USB breakout sits bottom-right: flat pads `DP · DN · 5V · GND`, plus
test points `GND`, `VCI1`, `DP`, `DK`. Note this is a *different* tap from the
external port that `USB-DEVICE-MODE.md` found to be host-fixed.

Also present, on the panel board near the KEY FFC: pads `KEY1`, `KEY2`, `RST`,
`ENC_A`, `ENC_C`, `5V`. **`KEY1` is the pad the force-flash procedure below
needs.**

## Recovery: force flash from USB, no PC required

The TS10 community's standard unbrick, for a unit stuck on the logo or looping.
It needs no adb, no UART, and no SPD tool.

1. Bench the unit — out of the dash, on a 12V supply.
2. Format a USB stick **FAT32**, extract the firmware to its **root**.
3. Plug the stick into **USB1**.
4. **Short KEY1 to ground.**
5. Power on, or restart with the reset pinhole.
6. Watch for the USB indicator to blink — that means it took.
7. **Release KEY1** once the update has started.
8. Wait ~15 minutes. Do not interrupt.
9. Verify under Settings → About.

A reported variant holds **VOL −** on the steering wheel instead of grounding
KEY1, restarting via the reset pinhole.

Reported failure mode: a unit with a corrupted bootloader will not start the
update at all under this method. If that happens, SPD/BROM (VID 1782) with
ResearchDownload and a matching `.pac` is the fallback — see the post-mortem.

## Firmware for this exact unit

Our build is `TS10S.2.1_20240928.091908`, and it is published:

- Stock: `https://lnx.pausacafeone.it/car_tablet/topway_ts10_firmware/TS10S.2.1_OS-EN/TS10S.2.1_20240928.091908_OS-EN.zip`
- Pre-rooted: same path, `..._OS-EN_root.zip`

Index: https://lnx.pausacafeone.it/wordpress/2022/12/09/topway-ts10-ts18-firmware/
(archive password reported as `TopwayTS10`). The timestamp matches this unit's
build exactly, so this is a same-version reflash rather than an upgrade.

Note the pre-rooted image would settle `ROOT-AND-SYSTEM-APP.md`'s open question
in passing — but recovery first, one variable at a time.

## Sources

- https://annssolutions.com/topway-ts10-uis7862a-ts18-uis8581a-firmware-update/
- https://xdaforums.com/t/new-topway-ts10-uis7862-6gb-ram-128gb-head-unit-q-as.4227947/
- https://xdaforums.com/t/firmware-for-ts10-uis7862-s-ums512_1h10_natv.4665540/
- https://lnx.pausacafeone.it/wordpress/2022/12/09/topway-ts10-ts18-firmware/

## Complete board inventory

Everything legible across the 13 photographs of 2026-10-20 23:45–23:46. This is
the **panel/carrier board** (`AW4C-0432-47` / `Toparea TS18/TS10M-A125M-V16`),
photographed from its component side in the black chassis frame. The SoM
(`AW4B-0432-4A`) carrying the SoC and eMMC is **not in any of these photos** —
no Unisoc marking, no memory package is visible anywhere on this side.

Marked **(?)** where the silkscreen or package marking is partly obscured and
the reading is inferred rather than certain.

### Silkscreen identity

| Text | Where |
|---|---|
| `YM-E-1205-2` | mid-board, large |
| `AW4C-0432-47` | directly above the board code |
| `TS18/TS10M-A125M-V16` | centre, beside the Toparea logo |
| `Toparea` + logo | centre |
| `BW36` | beside the board name |
| `D/C:M31C` | date code, beside the board name |

### Test points and pad groups

| Label group | Pads, in silkscreen order | Purpose |
|---|---|---|
| Top edge header | `GND · 3.3V · SDA · CLK · RST` | 5 plated holes — I²C/programming header |
| **Debug** | `GND · KEY0 · TX · RX` | 4 round test points, arrow on TX. **The console.** |
| Backlight | `12V · PWM · ADJ · GND` | 4 square pads, left of the YD1821B |
| Mic/reset group | `GND · 5V · RST · MIC- · MIC+` | 5 square pads, left edge |
| KEY group | `MIC+ · MIC- · ENC_A · ENC_C · RST · KEY2 · KEY1 · 5V` | round points beside the KEY FFC. **`KEY1` is the force-flash pad.** |
| Touch panel | `5V · SCL · SDA · EN · RST` | 5 square pads, lower centre |
| TP connector | `RST · VDD · INT · SDA · SCL` | round points beside the TP FFC |
| Panel control | `1V8 · RST · STB` | 3 round points beside the lower TFT FFC |
| USB breakout | `DP · DN · 5V · GND` (square pads) + `GND · VCI1 · DP · DK` (round points) | lower right |
| Power points | `+12V`, `GND`, `5V`, `LED+`, `LED-` | scattered |

### Connectors

| Marking | Type | Notes |
|---|---|---|
| `KEY` | FFC, ~12-pin | front panel keys / encoder |
| `TFT` | FFC, fine-pitch, ×2 | panel; one much wider than the other |
| `TP` | FFC | touch panel |
| `SD` | FFC | SD / card reader |
| `CVBS` | FFC, small | composite video in |
| (unmarked) | 2×8 pin header | top right, board-to-board |
| (unmarked) | 2×16 pin header | right edge, board-to-board |
| (unmarked) | 2×2 and 2×7 blocks | right edge, with teardrop pads |
| (unmarked) | 4 large plated holes | bottom left, below the QR label |

### Identified components

| Marking | Package | Identification |
|---|---|---|
| `YD1821B` / `AUDn.V` | SOIC-20 | audio power amplifier |
| `REALTEK 8761BTV` / `031AAC G01` | QFN | Bluetooth controller (RTL8761BTV) |
| `40.000` | SMD can | 40 MHz crystal, beside the RTL8761BTV |
| `P88C 2431` | SOT-89(?) | TL431-class shunt voltage reference |
| `19N33` | SOT-23 | MOSFET |
| `LPS D0E62` | SOT-23-5 | regulator (?) |
| `GEX` / `K28b` | SMB | TVS / rectifier diode |
| `K24` | SMA | diode |
| `JF` | 1206 | fuse (two present) |
| `100`, `150`, `0`, `34KE` | chip resistors | `150` sits on the USB breakout |
| `1AM`, `A7`, `MOY`, `2A`, `AOC` | SOT-23 | small-signal transistors/diodes |
| (blank) | QFP-64(?) | large unmarked QFP, centre — likely the panel MCU |
| (blank) | QFN-32/40 | two, near the TFT connectors — solder-flooded tops |
| (blank) | gold shield can, 3×3 castellated pads | lower centre |

### What the photos do not show

- **No SoC and no eMMC.** The `AW4B-0432-4A` module is elsewhere — other side of
  this board, or a separate stack. Finding the eMMC package would matter only if
  recovery ever came down to a clip-on read.
- **No reset pinhole.** The force-flash procedure wants one; its location is
  still unknown on this chassis.
- **No external USB port** in frame, only the breakout pads above.
- The board is in its frame throughout, so the outer edges and the reverse are
  unphotographed.

## What each pad group is for

Function assignments are read from the silkscreen plus standard practice for
this class of board. The four in the first table are the ones that matter for
getting back in.

### The ones that matter

| Pads | What it is | Why we care |
|---|---|---|
| `GND · KEY0 · TX · RX` | Debug UART, with a boot-mode key line in the same column | The console. Says whether `system_server` is crash-looping or the bootloader never hands off — which is the question the post-mortem could not answer. `KEY0` grounded at power-on pushes the chip down an alternate boot path. |
| `KEY1` (KEY group) | Front-panel button input #1 | **The force-flash trigger.** Ground at power-on per the USB update procedure above. |
| `RST` (several locations) | Reset | A reset that does not need the chassis pinhole. The one in the KEY group is the front-panel reset. |
| `DP · DN · 5V · GND` | A USB port broken out to pads | A second tap on a USB bus, independent of the external port that `USB-DEVICE-MODE.md` found host-fixed. |

### Display and backlight

- `12V · PWM · ADJ · GND` — backlight driver input: boost supply, digital (PWM)
  dimming, analog (ADJ) dimming, ground.
- `LED+ · LED-` — driver output to the panel's LED string.
- `1V8 · RST · STB` — panel bridge: 1.8V rail test point, reset, standby.
  **This is the board's evidence that 1.8V logic is present** — the reason to
  measure before connecting anything to TX.
- `TFT` ×2 — panel data connectors.

### Touch

- `5V · SCL · SDA · EN · RST` — capacitive touch controller: power, I²C pair,
  enable, reset.
- At the `TP` connector: `RST · VDD · INT · SDA · SCL` — same bus at the
  connector; `INT` is the controller's interrupt to the host.

### Front panel and audio

- `MIC+ · MIC-` — differential mic input; appears both standalone and in the
  KEY group.
- `ENC_A · ENC_C` — rotary encoder channels (the volume knob).
- `KEY2` — second button input, sibling of `KEY1`.
- `5V` / `GND` — panel assembly rails.

### Programming / service

- `GND · 3.3V · SDA · CLK · RST` (top edge, 5 plated holes) — `SDA`+`CLK`+`RST`
  at 3.3V reads as an **I²C programming header**, most likely for factory
  programming of the panel MCU or an EEPROM. Not a UART, not SWD.

### Power and unresolved

- `+12V`, `5V`, `GND` — rail probe points.
- `GND · VCI1 · DP · DK` — round test points by the USB breakout. `DP`
  duplicates the square pad; `VCI1` is a rail. **`DK` is uncertain** — possibly
  `DM` with the silkscreen partly lost, which would make it the pair's other half.

### Probing rule

Measure `TX` idle voltage before connecting anything; a confirmed 1.8V rail
exists on this board. Grounding `KEY1` is safe — it is a button input designed
to be pulled low. Shorting an unidentified point is not.

## Routes that do not involve flashing

Preferred over the USB force-flash, because none of them writes an image.

1. **Stock recovery, then recovery's own wipe.** Reset pinhole or key combo at
   power-on. Recovery's "wipe data / factory reset" is a *clean* wipe performed
   outside the running system, which is exactly what kills candidate 2 — a
   dangling `ENABLED_ACCESSIBILITY_SERVICES` row naming a package that is gone,
   as a partial wipe would have left. Cheapest thing that could actually fix it.
2. **adb from inside recovery.** ThunderSoft recoveries often start `adbd`. If
   this one does, the unit enumerates on USB *in recovery* despite Android's
   USB being host-fixed — different stack, different gadget config. That gives a
   way to mount `/data` and delete the offending rows surgically instead of
   wiping anything.
3. **Safe mode.** Boot without third-party apps. Useless if `system_server`
   itself is crashing; works if the loop is a launcher or app crash. Free to try.
4. **The UART**, read-only, as the diagnostic that says which of the three
   candidates is real before any lever is pulled.

## What the updater verifies — and what it does not

Relevant to whether a wrong or tampered image is dangerous. This is from the
community's observed behaviour, **not** from reading the updater binary.

| Stage | Checked? | Consequence |
|---|---|---|
| Package integrity, pre-write | **Yes**, probably — packages of this type carry their own integrity data | A truncated or corrupted download normally aborts instead of writing |
| Hardware/SoC compatibility, pre-write | **No** | This is precisely why "TS18 firmware on a TS10 is a permanent brick" is a standing warning. A working gate would refuse the package and nothing would brick. Assume the updater will write any structurally valid package handed to it. |
| Signature, post-write | **Yes** — AVB, `verifiedbootstate=green`, `flash.locked=1` | An unsigned or tampered image fails *at the next boot*. This is a trap, not a safeguard: you discover it by owning a unit that no longer starts. |

Two consequences:

1. Matching the exact build matters more here than on a platform with real
   pre-flash validation. Use the label: `TS10S`, `AW4C-0432-47`,
   `AW4B-0432-4A`, build `20240928.091908`.
2. **The mirror hosting our firmware publishes no checksum or signature.**
   lnx.pausacafeone.it is a community archive, not Topway and not the reseller.
   Package-internal integrity data proves the file is self-consistent, not that
   it is the image Topway built. Prefer the reseller or Topway, quoting the
   label above; treat the mirror copy as unverified until something confirms it.
