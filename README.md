# Tuner

A chromatic guitar tuner for Android, written in [sysl](https://sysl.sh) on
[Skitter](https://github.com/sysl-lang/skitter) and [syslUI](https://github.com/sysl-lang/syslui). It
listens to the microphone, names the nearest note in large type, and shows how far off it is on a
meter whose middle band — five cents either side — turns green when the string is in tune.

The same program runs on a Mac, where a synthesized tone can stand in for the microphone.

## What it is made of

| coordinate | version | what it does here |
|---|---|---|
| `skitter` | 0.1.0 | the Android activity, the Gradle build, the system bars |
| `syslui` | 0.1.3 | the interface — `meter`, and `.font_size` for the note name |
| `syslui-sdl` | 0.2.1 | the window and frame loop; `on_frame` is where the microphone is drained |
| `sdl3` | 0.3.2 | `open_recording_stream`, which also asks for the permission |
| `pitch` | 0.1.1 | YIN pitch detection, the high-pass filter ahead of it, and the note arithmetic |

`syslui` and `sdl3` are named directly although other coordinates bring them, because resolution
takes the highest version anybody asks for and the ones they ask for are older.

## How it listens

`program/tuner/pipeline.sysl` is the whole of the signal processing, and has no SDL in it:

- **48 kHz mono `f32`**, 70–1400 Hz — below a drop-D low string, above the highest fretted note.
- **A 65 Hz high-pass filter**, two sections, ahead of everything else — it takes out the rumble and
  the body resonance a phone or laptop microphone hears under a low E.
- **A 2048-sample window** (43 ms). YIN needs two periods of the lowest frequency, 1374 samples at
  70 Hz; the next power of two leaves room and holds three and a half periods of a low E.
- **A detection every 1024 samples** — 47 readings a second, each sample read twice — with YIN's
  threshold at 0.2.
- **The median of the last five readings**, shown once three agree, with frames whose clarity is
  under 0.8 left out — so two bad frames never move the needle and one stray frame never names a note.
- **A note that holds on**: once a note is shown it stays until the pitch is more than 60 cents from
  it, and the cents are read from that note — a string 55 cents flat of E shows *E −55* rather than
  flickering to *D# +45*.
- **A one-second hold** after the last clear reading, then back to *play a string*.

The threshold, the clarity and the filter were set against a real recording rather than a clean
tone: the YIN paper's 0.12 and a clarity of 0.9 read almost nothing from a real low E with a room's
rumble under it.

Everything is counted in samples, so the answer does not depend on how the audio arrives — a test
feeds the same second in random chunks, empty ones included, and gets the identical reading.

## Building and running on Android

You need the Android SDK (with `ANDROID_HOME` set, or `~/Library/Android/sdk`), a JDK from 17 to 25,
and sysl 0.0.158 or later.

```
./fetch-sdl3.sh
./gradlew assembleDebug
adb install -r app/build/outputs/apk/debug/app-debug.apk
adb shell am start -n sh.sysl.tuner/sh.sysl.skitter.SkitterActivity
```

`skitter run` does all four and follows the log. The first launch asks for the microphone; saying no
leaves the tuner up with *no microphone permission* on it.

### In the emulator, with the Mac's microphone

The emulator's microphone hears nothing until it is told to use the host's:

1. Open **Extended controls** (the `⋯` at the bottom of the emulator's toolbar).
2. **Microphone** → turn on **"Virtual microphone uses host audio input"**.
3. macOS may ask whether the emulator may use the microphone; allow it.

The setting does not survive a restart of the emulator.

## Running on a Mac

```
sysl run program
```

opens the microphone, which makes macOS ask on behalf of the terminal. To look at it without a
microphone, give it a tone instead:

```
TUNER_TONE=110.5 sysl run program
```

## Tests

```
sysl test program
```

runs the pipeline against synthesized strings: every open string reads its note within a cent, a tone
ten cents sharp reads +10, silence reads nothing, a stopped string is held and then cleared on
schedule, the median outvotes a burst of another pitch, and chunked delivery matches one big chunk.
Tones gliding between E2 and F2, or wavering around the 50-cent line between E2 and D#2, check that
the note shown switches only past 60 cents.

**`program/tuner/lowE.s16` is a real low E**: the user's own guitar, an unplugged electric, played
into a MacBook's built-in microphone — 9.5 s of 48 kHz mono, signed 16-bit little-endian, no header.
It carries the rumble below 65 Hz and a resonance at 80 Hz, louder than the string's fundamental,
that a real room and a real microphone put under the string. The suite feeds it through the pipeline
in 10 ms chunks: with the shipped settings E2 is shown for 73% of the chunks where the string sounds,
never another note, and the cents stay within a 16-cent spread; with the old ones nothing is shown.
