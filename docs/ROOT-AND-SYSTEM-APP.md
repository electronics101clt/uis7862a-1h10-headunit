# Root / su, and running ZOS as a system app

Goal: run **ZOS as a system app (uid 1000)** on this unit.

## There is no built-in su

- No `su` binary anywhere; `su`/`su -c`/`su 0` all fail.
- `adb root` → "adbd cannot run as root in production builds" (`ro.build.type=user`,
  `ro.secure=1`, `ro.debuggable=0`).
- CarSetting's `rwrite` / factory write channel runs as **system (uid 1000), not
  root** (`rwrite` is a `toolbox` symlink, executes at the caller's uid). So it is
  **not** a route to root.
- **Ceiling without flashing = uid 1000 (system)**, available today through
  CarSetting's Factory → "Adb shell" box, which runs as system.

## The uid-1000-by-signing method does NOT work on this unit

uid 1000 requires an app signed with the **platform key** and declaring
`sharedUserId="android.uid.system"`. The platform cert here is **ThunderSoft's own
private key**, not the public AOSP test key:

```
Signer DN:  CN=John, OU=Android, O=TW, L=Shenzhen, ST=Guangdong, C=CN
SHA-256:    aa6f9fb3070512ac962425797cd65aa585cf6202937ee3ceefb14b5802eabdf3
SHA-1:      409d305d6bb9d9108c48179e5e05641ca29b9cd0
```

("John" / "TW" = ThunderSoft — same John as `john@tw-desktop`.)

Because it is **not** the AOSP test key, signing ZOS with the AOSP test key +
`sharedUserId=android.uid.system` is rejected (signature mismatch with the existing
`android.uid.system`). Only ThunderSoft holds that private key. **This is the key
difference from the AC8257**, where the platform was AOSP-testkey-signed and that
method worked.

## What's actually achievable for ZOS

| Want | Needs | On this unit |
|---|---|---|
| Run privileged shell commands now | uid 1000 (system) | ✅ CarSetting Factory → "Adb shell" box |
| ZOS as **priv-app** (privileged perms) | root to place in `/system_ext/priv-app` | ⚠️ needs root (below); no platform key required for priv-perms |
| ZOS as true **uid 1000** (sharedUserId) | ThunderSoft platform private key | ❌ not available |
| ZOS holding **signature-level** perms | platform key | ❌ not available |

So: priv-app via root covers most "system app" needs; exact uid-1000 identity is
blocked by the missing platform key.

## Rooting path (if pursued) — Magisk + AVB re-sign

Device facts that shape it:

- Firmware: `UNISOC/uis7862a_1h10_Natv/uis7862a_1h10:12/SP1A.210812.016/39609:user/release-keys`
  - ThunderSoft `TS10S.2.1_20240928.091908_OS-EN`, app `TS10S_20241102.114435_APP-AC58`
- **A/B slots**, has `boot_a/b` **and** `init_boot_a/b`, `dtbo_a/b`, `vbmeta*`.
- AVB: `verifiedbootstate=green` (enforcing). `ro.boot.flash.locked=1` (locked),
  `sys.oem_unlock_allowed=0` though `ro.oem_unlock_supported=1`.
- Boot block is `root:root` — cannot pull `boot.img` as shell; must get it from the
  matching firmware package.

Community method for uis7862a (see sources):

1. Obtain the **stock boot.img** from the firmware/update.zip matching this build
   (TS10S AC58 / build 39609).
2. Install **Magisk v27** on the unit, patch boot.img → `magisk_patched-27xxx.img`.
3. **Re-sign for AVB** so `verifiedbootstate=green` still accepts it:
   `bash avb_script.sh boot.img magisk_patched-27xxx.img` → signed image.
   (Re-signing is what avoids needing to unlock the bootloader / disable AVB.)
4. Flash the signed patched boot:
   - via **SPD / BROM download mode** (SPRD U2S Diag / `spd_dump`, FDL2) — works on
     locked units and is the **unbrick** path if anything goes wrong; or
   - the unit's own `boot_new.img` self-flash (place signed image on internal
     storage, reboot) where supported.
5. Reboot → Magisk/`su` present, including over wireless adb.

**Risk:** wrong image/slot can bootloop. Recoverable by reflashing the full `.pac`
in BROM mode. Do not attempt without the matching firmware and a confirmed unbrick
setup. Note: USB-device mode to a PC doesn't work in Android on this board (see
`USB-DEVICE-MODE.md`); BROM mode enumerates differently (SPRD diag, VID 1782), so a
PC-side SPD connection may still be possible even though Android adb-over-USB isn't —
**unverified on this unit.**

## Sources

- https://xdaforums.com/t/general-fyt-based-spreadtrum-uis7862-s-unisoc-ums512-uis8581a-sc9863-q-a-mods-tips-firmware.4396339/
- https://xdaforums.com/t/fyt-tesla-style-head-units-uis7862-etc-drm-l3-root-mods-etc.4499743/
- https://shidevil.github.io/tutorial/Rooting-UIS7862-Android-Car-Headunit/
- https://www.hovatek.com/forum/thread-49095.html (SPRD/Unisoc PAC firmware & BROM)

## Built-in su investigation (ruled out)

Checked every candidate for a hidden root channel — none is a usable su:

- **`gocsdk`** (runs as root) = MCU serial bridge: `/system_tw/bin/gocsdk /dev/ttyS0
  500000`. Its sockets `goc_control` / `goc_data` / `goc_spp` are world-writable
  (`srw-rw-rw- root root`) but they pipe to the **microcontroller**, not a shell —
  not a command executor.
- **Root `sh` processes** are vendor init scripts only: `zlink5.sh`,
  `ip_monitor.sh` (started by init, not interactive).
- Other root daemons (`uniber`, `i_u`, `eq47925`, `srmi_proxyd`, `engpc`,
  `update_engine`, `cp_diskserver`) are telephony/modem/OTA services with no
  general command interface exposed to apps.
- **`rwrite`** (used by CarSetting's factory writes) is `/system/bin/rwrite ->
  toolbox`, i.e. a toolbox applet that runs at the **caller's uid**. CarSetting
  invokes it as **system (uid 1000)** — a system-privileged *file write*, not root
  and not a shell.

**Conclusion:** no built-in su. The privilege ceiling without flashing is **uid 1000
(system)** via CarSetting's Factory → "Adb shell" box. Root requires the Magisk +
AVB-resign flash above.
