# DoFun Telegram outreach - draft

Send to **@dofunapp** (`t.me/dofunapp`). The channel `t.me/dofun_app` invited
"system customization, HEADUNIT brand customization" directly to this handle.

Guidance before sending:
- Lead with the unit codes - proves you are a real integrator, not a tyre-kicker.
- **Do not send ZOS (source or APK) in first contact.** Question 3 learns whether
  they even need it before you hand anything over.
- The MOQ answer decides the whole plan. If it is hundreds of units, switch to
  buying stock and installing at the bay.
- **Leave John out.** Raise signing as a capability; raise the name only after a
  deal is in motion (see PLATFORM-AND-VENDORS.md).

---

## The message

Hello — I saw that you handle system customization and head unit brand
customization, and I'd like to produce custom-branded units.

I'm ZScreen Electronics, a mobile automotive installer in the US (Charlotte,
NC). I have my own Android launcher and app suite already running on the TS10S
platform, and I want to ship units under my own brand with my software
preinstalled, backed by US-based install and support.

To understand your process, a few questions:

1. What is the minimum order quantity for brand customization?
2. Can you build a system image with my launcher preinstalled as a system app,
   plus my own boot animation and app selection?
3. Would you platform-sign my app, or provide an engineering build plus signing
   so I can sign it myself? (My current units are platform-signed with your key,
   not AOSP test-keys, so I can't self-sign without you.)
4. Which hardware models can this be done on? I'm currently on TS10S / UIS7862A,
   13.3" — board AW4C-0432-47 / core AW4B-0432-4A, firmware
   TS10S.2.1_20240928.091908.
5. Per-unit pricing at the minimum quantity, and lead time.

Happy to share more about my software once I understand how you work.

Thanks.

---

## My own plan (the exit strategy this conversation serves)

Where this is meant to go, so the message above is pointed at the right outcome:

1. **Near term — install business as-is.** Keep fitting and supporting Android
   head units in Charlotte (ZScreen Electronics, zscreenusa.com). ZOS goes on at
   the bay. No vendor dependency; revenue today.
2. **Mid term — branded units, my software, their hardware.** Buy stock TS10S
   units in volume if MOQ for full customization is too high; install ZOS and
   brand them myself. Units arrive generic, leave as ZScreen units with US
   support — the honest differentiator over a box shipped from Shenzhen with no
   backing.
3. **Longer term — a true rebrand line.** A vendor-built image with ZOS as a
   system app, my boot animation, my app selection, platform-signed. This is the
   DoFun conversation above. Gated by MOQ, whether they'll sign without taking my
   source, and per-unit economics.
4. **Protect the software throughout.** Never hand over ZOS source. Prefer the
   "install it myself" shapes until a signing arrangement exists that does not
   require giving them the code. Assume anything sent to the vendor is kept.
5. **The signing dependency is the real hinge.** This platform's key is TopWay's
   own (CN=John), not the public AOSP test-key, so a system-app rebrand cannot be
   done without them — unlike the Jancar. That single fact is why the vendor
   relationship matters at all; everything short of it I can do alone.
