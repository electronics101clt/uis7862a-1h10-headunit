# How the large screen was recovered, and what actually corrupted its boot

Written 2026-10-21, after the unit came back. Supersedes the speculation in
`FACTORY-RESET-POSTMORTEM.md`, which is kept for the record.

## What fixed it

KEY1 grounded at power-on. The unit entered the bootloader's flash mode, looked
for a FAT32 stick, found none, printed **"USB FAT32 init failed"**, and fell
through to a normal boot. It came up on ZOS with **all user data intact** —
the pre-reset menu, apps and settings exactly as they were.

No firmware was written. No stick was present. The failed flash attempt is what
fixed it.

## What was actually wrong

A **pending command in the `misc` partition** (the bootloader control block).

1. An Android factory reset does not wipe anything itself. It writes a command
   into `misc` and reboots.
2. The bootloader reads that command and jumps to recovery, which performs the
   wipe.
3. Recovery clears the command **only on successful completion**.
4. Recovery never completed. The command stayed.
5. Every boot after: bootloader reads `misc` → jumps to recovery → fails →
   resets. ~7 seconds, indefinitely.
6. Grounding KEY1 sent the bootloader down the flash-mode path instead of
   honouring that command. The pending instruction was consumed, and the next
   boot ran normally.

**`/data` was never wiped.** That is proven by the user's own pre-reset state
being present after recovery, and it retrospectively disproves every theory in
the post-mortem that depended on `/data` having been destroyed or half-wiped.

### Why the timing mattered

| Observed | Means |
|---|---|
| ROM splash for **~7 seconds**, then restart | Bootloader handed off cleanly; kernel ran; died before userspace graphics |
| **No** animated boot animation, ever | `bootanimation` never started, so SurfaceFlinger never came up, so nothing in the Android framework ever ran |
| No USB keyboard response | USB host never enumerated; nothing above the kernel was alive to receive input |

Everything the post-mortem theorised about — HOME resolution, accessibility
rows, launcher availability — lives *above* the point where this unit stopped.
None of it was ever loaded.

## What Claude did and did not do

Audited across **307 transcript files in 10 project directories**, plus every
script and document in the ZOS repositories.

**Not once, on any unit:** `adb reboot recovery`, `adb reboot bootloader`,
`reboot autodloader`, `MASTER_CLEAR` broadcast, any write to
`/dev/block/by-name/misc`, `bootctl`, `set_active`, or `fastboot`. No writes
outside `/data` at all — nothing to `/system_tw`, `/vendor`, or `/sdcard/.Tcfg`.

The only matches are reads and documentation: a permission dump showing
`MASTER_CLEAR: granted=true` on a vendor app, a method-name listing from smali,
and web-search text about BROM recovery.

**No boot state was ever written by Claude on any unit.** What corrupted the
boot was the factory reset itself writing to `misc` — and the call to launch
that screen was denied by the owner before it ran; the reset was performed by
hand.

**What Claude did get wrong** was naming the AOSP reset screen
(`Settings$FactoryResetActivity`) as "the screen," while this repo's own
`README.md` had recorded the vendor's gated path — CarSetting Factory, PIN
`123456`, reset code `7890` — two days earlier. The TS10 community documents
that using the Android reset instead of the vendor one produces exactly this
boot loop. The wrong door was named; the right one was already written down.

## Is it safe to reboot? Yes.

Verified over adb, read-only, 2026-10-21:

| Check | Result |
|---|---|
| `enabled_accessibility_services` | `com.samiadom.Shutdown/...accessService` + `com.zscreen.zos/...ZosAccessibilityService` |
| Both packages installed? | Yes |
| Either disabled? | No |
| HOME resolves to | `com.zscreen.zos.StartActivity` (installed, enabled) |
| HOME candidates available | `com.android.launcher2.Launcher`, `com.zscreen.zos.StartActivity`, `com.android.settings.FallbackHome` |
| `ro.boot.verifiedbootstate` | `green` |
| `ro.boot.flash.locked` | `1` |
| `ro.boot.slot_suffix` | `_a` |
| `ro.boot.bootreason` | `recovery` |

The bootloader reads `misc` on **every** boot. This boot reached Android,
therefore `misc` holds no pending recovery command. The loop cannot resume on
its own — only another factory reset would write that command again.

`bootreason: recovery` records how *this* boot was entered (via the KEY1
flash-mode path). It is not a pending instruction.

**No dangling accessibility component**, which is the one remaining mechanism
that could crash-loop `system_server` at boot. Every service named by that row
points at a package that is installed and enabled.

### Correction to the post-mortem

Candidate 1 claimed this unit had no working HOME to fall back to — that both
candidates lived in `/data` and a wipe would take them together.
**That is factually wrong.** `com.android.launcher2.Launcher` is present and
resolves as a HOME candidate. If ZOS vanished, the unit would still have a home
screen.

## Rules going forward

1. **Never use the Android factory reset on a DoFun/ThunderSoft unit.** Vendor
   path only: Settings → Factory → `123456` → reset `7890`.
2. **KEY1 to ground at power-on is the proven escape** from a bootloader-level
   loop on this hardware. It needs no stick, no PC, and no disassembly — KEY1
   and KEY-GND are on the steering-wheel control wires of the main harness.
3. A failed flash attempt is a legitimate repair: entering flash mode clears a
   stuck `misc` command even when no firmware is written.
