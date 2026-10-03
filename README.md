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
| `pitch` | 0.1.0 | YIN pitch detection and the note arithmetic |

`syslui` and `sdl3` are named directly although other coordinates bring them, because resolution
takes the highest version anybody asks for and the ones they ask for are older.

## How it listens

`program/tuner/pipeline.sysl` is the whole of the signal processing, and has no SDL in it:

- **48 kHz mono `f32`**, 70–1400 Hz — below a drop-D low string, above the highest fretted note.
- **A 2048-sample window** (43 ms). YIN needs two periods of the lowest frequency, 1374 samples at
  70 Hz; the next power of two leaves room and holds three and a half periods of a low E.
- **A detection every 1024 samples** — 47 readings a second, each sample read twice.
- **The median of the last five readings**, shown once three agree, with frames whose clarity is
  under 0.9 left out — so two bad frames never move the needle and one stray frame never names a note.
- **A one-second hold** after the last clear reading, then back to *play a string*.

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
