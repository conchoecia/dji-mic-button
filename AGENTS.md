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

Don't hand-edit the settings file. Run the script — it applies every setting
below, installs the paste helper, and verifies the result:

```sh
./contrib/setup-handy            # configure, then verify
./contrib/setup-handy --check    # report only, change nothing
```

It quits Handy before writing (Handy keeps settings in memory and overwrites the
file on exit, silently discarding anything you changed while it ran), backs the
file up, relaunches, and re-reads it to confirm. Exit status is 0 only when
everything checks out, so an agent can branch on it.

What it sets:

| Key | Value | Why |
| --- | --- | --- |
| `keyboard_implementation` | `"tauri"` | **mandatory** — see below |
| `bindings.transcribe.current_binding` | `"f13"` | what the button sends |
| `selected_microphone` | the DJI receiver | else it records the laptop mic |
| `paste_method` | `"external_script"` | hands the text to the paste helper |
| `external_script_path` | installed helper | conditional "press enter" submit |
| `auto_submit` | `false` | else Return gets pressed twice |

It binds plain `transcribe`, not `transcribe_with_post_process`: Handy refuses
to register the latter while post-processing is off, which kills the button with
no indication.

**`keyboard_implementation` must be `tauri`.** This is the single most
important line in this document. Handy's macOS default is `handy_keys`, a
CGEventTap that **never receives keys emitted by Karabiner's virtual keyboard**.
The shortcut registers successfully, logs no error, and every press does
nothing at all — no log line, no reaction. `tauri` uses the system's real hotkey
registration and works immediately. Expect to lose an hour here if you skip it.

**How to verify, and what not to look for.** The `tauri` backend writes *no*
registration line at startup — searching for one and failing to find it proves
nothing. It logs the hotkey only when the key actually fires:

```
tauri global-shortcut event: binding=transcribe, shortcut=F13, state=Pressed
```

So silence in `~/Library/Logs/com.pais.handy/handy.log` means "not pressed yet",
not "broken". `setup-handy --check` encodes that distinction: it reports a
failure only for an actual `could not register` error.

**Conditional submit.** `contrib/handy-press-enter` receives the transcript and
presses Return only when the dictation ended with the words "press enter",
stripping the phrase first. Saying nothing else just pastes. This is a copy of
Wispr Flow's behaviour; Handy's own `auto_submit` submits after *every*
dictation, which is why the script turns it off. Use `contrib/handy-timing` to
see per-stage latency (transcription / LLM / paste) if anything feels slow.

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

Run the configuration check first — it is the cheap half and needs no hardware:

```sh
./contrib/setup-handy --check    # exit 0 when everything is right
```

Then have the user press the button, say a few words, and press again. Do not
try to verify by synthesizing a keypress: `tauri`/Carbon hotkeys respond only to
real hardware events, so a synthetic F13 proves nothing in either direction and
invites a wrong conclusion. Only the user can press the button.

Check `~/Library/Logs/com.pais.handy/handy.log` for a `Transcription completed`
line. Silence there means the key never arrived; see failure modes below. The
paste helper keeps its own log at `~/Library/Logs/handy-press-enter.log`, which
splits the two halves cleanly: an entry there means transcription worked and the
problem is in the paste, and no entry means it never got that far.

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
