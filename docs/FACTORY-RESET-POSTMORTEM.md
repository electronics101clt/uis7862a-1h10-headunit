# Factory reset across three units, and why this one boot-loops

Written 2026-10-20, after the owner factory-reset three head units on the same
day and got three different results. This unit — the large screen — is the one
that did not come back.

Graded the way the other repos grade: **proven**, **evidence**, **unproven**.
The cause here is **unproven**, and the point of this document is to say so
with the evidence attached, rather than to pick a story.

## The three results

| unit | how ZOS was installed | reset outcome |
|---|---|---|
| Jancar AC8257 / UJC201, `O7VODQ9HR48LDE89` | system app, `codePath=/jancar/oem/app/ZOS`, uid 1000, platform-signed | **reset cleanly, and ZOS survived as a launcher option** — intended. `/jancar/oem/app` is a read-only partition and a reset cannot reach it |
| the second unit | generic build, ordinary user app in `/data` | **reset cleanly.** ZOS went away with `/data`, the ROM's own launcher came back |
| **this unit**, `uis7862a_1h10_Natv`, `D244817163514186` | userland `/data` app, **HOME set over adb**, accessibility self-granted and pinned | **boot loop.** Unreachable since |

The first two outcomes are explained and are not faults. The Jancar kept its
system setup because a factory reset cannot wipe the partition it lives on —
that is the whole reason it was installed there. The second unit was a plain
`/data` app with no system entanglement, so the wipe was total and the unit
had a launcher to fall back to.

This unit sat between the two: a `/data` app that had nevertheless been given
system-level *state*. That is the difference worth examining.

## What was actually done to this unit

From the session transcripts and this repo's own notes. Everything here is
**proven** — it is a record of commands that ran.

1. **`cmd package set-home-activity com.zscreen.zos/com.zscreen.zos.StartActivity`**,
   run as an install step, repeatedly, documented as "idempotent … safe to run
   as a step every time". It writes a preferred-HOME row for user 0.
2. **ZOS self-granted its own accessibility service** by writing
   `Settings.Secure.ENABLED_ACCESSIBILITY_SERVICES` directly, because this ROM
   **auto-grants `WRITE_SECURE_SETTINGS`** at install. `SystemService` then
   **pinned** it — re-asserting the row on a tick, because
   `com.jancar.services` rewrites that set every ~20s.
3. **Nine globals and two accessibility rows** written by ZOS, per the Jancar
   audit, plus standby buckets, `captive_portal_mode`, `location_mode`.
4. **A VPN**, consented to by the owner, carrying the unit's connectivity.
5. **`pm uninstall com.zscreen.zos` / `pm install`** cycles — clean installs,
   never `-r`, so setup state was fresh each time.

### What was NOT done, checked rather than assumed

| suspicion | finding |
|---|---|
| a launcher was uninstalled or disabled here | **no.** The only package ever uninstalled on this unit is `com.zscreen.zos`. `com.android.launcher` was never touched. (This is the opposite of the Jancar, where `pm uninstall --user 0 com.android.launcher3` ran eleven times) |
| something was written outside `/data` | **no.** Nothing in any transcript writes `/system_tw`, `/vendor`, or `/sdcard/.Tcfg`. The `rwrite` lines in `USB-DEVICE-MODE.md` are *readings of CarSetting's smali*, not commands we ran — and they target Type-C nodes that do not exist on this board |
| the unit was flashed, or boot/AVB touched | **no.** `ROOT-AND-SYSTEM-APP.md` is a written-down plan that was never executed. `verifiedbootstate=green`, `flash.locked=1`, no image was ever built or pushed |
| a device owner or user restriictions were set | **no.** No `dpm` call in any transcript |
| I opened the factory reset menu | **no.** The `am start … Settings$FactoryResetActivity` call was **denied** by the owner before it ran. The reset was performed by hand on the unit |

**So nothing we did should survive a clean wipe of `/data`.** Every item in the
list above lives in `/data` and a complete reset removes all of it. That is the
honest starting point, and it is why the cause below is ranked rather than
stated.

## Candidate causes, ranked

### 1. There was no working HOME left to come back to — *evidence*

This unit's HOME before ZOS was **`app.lawnchair`** (per `README.md`), not the
stock launcher. A third-party launcher installed as HOME is usually a sign the
ROM's own launcher was unsatisfactory or absent from the home chain.

