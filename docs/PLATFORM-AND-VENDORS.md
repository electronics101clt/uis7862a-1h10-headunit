# Who actually makes this unit, and who to talk to

Researched 2026-10-21. Graded: **established** where a source states it plainly,
**inferred** where it is a reading of evidence.

## The stack

```
Unisoc UIS7862A      silicon
TopWay               the board - the same motherboard in every one of these,
                     whatever name is on the box
TopWay ROM base      /system_tw, com.tw.* - "tw" is TopWay
DoFun                the UI and settings layer: com.dofun.carsetting,
                     DoFunPlay, iLauncher (1000+ themes)
<retail badge>       the box, the listing, the warranty
```

**Retail names riding this exact stack:** Topway, Oknavi, EKIY, Hikity, Navifly
(Brazil), Junsun, Eonon, TOPNAVI, Sky Universe, Carditai, yeanav, Kufine,
Navi-IT, Just Navi, ALWINNER. XDA's summary: *"All devices using the TS10/TS18
platform have the same motherboard from Topway, regardless of the store name or
brand where you purchased the head unit."* Navifly is the clean example - sold
under that name in Brazil with nothing in the firmware mentioning it.

TS10/TS18 are also attributed to **Shenzhen Driving Control Technology** as
developer, with TopWay as the board manufacturer.

### Correction: "tw" is TopWay, not ThunderSoft

`README.md` previously recorded the ROM base as ThunderSoft. **That was an
unverified inference and it is probably wrong.**

| Claim | Status |
|---|---|
| TS = TopWay (TopWay Series) | **Established** - stated directly across XDA |
| `tw` appears throughout the ROM | **Established** - `/system_tw`, `com.tw.*`, `TWCore`, firmware variants `TW1`, `TW11` |
| TW = TopWay | **Inferred**, and the natural reading: it is TopWay's own firmware base |
| TW = ThunderSoft | **Unsupported.** ThunderSoft would abbreviate TS or THS, and TS is already TopWay's |

**A note on how that error spread.** When searching to check this, the top
result for the relevant terms was *this repository*, and the "evidence" returned
for ThunderSoft was this repo's own README line. An unverified claim had been
published, indexed, and was being fed back as independent confirmation. Worth
remembering before writing a guess down as a fact.

## `john@tw-desktop` - what it is

The USB-debugging password hardcoded in the vendor Settings app
(`AdbPreferenceController`). It is a shell prompt: user `john`, host
`tw-desktop`.

| Observation | Reading |
|---|---|
| `user@host` prompt format | Linux/Unix. Windows shows `C:\path>`; macOS defaults differently |
| `-desktop` suffix | Ubuntu's **ubiquity** installer generates `<username>-desktop` from the username typed at install |
| So the install username was `tw` | The machine was provisioned under a **company account**, not a person's |
| `john` is a separate login on it | A shared or reassigned workstation |
| Newer Ubuntu uses `<username>-<hardware model>` instead | `-desktop` points at an **older Ubuntu**, i.e. a long-lived build box |

So: a long-lived TopWay build machine on an older Ubuntu, set up under a `tw`
account, with `john` as a login. Someone typed their own shell prompt into a
password constant and it shipped.

**It is not malware and not an easter egg.** It is a hardcoded credential - a
string comparison gating a settings toggle. XDA treats it as a known factory
default ("not a personal secret"); a `john@tw-desktop3` variant is reported on
some units. There is no John to contact and nothing further recoverable from the
string.

**Why it matters anyway:** one password unlocks USB debugging on *every* unit
built on this ROM base. A weak lock, not a thief - but it tells you the
engineering posture of the platform.

**Still unchecked:** `ro.build.user` and `ro.build.host`. AOSP embeds the build
machine's username and hostname in the ROM. If those read `john` and
`tw-desktop`, he built the firmware rather than merely editing one file.

## Who to contact

| Party | For | Contact |
|---|---|---|
| **DoFun** | Software layer: launcher in the image, boot animation, settings branding. They advertise "system customization, HEADUNIT brand customization" | Telegram channel `t.me/dofun_app` (10,791 subscribers); direct `@dofunapp` / `t.me/dofunapp`; email `market@dofun.shop`; shops at `doufun.com` and `dofun.shop`; Facebook "DofunApp" |
| **TopWay / assembling factory** | Hardware, and minimum order quantity | Not a public channel; ask DoFun who to approach |

DoFun has no Amazon presence because they do not sell to consumers - they sell
the software layer to the people who do. "HEADUNIT brand customization" is
literally the last line of the stack above: it is the step every one of those
retail names already took.

**Unknown and worth asking first:** their MOQ. No public terms exist anywhere -
not on Alibaba listings, not in the channel.

## SECURITY: this platform's updater has been hijacked before

**The Hacker News, August 2026** - the **MoYu Group** (previously behind the
BADBOX botnet) compromised DoFun head-unit updaters.

- Abused the legitimate system app **`com.tw.core` (TWCore)**, which exists for
  analytics and updates.
- Delivered malicious APKs disguised as firmware updates via an **MQTT broker on
  a `cardoor.cn` subdomain**.
- Payload: ad fraud and a proxy botnet. No user interface at all. Nine commands
  including device fingerprint exfiltration (resolution, model, Wi-Fi details,
  MAC) and `loadlib2` to pull further modules.
