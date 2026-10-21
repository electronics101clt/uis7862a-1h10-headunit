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

## Full boot-safety audit, 2026-10-21

Read-only over adb, with the unit up for 21 minutes. The question answered here
is narrow: **is there anything on this unit that would stop it booting again?**

The mechanism that matters is a **dangling component reference** — a secure
setting naming a component whose package is missing or disabled. That is the
classic way to crash-loop `system_server` at boot. Every such setting was
checked, and every component it names resolves to a package that is installed
and not disabled.

| Setting | Names | Status |
|---|---|---|
| `enabled_accessibility_services` | `com.samiadom.Shutdown/...accessService`, `com.zscreen.zos/...ZosAccessibilityService` | both installed, neither disabled |
| `default_input_method` | `com.android.inputmethod.latin/.LatinIME` | installed |
| `enabled_input_methods` | latin + `com.google.android.googlequicksearchbox` | both installed |
| `assistant` | `com.google.android.apps.actionsservice/...MainActivity` | installed |
| `voice_recognition_service` | googlequicksearchbox | installed |
| `selected_spell_checker` | latin spellcheck | installed |
| wallpaper | `com.android.systemui/ImageWallpaper` | built-in |
| `enabled_notification_listeners` | *(null)* | nothing to dangle |
| `enabled_notification_assistant` | *(null)* | nothing to dangle |
| `autofill_service` | *(empty)* | nothing to dangle |
| `voice_interaction_service` | *(empty)* | nothing to dangle |
| device owner / profile owner | **none set** | no `dpm` state |

Other boot-relevant state:

| Check | Value |
|---|---|
| HOME resolves to | `com.zscreen.zos.StartActivity` (installed, enabled) |
| HOME fallbacks present | `com.android.launcher2.Launcher`, `com.android.settings.FallbackHome` |
| `/data` free | 16 GB of 21 GB (23% used) |
| `ro.boot.verifiedbootstate` | `green` |
| `ro.boot.flash.locked` | `1` |
| `ro.boot.slot_suffix` | `_a` |
| `ro.boot.bootreason` | `recovery` — records how *this* boot was entered (KEY1 flash-mode path), not a pending instruction |
| Disabled packages | `com.google.android.ims`, `com.sprd.powersavemodelauncher`, `com.dofun.carassistant.car` — none referenced by any setting above |
| Crash buffer | One vendor NPU service, `android.hardware.neuralnetworks@1.3-service-xtensa`, aborting at startup. The ROM's own, unrelated to anything installed, and the unit runs with it. |

**Verdict: safe to reboot.** No dangling references, no device owner, HOME
resolves and has fallbacks behind it, disk is not full, and `misc` holds no
pending command — proven by the fact that this boot reached Android at all.

### Caps Lock as a boot-stage probe

With the unit fully booted, a USB keyboard's **Caps Lock LED now lights**.
During the boot loop it did not.

That LED is set by `InputReader` inside `system_server`, not by the kernel. So
its silence during the loop means `system_server` never started — confirming
independently that the unit was dying in **early init**, before the Android
framework. It also means the USB-keyboard recovery trick was never going to
work here: nothing above the kernel was alive to receive a keystroke.

Useful as a general probe on this hardware:

| Keyboard behaviour | Boot reached |
|---|---|
| LEDs flicker on plug-in | USB power only |
| SysRq responds | kernel + `usbhid`, and sysrq not yet disabled by init |
| **Caps Lock toggles** | **`system_server` is running** |
| Keys reach apps | fully booted |
