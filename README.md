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
| ROM base | ThunderSoft ("tw" — `/system_tw`) |
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
authorization, not a password). It is a ThunderSoft addition baked into the vendor
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

(Matches the known ThunderSoft "john @ tw-desktop" USB-debug password seen on other
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

These are standard DoFun/ThunderSoft factory defaults, documented across head-unit
forums — not a personal secret.
