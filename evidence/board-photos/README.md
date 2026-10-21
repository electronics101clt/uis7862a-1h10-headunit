# Board photographs

Phone photographs of the large screen's panel/carrier board, taken 2026-10-20
23:45-23:46 while the unit was out of its chassis, plus the chassis label and
the QR string captured 2026-10-21.

Resized to 2048px for the repository; the originals are ~5 MB each and live on
the owner's phone. Everything read out of them is transcribed in
`docs/HARDWARE-ID-AND-RECOVERY.md`.

| File | What it shows |
|---|---|
| `board-01.jpg` | Wide shot, whole board in the black chassis frame |
| `board-02.jpg` … `board-13.jpg` | Close-ups, sweeping across the board |
| `key-connector-closeup.jpg` | **The `KEY` FFC and its labelled test points** |
| `chassis-label-1.jpg`, `-2.jpg` | `AC TS10S 7708 4+32G 13.3 FJ HW` |
| `qr-string-screenshot.png` | QR decode: `YM*E1208*12.AW4C-0432-47*TS18/TS10M-A125M-V16/V1.0/AW4B-0432-4A/TS10S.2.1_20240928.091908*00499` |

## The KEY connector

Left edge of the board, upper of the two left-edge FFCs (the wider one below it
is `TFT` - the panel; do not disturb it). Cream body, dark flip latch, ~15-16
contacts, `KEY` silkscreened below.

Test points to its right, in silkscreen order:

```
MIC+      microphone
MIC-
ENC_A     rotary encoder, channel A
ENC_C     rotary encoder, common
RST
KEY2      button
KEY1      button - and the force-flash lever
5V
```

The rotary, the push and the microphone all share this one cable. If the push
works while rotation and microphone are dead, the fault is at the `MIC`/`ENC`
end of this FFC - a seating problem or a cracked conductor - not the encoder.
