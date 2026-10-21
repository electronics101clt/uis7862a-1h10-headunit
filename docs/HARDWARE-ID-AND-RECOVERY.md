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