A reset removes `/data`, and therefore removes **both** Lawnchair and ZOS at
once. What is left is `com.android.launcher`, which has never been observed
resolving as HOME on this unit — only noted as "also present".

The repo already states this hazard for the small case and we never generalised
it to the large one. `docs/WIRELESS-DEBUG-LINK.md`:

> sequence link-risking steps deliberately: e.g. set HOME back to the stock
> launcher *before* uninstalling ZOS, so **there is no home gap** when the link
> bounces.

A factory reset is the maximal form of "uninstalling ZOS", and the step that
rule exists to protect — putting HOME back first — was never run, because a
reset was never thought of as that kind of operation. On these DoFun /
ThunderSoft builds a home gap at first boot does not produce a chooser; it
produces a restart loop.

**Why it is only evidence, not proven:** a clean reset also clears the
preferred-HOME row, so the framework should fall back to any remaining HOME
candidate. This requires `com.android.launcher` to be unable to serve, which is
plausible given Lawnchair's presence but has never been tested.

### 2. A secure-settings row outliving the package it names — *unproven*

If the wipe was **partial** rather than clean, `ENABLED_ACCESSIBILITY_SERVICES`
can survive still naming `com.zscreen.zos`, whose APK is gone. A dangling
accessibility component is a known way to crash-loop `system_server` at boot.

This unit is the only one of the three where that row was ever written, because
it is the only one whose ROM auto-grants `WRITE_SECURE_SETTINGS` to a `/data`
app. The Jancar held the same permission but as a genuine system app, and the
second unit never held it at all. **That makes this the one mechanism unique to
the unit that broke** — which is why it is listed second despite being
unproven.

### 3. The wipe was interrupted — *unproven*

On this unit **ZOS provides the network link**: adb is wireless-only, over
PdaNet WiFi Direct, and the repo records that the link drops on every install
because the app holding it is being replaced. A factory reset tears that path
down at once and far harder.

No evidence either way. It is here because an incomplete `/data` wipe is the
precondition for candidate 2, and this is the mechanism that would produce one.

## Recovery, and a correction

**USB is not a route in on this board.** I suggested a USB cable and
`adb wait-for-device` earlier in the session that produced this document; that
was wrong, and `docs/USB-DEVICE-MODE.md` already said so. The musb controller
is **host-fixed** — `can_change_mode=false`, extcon never senses VBUS — so the
unit cannot present as a USB device to a PC in Android at all. There is no boot
window to catch.

What is left, cheapest first:

1. **Recovery, from the unit itself.** Key combo / reset pinhole into the
   stock recovery, then a wipe or an SD-card update package. Hands on the unit,
   no PC needed.
2. **SPD / BROM download mode.** Unisoc silicon, so SPRD U2S Diag / `spd_dump`
   with the matching `.pac`. BROM enumerates as SPRD diag, **VID 1782** — a
   *different* USB path from Android's musb gadget, so the host-fixed port does
   not rule this out. **Unverified on this unit**, and it needs firmware
   matching `TS10S.2.1_20240928` / app `TS10S_20241102.114435_APP-AC58`,
   build 39609. The unit is A/B with AVB enforcing, so the image has to be right.
3. The CarSetting Factory and lock passwords in `README.md` are worth having to
   hand if any vendor screen does come up during recovery.

## What to change, before the next unit

1. **A factory reset is a link-risking, HOME-risking operation.** It belongs in
   the same category as `pm uninstall com.zscreen.zos`, under the same rule:
   **set HOME to a package that lives outside `/data` first**, and confirm that
   package actually resolves as HOME, before wiping anything.
2. **Never set HOME to a `/data` app as an unconditional install step.** This is
   `FORK.md`'s *"stop marrying it to a device"* in its most expensive form: not a
   hardcoded constant, but a hardcoded *change to the unit's own state*, applied
   every install, with no record of what it displaced and no way back once the
   app it points at is gone. The rule has been written three times for code; it
   was never applied to the adb steps beside the code, and the adb steps are
   what left a unit unbootable.
3. **Verify the fallback before relying on the wipe.** One command, while a unit
   is still up, answers candidate 1 for every future unit:
   `adb shell cmd package query-activities -a android.intent.action.MAIN -c android.intent.category.HOME`
   — if the only HOME candidates live in `/data`, that unit cannot survive a
   reset, and that is worth knowing *before* resetting it rather than after.
