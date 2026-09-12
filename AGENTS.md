# AGENTS.md

Instructions for an AI coding agent setting this up on someone's Mac. Follow the
steps in order and verify each one — most of this document is failure modes,
because every one of them below cost real debugging time and none announce
themselves.

## What you are building

A button on a clip-on wireless mic that starts and stops dictation. The user
presses a button on their shirt, talks, presses again, and the text appears in
whatever window has focus. Because the transmitter is worn, pressing it is not
"touching the computer" — that is the entire point.

The chain is:

```
mic button -> HID consumer usage 0xE9 -> Karabiner rule -> F13 -> dictation app
```

Karabiner is the intermediary because the button is **not a keyboard key**. It
arrives as a USB HID *consumer control* usage — the same category as the volume
keys on a media remote — and no dictation app listens for those.

## Prerequisites

Check each before starting; stop and tell the user if one is missing.

1. **macOS.** This is macOS-only.
2. **[Karabiner-Elements](https://karabiner-elements.pqrs.org/)** installed, and
   granted Input Monitoring (System Settings → Privacy & Security → Input
   Monitoring → `karabiner_grabber`). Without it the receiver is invisible.
3. **A DJI wireless mic**, receiver plugged in, transmitter powered on and
   paired. Verified on the Mic Mini; the Mic Mini 2 reports the same USB IDs.
4. **A dictation app** — Handy (local, free) or Wispr Flow (cloud, paid).
5. **Python 3** — `djimic` uses only the standard library.

## Install

```sh
git clone https://github.com/conchoecia/dji-mic-button.git
cd dji-mic-button
./djimic detect
```

`detect` must list the receiver. Expected on a DJI Mic Mini:

```
[  ok  ] Wireless Mic Rx  vendor_id=11427 product_id=16401
      8 button usage(s) this device can send:
        0xE9  volume increment       karabiner: "volume_increment"
        ...
```

It lists eight usages because that is everything the receiver's descriptor
permits, not because eight buttons exist. The transmitter button is `0xE9`.

If it lists nothing, the receiver is not on the USB bus — have the user plug it
in or power it on. Powering on the *transmitter* alone is not enough.

Then bind the button. Use `f13` unless the user asks otherwise: F13–F19 exist in
the keyboard protocol but on no Mac keyboard, so nothing can collide with them.

```sh
./djimic install --hotkey f13 --usage 0xE9
./djimic doctor
```

`--usage 0xE9` skips the interactive probe. It is correct for every DJI mic
measured so far; drop it to have `djimic` identify the button empirically.
`doctor` should report all green, with the rule present in every Karabiner
profile.

## Configure the dictation app

### Handy

Handy stores settings in
`~/Library/Application Support/com.pais.handy/settings_store.json`. **Quit Handy
before editing that file** — it holds settings in memory and overwrites the file
on exit, silently discarding your changes.

Required settings:

| Key | Value | Why |
| --- | --- | --- |
| `keyboard_implementation` | `"tauri"` | **Mandatory.** See below. |
| `bindings.transcribe.current_binding` | `"f13"` | what the button sends |
| `selected_microphone` | `"Wireless Mic Rx"` | else it records the laptop mic |

**`keyboard_implementation` must be `tauri`.** This is the single most
important line in this document. Handy's macOS default is `handy_keys`, a
CGEventTap that **never receives keys emitted by Karabiner's virtual keyboard**.
The shortcut registers successfully, logs no error, and every press does
nothing at all — no log line, no reaction. `tauri` uses the system's real hotkey
registration and works immediately. Expect to lose an hour here if you skip it.

After any change, relaunch Handy and confirm in
`~/Library/Logs/com.pais.handy/handy.log`:

```
Registered ... shortcut: transcribe -> Hotkey { modifiers: Modifiers(0x0), key: Some(F13) }
```

### Wispr Flow

Wispr's hands-free toggle is internally called `popo`, in
`~/Library/Application Support/Wispr Flow/config.json` under
`prefs.user.shortcuts`. Keys are **macOS virtual keycodes** joined by `+`
(`49`=Space, `63`=fn, `53`=Escape, `55`=Command).

F13 is keycode **105**. Quit Wispr, then set `"105": "popo"` and remove the old
`popo` entry. Wispr re-serializes the file on quit and preserves the change, so
it accepts a bare F13 as a legitimate binding.

Editing the file is necessary rather than merely convenient: Wispr records
shortcuts by listening for a keypress, and **no Mac keyboard has an F13 key**.

### Running both

Both apps can be bound to F13 and will each answer when running. Keep **only one
open at a time** — with both running, both record and the user gets two pastes.

## Verify

Have the user press the button, say a few words, and press again. Do not try to
verify by synthesizing a keypress: `tauri`/Carbon hotkeys respond only to real
hardware events, so a synthetic F13 proves nothing either way.

Check `~/Library/Logs/com.pais.handy/handy.log` for a `Transcription completed`
line. Silence there means the key never arrived; see failure modes below.

## Known failure modes

**Nothing happens, no log output at all.** Almost always
`keyboard_implementation: handy_keys`. Set it to `tauri`.

**Worked, then stopped after editing a shortcut in Handy's UI.** Recording a
shortcut leaves a stale registration; Handy's real listener then fails with
`Hotkey already registered: F13` and gives up. **Restart Handy** after recording
any shortcut.

**Button dead after enabling or disabling post-processing.** Handy refuses to
register `transcribe_with_post_process` while `post_process_enabled` is false.
If F13 is bound to that action and post-processing is off, the button does
nothing. Keep the two consistent: F13 on `transcribe_with_post_process` with
post-processing on, or on plain `transcribe` with it off.

**`djimic install` fails with `manipulators is missing or empty`.** An old copy
that passes the whole `karabiner.json` to `--lint-complex-modifications`, which
rejects any config at all. Pull the current version.

**System audio is muted and nothing plays.** Handy mutes output while recording
and restores it afterwards; killed mid-recording, it never restores. Fix with
`osascript -e 'set volume output muted false'`.

**The receiver's own button does nothing.** Correct — it emits no HID at all.
Only the transmitter's button works.

## Notes for the agent

- Prefer `djimic doctor` over reading `karabiner.json` yourself; it checks the
  receiver, every profile, and the `fn` key setting in one pass.
- `djimic` backs up `karabiner.json` before every write and rolls back if
  Karabiner rejects the result. You do not need to back it up yourself.
- The rule is written into **every** Karabiner profile by default, because a
  rule in one profile silently stops working the moment the user switches
  profiles — an annoying thing to debug.
- Only the user can press the physical button. Ask them to test rather than
  attempting to simulate it.
