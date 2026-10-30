# Mizmar Sampler (VST3) — Oriental/Arabic solo sampler

A monophonic, legato, microtonal sampler in C++ / JUCE. First instrument: Egyptian mizmar.

> **Status:** the source was written without access to a Windows machine or the JUCE
> sources, so it has **not been compiled or tested in FL Studio yet**. Expect to fix a few
> compiler errors on the first build. Paste them to me and I'll fix them.

---------------------------------------------------------------------------

## 1. Build (Windows, VST3)

**Requirements**

- Visual Studio 2022 (workload *Desktop development with C++*)
- CMake 3.22+
- Git (CMake downloads JUCE 8.0.4 automatically the first time)

**Command line (x64 Native Tools / any terminal)**

```bat
cd MizmarSampler
cmake -B build -G "Visual Studio 17 2022" -A x64
cmake --build build --config Release
```

Output: `build\MizmarSampler_artefacts\Release\VST3\Mizmar Sampler.vst3`
(and a `Standalone` .exe, handy for testing without a DAW).

To use an existing JUCE checkout: `cmake -B build -DJUCE_PATH=C:/dev/JUCE ...`

**Install in FL Studio**

1. Copy `Mizmar Sampler.vst3` to `C:\Program Files\Common Files\VST3\`
   (needs admin; or set `COPY_PLUGIN_AFTER_BUILD TRUE` and run the IDE as administrator).
2. FL Studio: *Options > Manage plugins > Find plugins* (or *Find installed plugins*).
3. Add it as an instrument: *Add > Mizmar Sampler*. Notes come from the piano roll.

JUCE is AGPLv3 / commercial. If you ever distribute the plugin, check JUCE's license terms.

---------------------------------------------------------------------------

## 2. First run (milestone 1: one sample, legato, 12 buttons)

1. Copy `ExampleData\Instruments\Mizmar` to
   `%APPDATA%\MizmarSampler\Instruments\Mizmar`
   (the plugin creates `%APPDATA%\MizmarSampler\` on first start).
   It contains a synthetic looped placeholder tone, so you can test everything before you have recordings.
2. Open the plugin; a button "Mizmar" appears at the top. Click it if it isn't already selected.
3. Play two overlapping notes with **Legato on**: no new attack, the pitch glides.
   Turn Legato off: every note re-attacks.
4. Toggle **E** and **B**: those notes are now a quarter tone (50 cents) lower in every octave.

Use *Folder...* in the plugin to point it at a different instruments folder (saved with the project).

---------------------------------------------------------------------------

## 3. Adding a new instrument

```
<instruments root>\                     default: %APPDATA%\MizmarSampler\Instruments
    Mizmar\                             <- folder name = button name
        instrument.json                 <- required
        mizmar_C3.wav
        mizmar_E3.wav
        mizmar_G3.wav
    Ney\
        instrument.json
        ney_D4.wav