- Researchers called it the first documented malware with an infection chain
  specific to car head units.

**Both named components exist on this unit:**

```
/system_tw/priv-app/TWCore/TWCore.apk              -> com.tw.core
/data/app/.../cn.cardoor.d3dview-.../base.apk      -> cn.cardoor.d3dview
```

**This is exposure, not infection.** TWCore is the vendor's own legitimate app
and `cardoor.cn` is vendor infrastructure; the attack abused them rather than
installing them. Nothing observed on this unit indicates compromise.

**Consequence:** do not take firmware from untrusted sources on this platform.
This is the concrete reason to prefer a checksummed image from DoFun over the
community mirror recorded in `HARDWARE-ID-AND-RECOVERY.md`.

### Host machine checked

The owner's Linux workstation was swept for the published indicators -
`cardoor.cn`, `jarservice`, `loadlib2` - across scripts and configs, plus
systemd user units, autostart entries and cron. **Nothing found.** The only APKs
on the machine are the owner's own ZOS rollback files. The malware is Android
ARM code and cannot execute on an x86 Linux desktop in any case.

### The same thing happens to anyone who ships firmware

`john@tw-desktop` is structurally identical to this machine's own prompt:

```
john@tw-desktop                    a TopWay build box
jonathan@jonathan-MacBookPro12-1   this workstation, Ubuntu 24.04.3
```

Both are shell prompts. The only difference is what the hostname was derived
from: this machine was installed under the username `jonathan`, so Ubuntu
generated `jonathan-<hardware model>`; that one was installed under `tw`, the
company abbreviation, so the older installer generated `tw-desktop`. One machine
belongs to a person, the other belonged to a company before it belonged to
whoever sat at it.

**What that means for shipping our own units.** AOSP stamps the build machine
into every ROM it produces:

```
ro.build.user     the username that ran the build
ro.build.host     the hostname of the machine
```

If ZOS ever ships preinstalled in a vendor image, whatever account and machine
compiled it is published in that image, on every unit, permanently - exactly as
`john@tw-desktop` was. So:

1. **Build from a purpose-named machine and account**, not a personal laptop.
   `zscreen-build` beats `jonathan-MacBookPro12-1` on every unit sold.
2. **Never put a real string in a placeholder.** The password in this platform's
   Settings app is a developer's own shell prompt, typed because it was on
   screen. It is now a universal credential across five hardware generations and
   a decade of firmware, and the person who typed it had no say in that.
3. **Audit for hardcoded constants before handing an image to a vendor** - the
   firmware outlives the decision.

The lesson is not about one engineer. It is that anything left in a shipped
image is permanent, public, and attributable.

## "John" - the settled reading

Combining the public deep-research pass with the certificate read off this unit,
the most defensible account:

- **John is a work name, not a mystery.** `CN=John, O=TW, L=Shenzhen,
  ST=Guangdong` on the ROM's platform signing cert. Export-facing Chinese firms
  routinely have engineers adopt a Western given name for overseas business, so
  "John" is almost certainly a Shenzhen engineer's work alias - on the key that
  signs TopWay's firmware. The cert's location fields match TopWay's actual
  location (Shenzhen, Guangdong), which is what a genuine self-generated signing
  key looks like.

- **The cert ships on every unit.** Every `/system_tw` system app is signed with
  that key, so the certificate is embedded in every APK, on every TopWay unit
  worldwide. It is a build byproduct, not anything specific to this unit.

- **A public-records search finds no person**, and that is consistent rather than
  contradictory: a work alias on a signing cert leaves no LinkedIn/GitHub/press
  trail. (An independent deep-research pass reached "not a real person / default
  credential, high confidence" - correct on web evidence; it simply could not
  see the cert, which is in the firmware, not online.)

- **`tw-desktop` is a dead address, not an attack surface.** It was an Ubuntu
  build machine's hostname at some point (installer default under a `tw`
  account), almost certainly long since renamed, reimaged or retired. The
  hostname was only ever local to that build box - it never pointed anywhere
  routable. There is nothing to SSH to; the string is a fossil that got promoted
  from a machine name to a password constant.

**Net:** "John" is a real engineer behind TopWay's signing, reachable through no
public channel. Worth keeping in mind only as context if the name surfaces
naturally in a vendor conversation - not something to pursue or act on.

## Starting the brand-customization conversation (DoFun Telegram)

The commercial route, through `@dofunapp`. Lead with the unit codes, ask the
gating questions, and do not send ZOS (source or APK) in first contact - find
out whether they even need it first. The MOQ answer decides everything: if it is
hundreds of units, the plan shifts to buying stock units and installing ZOS at
the bay instead.

Questions that gate the deal:
1. Minimum order quantity for brand customization.
2. Can they build an image with the launcher preinstalled as a system app, plus
   boot animation and app selection.
3. Would they platform-sign the app, or provide what is needed to self-sign
   (this unit's platform key is TopWay's own, **not** the AOSP test-key - see
   `BUILDING-YOUR-OWN-IMAGE.md`, so self-signing needs their cooperation).
4. Which hardware models - currently TS10S / UIS7862A, 13.3",
   AW4C-0432-47 / AW4B-0432-4A, build TS10S.2.1_20240928.091908.
5. Per-unit price at MOQ and lead time.

Leave John out of the opener - it reads as odd to a sales contact.
