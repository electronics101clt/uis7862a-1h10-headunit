# How the large screen was recovered, and what actually corrupted its boot

Written 2026-10-21, after the unit came back. Supersedes the speculation in
`FACTORY-RESET-POSTMORTEM.md`, which is kept for the record.

## What happened, in order

A compressed account of the whole episode, from the reset to the revival. The
sections below carry the evidence for each claim.

| # | Event |
|---|---|
| 1 | Claude located the reset screen on the unit and named **`com.android.settings/.Settings$FactoryResetActivity`** - the AOSP one. Its attempt to launch it was **denied by the owner**. The vendor's gated path (CarSetting Factory, PIN `123456`, reset `7890`) was already recorded in this repo's `README.md` two days earlier and was never mentioned. |
| 2 | The owner performed the reset **by hand**, through that AOSP screen. |
| 3 | The reset did not wipe anything. It wrote a command into the **`misc`** partition and rebooted - which is all an Android factory reset does. |
| 4 | The bootloader read that command and jumped to recovery to carry out the wipe. **Recovery could not complete it.** The command is only cleared on success, so it stayed. |
| 5 | Every boot from then on: bootloader reads `misc` -> jumps to recovery -> fails -> resets. **~7 seconds, indefinitely.** The ROM splash appeared; the animated boot animation never did. |
| 6 | The unit was unreachable for roughly a day. adb is wireless-only here and the network stack never started. A USB keyboard never enumerated. Nothing above the kernel was ever alive. |
| 7 | Diagnosis went through three wrong answers - missing HOME, a dangling accessibility row, a half-wiped `/data` - each ruled out in turn by the splash timing and by the community's documentation of this exact failure. |
| 8 | The board was photographed and identified: **AW4C-0432-47 / AW4B-0432-4A, TS10S.2.1_20240928.091908, 4+32G, 13.3-inch**. A `GND/KEY0/TX/RX` debug header was found, and `KEY1` was identified on the harness. |
| 9 | The owner grounded **KEY1** at power-on. The bootloader entered flash mode, found no USB stick, printed **"USB FAT32 init failed"**, and fell through to a normal boot. |
| 10 | **The unit came up on ZOS with every byte of user data intact** - the pre-reset menu, apps and settings exactly as they were. No firmware was written. The failed flash attempt is what fixed it, because entering flash mode consumed the stuck command. |
| 11 | A full read-only audit found no dangling component references, no device owner, HOME resolving with two fallbacks behind it, and 16 GB free. |
| 12 | The owner rebooted deliberately. It came back clean: **`bootreason: reboot,userrequested`**, where the previous boot had been `recovery`. |

### What this cost, and what it did not

**It did not cost any data.** `/data` was never wiped - 247 package directories
survive, and ZOS's `firstInstallTime` predates the incident.

**It was not caused by ZOS.** The unit boots normally with ZOS installed,
enabled and set as HOME, with zero crashes attributable to it across two boots.

**It was caused by using the wrong reset path**, which the TS10 community
documents as producing exactly this boot loop - and the right path was already
written down in this repository before it happened.

---

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

## Pre-existing defect: the NPU HAL crash-loops permanently

Found during the deep check, 2026-10-21. **Not related to the boot loop**, but
it is a real fault on this unit and worth having written down.

```
/vendor/bin/hw/android.hardware.neuralnetworks@1.3-service-xtensa
signal 6 (SIGABRT)
Abort message: 'Check failed: totalCores != 0 '
```

Restarting every ~5 seconds. **288 crashes in 24 minutes of uptime**, and
counting. `init.svc.neuralnetworks_hal_service_xtensa` is permanently
`[restarting]`, and init has flagged it:

```
[sys.init.updatable_crashing]: 1
[sys.init.updatable_crashing_process_name]: neuralnetworks_hal_service_xtensa
```

### Why it is not the boot loop

**It is doing this right now while the unit is up and stable.** If it caused a
reboot, the unit would be looping in front of us.

The reason it cannot is in its own init file,
`/vendor/etc/init/android.hardware.neuralnetworks@1.3-service-xtensa.rc`:

```
service neuralnetworks_hal_service_xtensa /vendor/bin/hw/android.hardware.neuralnetworks@1.3-service-xtensa
    class hal
    user system
    group system
```

No `critical` flag. In Android init only a `critical` service triggers a device
reboot after repeated crashes; everything else is restarted indefinitely.
`sys.init.updatable_crashing` is init *noting* the loop, not acting on it.

### What causes it

The Xtensa NPU HAL asks the driver for the available core count, gets **zero**,
and aborts on its own assertion. That is a vendor image shipping a
neural-processing HAL for silicon this board does not have enabled — routine on
these units, where a generic Unisoc vendor build lands on a reduced-feature
board. This unit is the **4+32G** variant; the NPU is one of the parts that is
not there.

### Not ours

`/vendor/bin/hw` is an unmodified vendor binary. The transcript audit across 307
files found **no writes outside `/data`** on any unit, ever — nothing to
`/vendor`, `/system_tw`, or `/sdcard/.Tcfg`. This has been happening since the
unit was new.

