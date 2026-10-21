# Writing your own image without bricking a unit

What it would take to make ZOS a system app on a TopWay unit the way it is on
the Jancar, which of the locks are real, and - most importantly - **what not to
touch**, so a unit is never bricked permanently the way the large screen nearly
was.

Written 2026-10-21. Graded: **legally/technically established**, **checkable**,
or **do-not-attempt**.

## The three locks, and whether each opens

### 1. Kernel source - you are legally owed it (established)

The Linux kernel is GPLv2. Anyone shipping it must provide the source on
request. **Unisoc and TopWay are obligated.** Many Chinese vendors stall, but it
is a legitimate demand with law behind it, and it is the kind of thing a
commercial partner relationship actually unlocks. No hacking required - ask.

### 2. AVB / vbmeta signing key - NOT obtainable (do-not-attempt)

This unit is `verifiedbootstate=green`, `flash.locked=1`. The bootloader
verifies every image it hands off to against the vendor's AVB key. A
self-built image **will not boot** without that key, and no engineer can hand it
out - it protects their entire fleet.

**This is the brick line.** Flashing a modified or self-signed boot/system image
to a locked-AVB unit produces a unit that fails verification at boot and does
not come up. Unlike the factory-reset loop (which was recoverable because the
images were intact), a bad AVB signature is **not** fixed by KEY1 or a USB
reflash of the same bad image. Recovery then means SPD/BROM with a known-good
`.pac`, and if the layout is wrong that can fail too.

**The realistic version of this ask:** an engineering image with AVB relaxed or
the bootloader unlocked, which vendors do provide to commercial partners. That
is a negotiation, not a hack.

### 3. Platform signing key - possibly already public (checkable)

A system app's privilege comes from being signed with the ROM's **platform
key**. Many Chinese ROMs ship signed with the public AOSP **`test-keys`** - the
ones in every AOSP checkout, fingerprint known to everyone. If TopWay's platform
cert is the AOSP test-key, you can platform-sign your own system app **with no
vendor cooperation at all**.

Check, read-only:

```
adb shell pm list packages -f com.tw.core        # find a system APK path
adb pull <that path> tw.apk
apksigner verify --print-certs tw.apk             # or: keytool -printcert -jarfile
```

Compare the SHA-256 of the signing cert to the AOSP test-key
(`build/target/product/security/platform.x509.pem` in any AOSP tree). If they
match, the key is public and ZOS can be signed with it. If they do not, the key
is TopWay's own and only they have it.

This is already solved on the Jancar - ZOS is platform-signed there and its
`BIND_APPWIDGET` grant proves the signature is accepted. The question is only
whether TopWay reused the public key.

## The route that needs no keys at all (the Jancar method)

ZOS is a system app on the Jancar **without** any vendor key, because of a
writable partition that PackageManager scans:

- `/system` (mmcblk0p31) is **hardware write-protected** - writes return rc=0,
  read back correctly, then vanish on `drop_caches`. Do not rely on it.
- `/jancar/oem/app` (mmcblk0p37) is mounted rw, PMS was patched to scan it, and
  writes survive `sync` + `drop_caches`. An app placed there gets `SYSTEM` and
  `PERSISTENT` flags.

**The open question for TopWay:** is there an equivalent? The unit has
`/system_tw` as a separate partition with its own `priv-app` (every `com.tw.*`
and `com.dofun.*` app lives there). Whether it is writable, and whether writes
persist, is a **test, not a negotiation** - and a safe one, because writing an
APK to a data-like partition cannot brick the unit the way an AVB-failing image
can.

The canary test (from the Jancar `install-system.sh`), read-mostly:

```
adb root
adb shell 'su 0 sh -c "echo canary > /system_tw/.zos-canary"'   # WRITE - ask first
adb shell su 0 sync
adb shell 'su 0 sh -c "echo 3 > /proc/sys/vm/drop_caches"'
adb shell su 0 cat /system_tw/.zos-canary                        # survived?
adb shell su 0 rm -f /system_tw/.zos-canary
```

If the canary survives the cache drop, `/system_tw/priv-app` is a viable home
for a platform-signed ZOS with no vendor involvement. **Note this still needs
root** (`adb root` / `su`), which this `user` build does not grant without a
Magisk flash - see `ROOT-AND-SYSTEM-APP.md`. So on a stock unit this route is
blocked at the root step, not the partition step.

## What is safe vs what bricks

| Action | Risk |
|---|---|
| `getprop`, `pm list`, `pm path`, pulling an APK to read its cert | **None** - read-only |
| Writing an APK to a `/data` or data-like vendor partition | Low - worst case the app does not run; no boot impact |
| `cmd package set-home-activity` to a `/data` app | **Medium** - this is what started the whole incident; always record prior HOME, never on a unit about to be reset |
| `pm uninstall` of a launcher or HOME app | **Medium** - can leave a home gap |
| Factory reset via the **AOSP** screen | **High** - the documented brick path on this platform; use `123456`/`7890` only |
| Flashing a modified/self-signed boot or system image to locked AVB | **CRITICAL** - the one action that bricks past KEY1 and past a same-image USB reflash |

**Rule of thumb:** anything that only touches `/data` or a scanned app partition
is recoverable. Anything that touches `misc`, the bootloader, or an AVB-verified
image is where units die. Stay on the first side of that line without a vendor
engineering image in hand.