```

Press *Rescan* in the plugin after adding a folder.

### instrument.json

```json
{
  "name": "Mizmar",
  "zones": [
    { "file": "mizmar_C3.wav", "rootNote": "C3", "lowKey": 46, "highKey": 49,
      "loop": true, "loopStart": 22050, "loopEnd": 88200, "gainDb": 0 },
    { "file": "mizmar_E3.wav", "rootNote": 52 }
  ]
}
```

| field | meaning |
|---|---|
| `file` | WAV filename relative to the instrument folder (mono or stereo, any sample rate / bit depth) |
| `rootNote` | note at which the sample plays unpitched. Number (60) or name (`"C4"`, `"F#3"`, `"Bb2"`); **C4 = 60** |
| `lowKey`, `highKey` | optional key range (both needed). If omitted, the keyboard is split halfway between neighbouring roots |
| `loop`, `loopStart`, `loopEnd` | loop points in **sample frames**; `loopEnd` is exclusive. Needed for sustained notes. Without a loop the sample plays once and stops |
| `gainDb` | per-sample level trim |

Recording tips for the mizmar:

- Record a long, steady sustained note (3–6 s) per sample; pick loop points in a stable part,
  on zero crossings with matching waveform phase, in an audio editor (Reaper, Audacity...).
- One sample every 2–3 semitones is plenty. Pitches between samples are made by resampling
  (cubic interpolation), and quarter tones are also just resampling, so don't record them.
- Legato: when the next note maps to a *different* sample, the plugin crossfades to the new
  sample (starting at its `loopStart`, so the new attack is skipped). Looped samples are therefore recommended.

---------------------------------------------------------------------------

## 4. Microtonal system and maqamat

- 12 buttons (C...B): when ON, every MIDI note with that pitch class is detuned by **-Tune Amount** cents.
- **Tune Amount**: 0–100 cents, default 50 (quarter tone). Try 30–45 for other intervals.
- **Maqam preset** dropdown: choosing a maqam switches the matching buttons on. The preset's
  cent values replace the plain "-Tune Amount" for those notes, scaled by `Tune Amount / 50`
  (Tune Amount 50 = exactly the table; 40 = 80 % of it).
  Choosing *Manual* leaves the buttons as they are.
- Changing buttons / Tune Amount while a note is held glides the pitch (10 ms) instead of clicking.

Maqam table: `%APPDATA%\MizmarSampler\Maqamat\maqamat.json` (created on first run, edit freely,
restart the DAW/plugin to reload). 12 values = cents for C, C#, D, D#, E, F, F#, G, G#, A, A#, B.

```json
{ "maqamat": [ { "name": "Rast", "offsets": [0,0,0,0,-50,0,0,0,0,0,0,-50] } ] }
```

The built-in tables assume the usual tonic (Rast on C, Bayati/Saba/Hijaz/Kurd on D,
Sikah on E half-flat, Nahawand on C) and use a flat 50 cents. They are starting points;
adjust them to your own ear and tuning (e.g. -35 for a "Cairo" half-flat).
New names are appended to the dropdown. Don't reorder/rename existing entries once you have
projects using them (the dropdown is stored as an index).

---------------------------------------------------------------------------

## 5. Controls

| control | effect |
|---|---|
| Legato | on: overlapping notes glide, no re-attack; releasing a key returns to the previous held key |
| Glide | portamento time (5–300 ms, default 40) |
| Vib Speed / Vib Depth | vibrato LFO 0–10 Hz, depth 0–100 cents |
| Mod wheel (CC1) | adds up to 50 cents of vibrato depth on top of Vib Depth |
| Bend Amount | pitch-wheel range, 0–12 semitones (default 2) |
| ADSR | envelope |
| Gain / Pan | output level (-36...+12 dB) / equal-power pan |

Final pitch (cents) = note*100 + quarter-tone offset (glided) + pitch bend + vibrato.

All controls are APVTS parameters (automatable). State = all parameters + toggles + selected
instrument + instruments folder.

---------------------------------------------------------------------------

## 6. Code map

| file | role |
|---|---|
| `PluginProcessor` | parameters (APVTS), state, MIDI splitting, background loading + lock-free hand-off |
| `SamplerEngine` | mono last-note voice, legato/glide, crossfade, ADSR, vibrato, Hermite resampling |
| `MicrotonalManager` | pitch-class -> cents offset (toggles + Tune Amount + maqam table) |
| `PresetManager` | instrument folder scan, JSON + WAV loading, maqam table |
| `PluginEditor`, `OrientalLookAndFeel.h` | resizable dark-red UI |

Real-time safety: `processBlock` only reads atomics, runs the engine and does arithmetic.
WAV decoding and JSON parsing run on a loader thread; the finished instrument is published with a
single atomic pointer store, and the last few instruments are kept alive so the audio thread can never
touch freed memory. Sample-rate changes are handled by `prepareToPlay` (rates are computed per
sample from file rate / host rate); block size is irrelevant because processing is per-sample.

## 7. Known limitations / next steps

- Switching instrument while a note sounds cuts the old sound instantly (tiny click).
- No crossfade at loop points: choose good loop points (or add a loop crossfade later).
- Velocity is mapped linearly to level; no velocity layers or round-robin yet.
- Maqam file is read once at plugin load.
- The instrument button grid has no scrolling: with very many instruments, enlarge the window.
