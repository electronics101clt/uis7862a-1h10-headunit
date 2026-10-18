# USB device-mode (ADB-to-PC) investigation

**Goal:** get the head unit to present as a USB **device** to a PC so adb works
over a USB-A↔USB-A cable (the way a Jancar unit did with the same cable).

**Outcome: not reachable on this unit without root.** Wireless ADB is the working
channel. Full reasoning below so it isn't re-litigated.

## What the unit is

- SoC: Unisoc `uis7862a`, USB controller is the **older musb OTG**
  (`sys.usb.controller = musb-hdrc.1.auto`, node `5fff0000.usb`), **not** a Type-C
  `usbc0` role-switch controller.
- Android 12 (API 31), `user` build: `ro.build.type=user`, `ro.debuggable=0`,
  `ro.secure=1`.

## The controller is stuck in HOST role

- `/sys/devices/platform/soc/soc:aon/5fff0000.usb/musb-hdrc.1.auto/mode = a_suspend`
  — an OTG **A-device (host)** state.
- `/sys/class/udc/musb-hdrc.1.auto/state = not attached` — the gadget is composed
  (`/config/usb_gadget/g1/UDC = musb-hdrc.1.auto`, `ffs.adb` loaded,
  `sys.usb.config = adb`) but the controller never sees a host.
- extcon (`/sys/class/extcon/extcon0/state`) stays `USB=0 USB-HOST=0` on every
  socket tried — no cable/VBUS is sensed on this controller.
- USB HAL (`dumpsys usb`): `current_mode=dfp` (host), `supported_modes=dual`,
  **`can_change_mode=false`**, `can_change_data_role=false`, `host_connected=false`.

So the port is host-fixed and the HAL itself reports the role as non-switchable.

## No software toggle exists for this board

- **Developer options:** only the three stock AOSP USB items (USB debugging / Debug
  mode when USB is connected / Revoke authorizations). No host-device selector.
  (The XDA "Developer options → ⋮ → USB Computer Connection" instruction is for a
  different ROM; it does not exist here.)
- **EngineerMode** (`com.sprd.engineermode`): zero USB/OTG components. Entry is by
  dialer secret code, no password — see codes below.
- **CarSetting** (`com.dofun.carsetting`): *has* a USB device-mode switch
  (`fragment/aa$5`), but it writes to **Type-C** nodes that don't exist here:
  ```
  rwrite /sys/bus/platform/devices/usbc0/otg_role   usb_device
  rwrite /sys/bus/platform/devices/usbc0.2/otg_role usb_device
  rwrite /sys/bus/platform/devices/10.usbc0/otg_role usb_...
  ```
  All three paths return `No such file or directory` on this musb unit.

## Why we can't force it

- The musb `mode` file is `root:root 0644`. Writing it needs **root**.
- No root available: no `su` binary anywhere, `su`/`su -c`/`su 0` all fail,
  `adb root` → "adbd cannot run as root in production builds", no suid-root binaries.
- **SELinux is Permissive** (`getenforce = Permissive`) — but permissive only drops
  SELinux **MAC** denials; **DAC** (Unix perms) still applies, so uid≠0 still can't
  write a root-owned file.
- **CarSetting runs as uid 1000 (system)**, platform-signed
  (`sharedUser=android.uid.system`), and its Factory menu has an on-device
  **"Adb shell"** box that runs commands as **system**. Still not root — system (1000)
  cannot write the root-owned `mode` file either.
- `rwrite` is just a **toolbox applet** (`/system/bin/rwrite -> toolbox`, not suid);
  it writes at the caller's uid, so it grants no elevation on its own.
- `setprop` on USB props (`sys.usb.config`, `persist.usb.otg`) has no effect — the
  role is below the property layer, decided by the musb/ID-pin, and the HAL re-derives
  state.

## Bottom line

On this musb-based `uis7862a` unit, USB-device ADB to a PC is **not achievable
without root**, and this is a locked `user` build. Use **wireless ADB**:

```
adb connect 192.168.49.100:<port>   # port is randomized per adbd restart
```

The wireless-debug port changes on every `adbd` restart; find it by mDNS
(`avahi-browse -tr _adb-tls-connect._tcp`) or a quick port scan of the unit.

## EngineerMode entry (no password — dialer secret codes)

Dial `*#*#CODE#*#*`:

| Code | |
|---|---|
| `83781` | main EngineerMode |
| `83782` | secondary menu |
| `837866` / `837868` | sub-menus |
| `33284`, `1688`, `759`, `0000` | other entries |
