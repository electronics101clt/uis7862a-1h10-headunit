# Evidence

Raw material for the boot-loop episode of 2026-10-20/21, kept unedited so the
account in `docs/BOOTLOOP-RESOLVED.md` and `docs/FACTORY-RESET-POSTMORTEM.md`
can be checked against the record rather than taken on trust.

## `SESSION-TRANSCRIPT.md`

The complete Claude Code session, session id
`fcc19d5e-ec6a-4a2b-8378-f24714a0d725`. 680 entries, exported in order and in
full: every message from both sides, every tool invocation with its arguments,
and every tool result including command output.

**Nothing is summarised, reordered or edited.** The single substitution is
inline base64 image data, replaced by `[inline image data omitted - phone
photograph]`. Those images are the phone photographs of the board; what was read
out of them is transcribed in `docs/HARDWARE-ID-AND-RECOVERY.md`.

The original log is a 50 MB JSONL file, the bulk of which is that image data.
It remains on the owner's machine at
`~/.config/claude5pro3/projects/-home-jonathan/fcc19d5e-ec6a-4a2b-8378-f24714a0d725.jsonl`
if a byte-exact copy is ever needed.

### Why it is here

The session contains the decisions that led to the unit being bricked and the
ones that recovered it - including the point where the AOSP reset screen was
named and the vendor's gated path was not, and the point where the owner's
rejection of that tool call is recorded verbatim. It is kept so that account
cannot be rewritten after the fact.
