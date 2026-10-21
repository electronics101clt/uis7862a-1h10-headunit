# uis7862a_1h10_Natv — Head Unit Notes

Device-specific working notes for the Android car head unit reporting itself as
`uis7862a_1h10_Natv`. Owner-owned unit; this repo is the reference for working on it.

## Device identity

| Field | Value |
|---|---|
| Model / device | `uis7862a_1h10_Natv` / `uis7862a_1h10` |
| Android | 12 (API 31) |
| ABI | arm64-v8a |
| Serial | `D244817163514186` |
| SoC vendor | Unisoc / Spreadtrum (`com.unisoc.*`, `vendor/sprd`, SoC `uis7862a`) |
| ROM base | **TopWay** ("tw" — `/system_tw`, `com.tw.*`). See `docs/PLATFORM-AND-VENDORS.md`; an earlier revision of this file attributed `tw` to ThunderSoft, which is not supported by evidence. |
| Car skin | DoFun (`com.dofun.carsetting`) |
| Launcher (HOME) | `app.lawnchair` (Lawnchair); stock `com.android.launcher` also present |
| Notable apps | `com.zjinnova.zlink`, `com.zscreen.wifisettings` |

## Wireless ADB

- Reached over WiFi Direct / PdaNet hotspot (`DIRECT-PdaNet-...`), device IP `192.168.49.100`.
- adb TLS (wireless debug) port: **44663** (`service.adb.tls.port`). This machine's key is already paired/trusted.
- Connect: `adb connect 192.168.49.100:44663`
- The device also advertises `10.1.10.1:44663` on its own `tun0` (`/32`) — same daemon, not routable from outside; ignore it, use `192.168.49.100`.
- USB debugging was off by default (`adb_enabled=0`); wireless debugging on (`adb_wifi_enabled=1`).

## CONFIRMED: USB debugging password

Toggling **USB debugging** in Developer options pops a vendor password `EditText`
before the stock "Allow USB debugging?" dialog. This gate is **not AOSP** — Google
does not password-protect USB debugging (entry to Developer options asks at most for
the existing lock-screen credential; USB debugging itself is gated by RSA key
authorization, not a password). It is a vendor (TopWay) addition baked into the vendor
Settings app.

**Password:** `john@tw-desktop`

Source — hardcoded in the vendor-modified Settings:

- APK: `/system_tw/priv-app/Settings/Settings.apk` (`com.android.settings`)
- Class: `com.android.settings.development.AdbPreferenceController`
- Check: `lambda$showConfirmationDialog$0` compares the dialog `EditText` against a
  hardcoded literal:

```smali
const-string p2, "john@tw-desktop"
...
invoke-virtual {p2, p1}, Ljava/lang/String;->equals(Ljava/lang/Object;)Z
```

(Matches the known "john@tw-desktop" USB-debug password seen on other
units with this ROM base.)

## Gotchas

- Enabling USB debugging restarts `adbd`, which can briefly drop the wireless adb
  *session* on 44663 (it re-establishes, key already trusted). It does **not** touch
  the WiFi Direct/PdaNet link the launcher holds.
- Boot is heavy: load average ~10 for the first several minutes. WiFi link showed
  ~10% ping loss / zeroed `/proc/net/wireless` quality early after boot.

## CONFIRMED: CarSetting factory / lock passwords

The DoFun CarSetting app (`com.dofun.carsetting`) gates its Factory and lock
screens with a password. These are **hardcoded defaults** in its `PasswordFragment`
(`fragment/s.smali`); an override, if ever set, is stored in
`/sdcard/.Tcfg/password.cfg`. On this unit that file does **not** exist, so the
defaults below are live.

| Dialog / lock | Internal key | Password |
|---|---|---|
| **Factory** | `FACTORY_LOCK` | **`123456`** |
| User lock | `USER_LOCK` | `8888` |
| Reset Factory | `RESET_FACTORY` | `7890` |

These are standard DoFun/TopWay factory defaults, documented across head-unit
forums — not a personal secret.

## USB device-mode (ADB over USB) — investigated, not reachable without root

Full writeup: [docs/USB-DEVICE-MODE.md](docs/USB-DEVICE-MODE.md).

Short version: this is the older **musb OTG** controller (`musb-hdrc.1.auto`), fixed
in **host** role (`mode=a_suspend`); the USB HAL reports `can_change_mode=false`.
No host/device toggle exists in Developer options, EngineerMode, or CarSetting for
this board (CarSetting's switch targets Type-C `usbc0/otg_role`, which doesn't exist
here). The role file is `root:root` and there's no root (`user` build, no `su`,
`adb root` refused; SELinux is Permissive but DAC still blocks). **CarSetting runs as
uid 1000 (system)** and its Factory menu has an on-device "Adb shell" box — useful,
but system ≠ root, so it can't flip the role either. **Use wireless ADB.**

EngineerMode entry codes (no password): `*#*#83781#*#*` (main), also 83782, 837866,
837868, 33284, 1688, 759, 0000.
