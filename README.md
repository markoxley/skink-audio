# skink-audio

Pure-Skink audio processing: PCM buffers, WAV codec, DSP, FFT spectrum
analysis, synthesis, and subprocess playback. No external libraries —
all DSP math runs in Skink; playback shells out to whatever system
player exists.

## Install

```
scales add markoxley/skink-audio
```

## Quick start

```skink
import "mod/markoxley/skink-audio/au_pcm"
import "mod/markoxley/skink-audio/au_synth"
import "mod/markoxley/skink-audio/au_wav"
import "mod/markoxley/skink-audio/au_play"

// synthesize a 440Hz sine, 1 second, 16kHz mono
b := au_synth.Tone(au_synth.WAVE_SINE, 440.0, 1.0, 16000, 0.8)

// save it
au_wav.Write("tone.wav", &b)

// play it through aplay/paplay/ffplay/afplay/play
au_play.Play(&b)
```

## Module layout

| Module | Contents |
|---|---|
| `au_pcm` | `Buffer` (interleaved float64, normalized ±1), S16 codecs, mono/stereo conversion, peak/RMS |
| `au_wav` | RIFF/WAVE PCM16 `Encode`/`Decode`/`Read`/`Write` — tolerant chunk parser |
| `au_dsp` | `Gain`, `GainDb`, `Normalize`, `Mix`, `Reverse`, `FadeIn`/`FadeOut`, `Echo`, `Lowpass`, `Highpass`, `Resample`, `Clip` |
| `au_fft` | `Magnitude` spectrum, `DominantFreq`, `Pow2Ceil`, `WindowHann` |
| `au_synth` | Oscillators (sine/square/saw/triangle/noise), `ADSR` envelope, `NoteToFreq`, `Tone`, `Note`, `Chord` |
| `au_play` | `FindPlayer`, `PlayWavFile`, `Play` — writes a temp WAV and execs the first available system player |

## Examples

**Musical note with envelope:**

```skink
env := au_synth.DefaultADSR()          // 10ms attack, 0.7 sustain, 200ms release
n := au_synth.Note("A4", au_synth.WAVE_SINE, 0.5, 44100, 0.8, env)
```

**Chord:**

```skink
var names []string
names = append(names, "C4"); names = append(names, "E4"); names = append(names, "G4")
chord := au_synth.Chord(names, au_synth.WAVE_SINE, 1.0, 44100, 0.9)
```

**Pitch detection:**

```skink
b, err := au_wav.Read("input.wav")
m := au_pcm.Mono(&b)
au_fft.WindowHann(m.Samples)
freq := au_fft.DominantFreq(m.Samples, m.Rate)
```

**Resample + filter:**

```skink
down := au_dsp.Resample(&b, 22050)
au_dsp.Lowpass(&down, 4000.0)
au_dsp.Normalize(&down, 0.95)
```

## Playback notes

`au_play` tries `aplay`, `paplay`, `ffplay`, `afplay`, and `play`
(SoX) in that order. `FindPlayer()` reports which one is installed;
`Play()` returns an error when none is found. Playback is blocking —
run inside `spawn` for non-blocking behavior.

## Tests

```
bash run_tests.sh
```

16 tests cover PCM math, S16/WAV round-trips (including file IO and
malformed-input rejection), gain/mix/fade/echo/filters/resample, FFT
dominant-frequency recovery, ADSR envelope shape, note/chord
synthesis, and player detection.