4. **Record what HOME was before changing it**, so a reset or a rollback has a
   named target instead of a guess.

---

# Addendum, 2026-10-21: the cause is no longer unproven

Everything above this line was written before the board was photographed, the
boot behaviour was timed, and the platform's own documentation was found. Three
findings supersede the ranking above.

## 1. The AOSP reset path is a known brick on this platform

The reset was performed through **`com.android.settings/.Settings$FactoryResetActivity`**
— AOSP's reset screen inside ThunderSoft's modified Settings. This is **not**
the vendor's reset. The vendor's is a different app: `com.dofun.carsetting`,
reached as *Settings → Factory → PIN `123456` → reset code `7890`*.

The TS10 community states the distinction explicitly:

> "Try factory reset from the settings menu, **NOT Android reset**. This is an
> important distinction." … "don't use the reset factory option inside *More*."

And records the consequence of getting it wrong as exactly this unit's symptom:
a boot loop stuck at the **Powered by Android** screen.

**This supersedes candidates 1, 2 and 3 above.** It is not an inference from
what we did; it is a documented failure mode of the specific action taken.

**And the gated path was already written down in this repo** — `README.md`
recorded `FACTORY_LOCK 123456` / `RESET_FACTORY 7890` two days before the reset.
It was never mentioned when the reset screen was being located. That is the
actual fault: not that the AOSP screen was launched — the owner denied that call
and performed the reset by hand — but that it was the only screen named, while
the correct one sat in our own notes.

## 2. The boot stops in early init, below the Android framework

Observed: the ROM splash lasts about **7 seconds**, then the unit restarts. It
**never reaches an animated boot animation**.

(An earlier draft of this addendum recorded 1-3 seconds and concluded the fault
was at the bootloader. That timing was wrong. At 7 seconds the bootloader has
handed off successfully - so AVB passed and the boot images verify - the kernel
has run, and the failure is in **early init**.)

`bootanimation` is a userspace service started once SurfaceFlinger is up, well
before `system_server` completes. Never seeing it means nothing in the Android
framework ever runs - no launcher, no HOME resolution, no accessibility service.

This **rules out candidates 1 and 2 of the original ranking** on their own terms:

- A missing HOME yields a booted system with no launcher, not a splash hang.
- A dangling accessibility row crashes `system_server` - and the boot animation
  would be looping on screen while it did.

What fits a ~7 second cycle is **init failing on `/data`**: the AOSP reset
formatted the partition in a way this vendor ROM will not accept - wrong
encryption policy, or missing the vendor-seeded structure its init expects - so
the mount or prepare step fails fast and init reboots. That is consistent with
finding 1: it is the specific damage the vendor's gated reset exists to avoid.

## 3. Consequences for recovery

Everything that operates above the failure point is unavailable:

| Route | Status |
|---|---|
| adb | Dead. Wireless-only on this unit, and the network stack never starts. |
| USB keyboard / SysRq | Effectively dead. USB host may just manage to enumerate within ~7s, but Android's `init.rc` writes `/proc/sys/kernel/sysrq 0`, closing the SysRq window as init runs; no Caps Lock response was observed. The Alt+PrtScn+`i` trick is in any case documented for **Astar_d7 / Android 6**, not for UIS7862. |
| Recovery menu navigation | Requires input the unit cannot accept. |
| `/data` wipe from recovery | **Would address the fault** - but requires input the unit cannot accept. |

What remains is the one lever that runs *below* everything else: **KEY1 to
ground with a prepared USB stick in USB1**, read by the bootloader off a GPIO
before USB, the kernel, or init exist. The bootloader is demonstrably alive and
healthy - it paints the splash and successfully hands off to the kernel.

The platform's documented repair for this exact state is *System Upgrade →
System Local Upgrade → **"wipe data and format flash"** → USB*, ~6 minutes, then
roughly a minute of auto-initialisation on first boot. On a unit that will not
boot, the bootloader-driven force-flash is the same operation. **Use the variant
that formats data** — a plain image update leaves the mangled partition in place.

## Corrections to the body above

- "The cause here is **unproven**" — no longer true; see finding 1.
- The recovery section's "USB is not a route in" holds **for adb only**. The
  unit reading a stick is a different mechanism and is the working route.
- `docs/HARDWARE-ID-AND-RECOVERY.md` carries the board identity, the pad map,
  the firmware, and the procedure.
