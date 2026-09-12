# dji-mic-button

Find out what the button on a DJI wireless mic actually sends, then bind it to
any macOS hotkey with Karabiner-Elements.

Clip the mic on, walk away from the desk, tap the button, talk. Your dictation
app starts and stops without you touching the keyboard.

## The problem

The button on a DJI Mic is not a keyboard key. The receiver reports it as a USB
HID *consumer control* usage — the same category as the play/pause and volume
keys on a media remote — and **which usage it sends is not documented
anywhere**. Existing projects hardcode a guess that happened to work on the
author's unit.

This tool doesn't guess. It reads the receiver's HID report descriptor to
enumerate every usage the hardware is physically capable of sending, then
identifies the real one empirically: it temporarily maps each candidate to a
distinct word, you press the button, and the answer types itself.

## Quick start

```sh
git clone https://github.com/conchoecia/dji-mic-button.git
cd dji-mic-button
./djimic detect      # what's plugged in, and what it can send
./djimic install     # probe the button, then bind it
```

`install` walks you through the probe and writes the rule. That's the whole
setup. Then point your dictation app at the same hotkey.

Requires macOS and [Karabiner-Elements](https://karabiner-elements.pqrs.org/).
No other dependencies — it's one Python 3 file using only the standard library.

## Commands

| command | what it does |
| --- | --- |
| `djimic detect` | List attached DJI devices and every usage each can send |
| `djimic probe` | Identify the button by pressing it |
| `djimic install` | Probe, then bind the button to a hotkey |
| `djimic doctor` | Check the whole setup end to end |
| `djimic uninstall` | Remove the rule from every profile |
| `djimic restore` | Roll back to the most recent backup |

Useful flags:

```sh
djimic install --hotkey f13              # default is fn+space
djimic install --usage 0xE9              # skip the probe, you already know
djimic install --profiles current        # default writes to all profiles
djimic --config /tmp/test.json install   # dry-run against a copy
```

## Choosing a hotkey

`--hotkey` takes `+`-separated parts, e.g. `fn+space`, `ctrl+opt+cmd+d`, `f13`.
Modifiers understood: `fn`, `cmd`, `opt`/`alt`, `ctrl`, `shift`. Anything else
is passed to Karabiner as a key name.

**If you might change dictation apps, target `f13`.** F13–F19 exist in the
keyboard protocol but on no Mac keyboard, so nothing can collide with them.
Bind F13 in whatever app you use today; switch apps later and you rebind
*in the app*, never touching Karabiner again.

`fn+space` is the default because it's Wispr Flow's built-in hands-free
shortcut, so it works with zero configuration on the app side. It's unclaimed
by macOS — `Cmd+Space`, `Ctrl+Space`, `Ctrl+Opt+Space` and `Cmd+Opt+Space` are
all taken, but nothing uses `fn+space`. One caveat: `fn` is an unusual
modifier and some apps ignore the flag, so if your dictation app *isn't*
holding the hotkey globally, `fn+space` can fall through as a plain space.
`djimic doctor` warns if your `fn` key is set to start macOS dictation.

## Pick your trigger shape

A button press is momentary — down and up in one motion. That makes it a good
fit for a **toggle** ("press to start, press again to stop") and a poor fit for
**push-to-talk** ("hold to talk"), which would only ever record a blip. Bind
the mic button to your app's toggle-style shortcut. In Wispr Flow that's
*Hands-free mode*, not *Push to talk*.

## Submitting the prompt

The button starts and stops dictation. It does not *send*, which is the thing
you actually want when you're talking to a CLI agent and the prompt has to run.
What's available depends on the dictation app, so here is what each one really
does.

**Wispr Flow: say "press enter" as the last words of the utterance.** The
phrase is stripped from the text and Enter is simulated once the paste lands.
The first time you use it you get a notification asking to turn the feature on;
after that it is `pressEnterCommandEnabled` in
`~/Library/Application Support/Wispr Flow/config.json`.

**Wispr Flow has no always-submit setting.** Measured against 1.6.827, the
whole feature is one regex

```
/([.!?;])?[\s]*\bpress enter\s*([.!?,;:-]*)\s*$/i
```

matched against the formatted text, falling back to the raw ASR text when the
LLM strips the phrase during formatting. Nothing else in the app sets
`shouldPressEnterAfterPaste`.

The `$` is the part that bites. The phrase has to end the utterance, so
anything arriving after it kills the match — and you get no Enter *and* the
words "press enter" left in your text. Another voice in the room is enough: one
observed failure was a bystander saying "under" as the dictation closed,
yielding "…press enter under", which matches nothing.

**Handy: Settings → Advanced → Auto Submit.** Off / Enter / Ctrl+Enter /
Cmd+Enter, sent after every paste with no phrase involved — see
`should_send_auto_submit` in `src-tauri/src/clipboard.rs`, measured against
0.9.6. The only condition is that the paste method isn't `None`. If you want
the mic button to submit every time without thinking about it, this is the one
that does it, and it needs no daemon and no accessibility grant of ours: bind
Handy to `f13` and run `djimic install --hotkey f13`.

## Open mic

**None of them listen on their own.** Every one needs a press to start.

- **Wispr Flow** tracks silence only to diagnose a dead-microphone bug — it
  logs `Audio silence bug detected` and never stops the recording on its own.
  "Hey Flow" is not a wake word: it is a phrase matched *inside* an
  already-running dictation to route that utterance into Command Mode.
- **Handy** runs Silero VAD, but the VAD filters silence out of the recording
  rather than ending it; `VadPolicy` is `Disabled` / `Offline` / `Streaming`,
  all frame-filtering profiles. `always_on_microphone` is "keep microphone
  active for faster response" — it holds the audio stream open to cut startup
  latency, not hands-free capture. All three activation modes (`Toggle`,
  `PushToTalk`, `HoldOrToggle`) are key-driven.
- **superwhisper** documents its auto-send as "Hold Shift to Auto-Send", a
  modifier you have to hold, so its submit step isn't touch-free either.
  (Secondhand write-ups describe a configurable auto-stop-after-silence; that
  is not in the settings docs and is unverified here. Unlike the two findings
  above, this one is read, not measured.)

For this tool that gap matters less than it looks, because the transmitter
clips to your shirt. Pressing *it* is not touching the computer, so a worn
button plus Handy's Auto Submit is hands-free end to end with no phrase to
mishear.

Handy is the one worth watching: it already has both halves — a VAD that knows
when you stopped talking, and an unconditional submit. "End the recording after
N seconds of silence" is the only missing piece.

## What it writes

Exactly one thing: a complex modification in `~/.config/karabiner/karabiner.json`,
scoped with `device_if` so it only fires for the DJI receiver and leaves the
volume keys on your other keyboards alone.

```json
{
  "description": "DJI mic button (volume increment) -> fn+space  [djimic]",
  "manipulators": [{
    "type": "basic",
    "conditions": [{"type": "device_if",
                    "identifiers": [{"vendor_id": 11427, "product_id": 16401}]}],
    "from": {"consumer_key_code": "volume_increment",
             "modifiers": {"optional": ["any"]}},
    "to": [{"key_code": "spacebar", "modifiers": ["fn"]}]
  }]
}
```

No daemon, no launch agent, no background process, nothing outside that file.
Every write is backed up first (`karabiner.json.djimic-backup.<timestamp>`),
linted with `karabiner_cli`, and rolled back automatically if Karabiner rejects
it.

By default the rule goes into **every** Karabiner profile. A rule in only one
profile silently stops working the moment you switch profiles, which is an
annoying thing to debug.

## Findings

Measured on a DJI Mic Mini (receiver: `Wireless Mic Rx`, vendor `11427`/`0x2CA3`,
product `16401`/`0x4011`):

- **The transmitter button sends consumer usage `0xE9` — `volume_increment`.**
- **The receiver's own button sends no HID at all.** There is nothing to bind
  there; don't waste time on it.
- The receiver's descriptor allows only these eight, all on Report ID 6:
  `volume_increment` (0xE9), `volume_decrement` (0xEA), `mute` (0xE2),
  `play_or_pause` (0xCD), `scan_next_track` (0xB5), `scan_previous_track`
  (0xB6), `pause` (0xB1), `stop` (0xB7). Its Telephony collection is declared
  entirely Constant — padding, not real data — and usage pages `0xFF90` /
  `0xFF82` are DJI's own firmware channel, not keystrokes.
- The original **Mic Mini and the Mic Mini 2 report the same USB IDs**, so a
  rule written for one works on the other.

Two Karabiner gotchas worth recording: `pause` and `stop` are **not** valid
`consumer_key_code` strings even though those words appear inside the Karabiner
binary — they must be given as the raw integers `177` and `183`. And this has
to be a *complex* modification: simple modifications can't emit a modifier
chord, and can't express `modifiers: {optional: ["any"]}` on the `from` side.

## Troubleshooting

**"I don't see it in Simple Modifications."** You won't, ever. Look under
**Settings → Complex Modifications**. The receiver itself appears under
**Settings → Devices** as *Wireless Mic Rx*.

**The probe types nothing.** Karabiner isn't seeing the device. Check that the
receiver is listed under Settings → Devices, and that Karabiner has Input
Monitoring permission (System Settings → Privacy & Security → Input
Monitoring → `karabiner_grabber`).

**The button fires two different hotkeys.** Another rule for the same device is
still installed. `djimic install` removes any rule bound to the receiver and
tells you what it displaced, so re-running it fixes this.

**It worked, then stopped.** You probably switched Karabiner profiles. Run
`djimic doctor` — it flags profiles that are missing the rule.

## Prior art

- [caezium/dji-mic-wispr-flow](https://github.com/caezium/dji-mic-wispr-flow) —
  the shell installer this grew out of, for the Mic Mini 2 and Wispr Flow.
- [Johnixr/dji-mic-dictation](https://github.com/Johnixr/dji-mic-dictation) —
  same idea targeting macOS dictation and Typeless.

Both hardcode the usage. This one measures it, works with any hotkey and any
app, and cleans up after itself.

## License

MIT