### What it costs

Continuous process churn: a fork, a crash, a dump and a restart every five
seconds, forever. It is a visible contributor to the load average of ~9 that
this unit carries, which the README already records as "boot is heavy".

Harmless to correctness, wasteful in practice, and not fixable without a vendor
image that omits the HAL.

## Post-reboot audit, 2026-10-21 — the question settled

The owner rebooted the unit deliberately. It came back normally. That reboot,
and the state afterwards, settles whether the loop was caused by ZOS or by the
factory reset.

### The reboot itself

| Property | Before (recovered boot) | After (deliberate reboot) |
|---|---|---|
| `ro.boot.bootreason` | `recovery` | **`reboot,userrequested`** |
| `ro.boot.slot_suffix` | `_a` | `_a` |
| `ro.boot.verifiedbootstate` | `green` | `green` |

The bootloader is no longer being diverted to recovery. The pending command in
`misc` is gone, and a normal reboot now does what a normal reboot should.

### No wipe ever executed

| Evidence | Value |
|---|---|
| `/data` | created 2024-09-27 21:45 — never recreated |
| `/data/misc` | same stamp — untouched |
| `/data/data` | **247 package directories present** |
| `com.zscreen.zos` `firstInstallTime` | 2024-09-27 22:31 — predates the incident |
| ZOS data directory | present, never removed |

A completed factory reset recreates `/data` and removes every package
directory; `firstInstallTime` would have reset with it. None of that happened.

**Caveat on the dates:** this unit's RTC falls back to 2024-09-27 when it boots
without network time (the same reason a photograph of the recovered unit showed
"Friday 27 September"). The timestamps are unreliable as calendar dates but
reliable as relative evidence — they are the original provisioning stamp, not a
fresh one.

So the reset was **written and pending, never performed**. The unit spent the
entire outage obeying an instruction that recovery could not carry out.

### ZOS is clean across both boots

- Zero `com.zscreen.zos` entries in the crash buffer.
- Zero `system_server` crashes.
- The only crashing process is the vendor NPU HAL — 49 aborts in the first 4
  minutes after reboot, identical `Check failed: totalCores != 0`, exactly as
  before. Pre-existing, `class hal` and not `critical`, so it cannot reboot the
  device.

### Verdict

**The boot loop was the factory reset, not ZOS.** The unit boots normally with
ZOS installed, enabled and set as HOME, twice in a row, with no crash
attributable to it.

ZOS remains a suspect for the *other* unit, which looped while its APK was being
replaced — a different failure, a different mechanism (see the accessibility
re-pin), and unproven without logs from that unit.

## The firmware that was staged (and why it was never flashed)

Before the unit was recovered by KEY1, the plan was a USB force-flash. The exact
build for this unit was located, downloaded and verified:

- File: `TS10S.2.1_20240928.091908_OS-EN.zip` (1.95 GB)
- Stock:  `https://lnx.pausacafeone.it/car_tablet/topway_ts10_firmware/TS10S.2.1_OS-EN/TS10S.2.1_20240928.091908_OS-EN.zip`
- Rooted: same path, `..._OS-EN_root.zip`
- Index:  `https://lnx.pausacafeone.it/wordpress/2022/12/09/topway-ts10-ts18-firmware/` (archive password reported `TopwayTS10`)
- Verified: `unzip -t` clean; `sha256 b8b1f658c8b29c1c84a03788160ac3bf23ee39eb4595deae51053e7d67b497c7`
- Community mirror, **no vendor checksum** - see the updater-hijack advisory in
  `PLATFORM-AND-VENDORS.md`. A checksummed image from DoFun is preferred.

It was never flashed. KEY1 recovered the unit with all data intact, so the flash
became unnecessary - the file is kept only as a fallback.

## Open question: could ZOS still be implicated?

The large screen's loop is attributed to the AOSP factory reset (stuck `misc`
command), and the post-reboot audit found no wipe occurred and no ZOS crash.
That is strong, but it is **not a root-cause proof**, and the limits are honest:

- **No root on this unit.** It is a `user` build; `adb root` is refused and
  there is no `su` without a Magisk flash. The deepest logs (early-init, the
  `misc` contents at the time, kernel dmesg across the loop) could not be read.
- **No UART.** The `GND/TX/RX` debug pads were identified but never connected;
  no serial console capture of the actual boot failure exists.
- **No JTAG.** No hardware debug rig, so no way to observe the bootloader's read
  of `misc` directly.

So the account rests on **inference from what was observable over adb after the
fact** (intact `/data`, the `recovery` then `reboot,userrequested` bootreasons,
no dangling component), plus the community's documentation that the AOSP reset
path bricks these units this way. That is a well-supported conclusion, not a
logged one. If the other unit's install-time loop is ever reproduced with a UART
attached, the ZOS accessibility-pin mechanism could be confirmed or ruled out
directly - which this unit's constraints did not allow.
