# APU (Audio Processing Unit)

**Source:** `src/Sharpie.Core/Hardware/Apu.cs`

The APU synthesizes the Sharpie's 8 monophonic audio channels in software. It is triggered by the display/audio driver to fill an output buffer at 44.1 kHz, mixing all channels and applying an ADSR envelope per channel.

## Channel Configuration

Each of the 8 channels owns a 4-byte control block in audio RAM, starting at `Memory.AudioRamStart + (channel * 4)`:

| Offset | Size | Meaning |
| --- | --- | --- |
| `+0` | word (LE) | Frequency. For channels 0-5 this is the pitch in Hz; for noise channels 6-7 it encodes the noise period. |
| `+2` | byte | Channel max volume (0-255, scaled to 0.0-1.0). |
| `+3` | byte | Control byte: bit 0 = **gate** (note on/off), bits 1-7 = **instrument ID** (0-127). |

## Waveforms

| Channel | Waveform |
| --- | --- |
| 0, 1 | Square (with polyblep anti-aliasing) |
| 2, 3 | Triangle |
| 4, 5 | Sawtooth (with polyblep) |
| 6, 7 | Noise (software LFSR-style white noise) |

### Noise Channels (6-7)

Noise uses a sample counter with a variable period derived from the frequency value:

```
currentFreq *= 128
period = max(1, (127 - currentFreq) / 4)
```

- High notes (e.g. 84 = hi-hat) yield a tiny period => fast, bright noise.
- Low notes (e.g. 20 = kick drum) yield a large period => slow, crunchy noise.
- A new random value is drawn every time the counter reaches the period.

## ADSR Envelope

Sixteen stage registers are tracked per channel (`_stages[8]`, `_volumes[8]`). The envelope state machine:

```
Idle -> Attack -> Decay -> Sustain -> Release -> Idle
```

- **Gate on** (control bit 0 = 1): the stage enters `Attack` unless already active; a retrigger (gate was off, or frequency/control changed) halves the current volume and restarts `Attack`.
- **Attack:** volume climbs toward channel max volume at `aStep`.
- **Decay:** volume falls toward the sustain level.
- **Sustain:** volume holds at `sustain * channelMaxVolume` while the gate stays on.
- **Gate off:** stage moves to `Release`; volume decays to 0, then `Idle`.

The 4 envelope parameters `(Attack, Decay, Sustain, Release)` come from the **instrument table** on the motherboard (up to 128 instruments, each 4 bytes: A, D, S, R). `INST`/`INSTR` opcodes and `DefineInstrument()` populate the table.

Step sizes are roughly `param / 100000` (plus a tiny epsilon to guarantee progress), so the envelope is tuned for a 44.1 kHz sample rate.

## Mixing & Output

`FillBufferRange(float[] writeBuffer, int sampleCount)`:

1. Returns silence immediately if the APU is disabled.
2. For each sample: advances the sequencer (see below), sums all 8 channels, and applies `tanh(sample * 0.3)` soft clipping.
3. A pointer overload (`float* writeBuffer, uint sampleCount`) exists for the native/marshaled audio path and works the same, minus the enable check.

## Sequencer Heartbeat

The APU also drives the optional **hardware song sequencer** (`Sequencer.cs`) on a sample-accurate cadence:

- `AdvanceRate = 1024 / TempoMultiplier`.
- Every time a global sample counter crosses `AdvanceRate`, `Sequencer.Instance.Step()` runs.

This lets sequenced songs stay frame-rate-independent because the sequencer ticks from the audio clock, not the CPU clock.

## Note Priority

`_notePriority[8]` tracks whether each channel currently holds a priority note. `SetNotePriority` / `IsCurrentNotePrioritized` let the `PLAY` opcode (which uses priority `true, allowOverride true`) override repeated notes while the `SONG` sequencer path (priority false) cannot step on an actively playing high-priority note.

## Lifecycle

| Method | Purpose |
| --- | --- |
| `Reset()` | Clears all phase, volume, stage, frequency, control, noise, and note-priority state; resets the global sequencer counter. |
| `Enable()` / `Disable()` | Toggle audio output (silence when disabled). |
| `ResetPhase(channel)` | Zeroes a channel's phase and returns it to `Idle`. |
| `ClearPhases()` | Resets all 8 phases. |
| `LoadDefaultInstruments()` | Installs 4 built-in instrument presets (fast attack / soft attack / slow attack / percussive) into instruments 0-3. |
| `RetriggerChannel(channel)` | Queues an envelope retrigger for the next sample. |

### Default Instruments

| Index | A | D | S | R | Character |
| --- | --- | --- | --- | --- | --- |
| 0 | 0x0F | 0x00 | 0xFF | 0x05 | Fast attack, full sustain, short release. |
| 1 | 0x05 | 0x10 | 0xAA | 0x10 | Soft attack, medium decay, medium sustain. |
| 2 | 0x02 | 0x05 | 0x88 | 0x40 | Slow attack, long release. |
| 3 | 0xF0 | 0x20 | 0x00 | 0xF0 | Instant attack, fast decay, no sustain (percussive). |

## Anti-Aliasing

`PolyBlep(phase, delta)` applies polynomial band-limited step correction to square and sawtooth waves, removing the harsh aliasing that comes from naively summing a hard-edged waveform at a low sample rate. Triangle is piecewise-linear and needs no correction.