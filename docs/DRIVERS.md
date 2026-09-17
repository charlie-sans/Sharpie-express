# I/O Drivers

**Source:** `src/Sharpie.Core/Drivers/`

The Core emulator is hardware-agnostic. All platform-specific behavior funnels through a small set of driver interfaces/abstract classes that each runner implements. The `SharnewConsole`/`Motherboard` constructor takes these drivers and calls them during the frame loop.

## IDisplayOutput

```csharp
public interface IDisplayOutput
{
    void Initialize(int internalResolution, string windowTitle);
    bool ShouldCloseWindow(SharpieConsole? emulator = null);
    void Cleanup();
    void HandleFramebuffer(byte[] frameBuffer);
    int GetWindowHeight();
    int GetWindowWidth();
}
```

- `Initialize` is called once at boot (the Motherboard always requests internal resolution 256 with title `"Sharpie"`).
- `ShouldCloseWindow` is polled each frame for keep-going semantics (the Raylib runner passes the emulator so it can also honor HALT/exit).
- `HandleFramebuffer` receives the BGRA8888 frame (see Ppu.Bridge) to present.
- Height/width are used by the runner for upscaling logic.

## IAudioOutput

```csharp
public interface IAudioOutput
{
    void Initialize(int sampleRate);
    void HandleAudioBuffer(float[] audioBuffer);
    void Cleanup();
}
```

- `Initialize` sets the device sample rate (the Motherboard always requests 44100).
- `HandleAudioBuffer` pushes a chunk of mixed samples (see APU.md) to the device. The Raylib runner fills its streaming buffer from audio RAM with `SharpieConsole.FillAudioBufferRange` and hands it here.

## InputHandler (abstract)

```csharp
public abstract class InputHandler
{
    public abstract (byte, byte) GetInputState();
    protected void AddKeyToState(ref byte controllerState, ControllerKeys key);
    protected bool IsButtonStateOn(byte controllerState, ControllerKeys button);
    protected enum ControllerKeys : byte { Up, Down, Left, Right, ButtonA, ButtonB, ButtonStart, ButtonOption }
}
```

`GetInputState` returns a tuple of two controller byte states (player 1 & player 2). Runners poll their platform input (keyboard + gamepad), fold it into the `ControllerKeys` bitmask via `AddKeyToState`, and return the two bytes. The Motherboard bakes these into `ControllerStates[2]` on each `Step()`, where the CPU's `INPUT` opcode reads them.

Button bit layout:

| Bit | Button |
| --- | --- |
| 0x01 | Up |
| 0x02 | Down |
| 0x04 | Left |
| 0x08 | Right |
| 0x10 | A |
| 0x20 | B |
| 0x40 | Start |
| 0x80 | Option |

## DebugOutput (abstract)

```csharp
public abstract class DebugOutput
{
    protected ConcurrentQueue<string> MessageQueue;
    protected DebugOutput(int size);
    public void PushDebug(string message);
    public void LogAll();
    public abstract void Log(string message);
}
```

Backs `OUT_R`/`OUT_B`/`OUT_W` debug opcodes and motherboard diagnostics. `PushDebug` drops the oldest message when the queue is full, `LogAll` drains it into `Log` (e.g. stdout), and runners use it for their console/debug surfaces. The headless runner redirects here for test assertions.

## ISaveHandler

```csharp
public interface ISaveHandler
{
    string? SavePath { get; }
    void SaveToDisk(ReadOnlySpan<byte> saveRam);
    ReadOnlySpan<byte> LoadSaveData(ushort byteAmount);
}
```

Backs the `SAVE` and `ALT LOAD` opcodes. The Motherboard slices the requested RAM range (`SaveRam(start, length)`), hands it to `SaveToDisk`, and `LoadFromDisk` reads back through `LoadSaveData`. Runners provide file-backed implementations (labeled by the current cartridge title).

## Sequencer (internal)

**Source:** `src/Sharpie.Core/Drivers/Sequencer.cs`

The hardware music sequencer plays a compact 4-byte-per-packet song format from main memory (usually in a ROM bank or work RAM):

| Byte | Meaning |
| --- | --- |
| `0` | Channel (0-7). |
| `1` | Note (0 = stop channel, 1-127 = pitch). |
| `2` | Duration (frames). |
| `3` | Instrument. |

Reserved channel values:

| Channel | Command |
| --- | --- |
| `0xFE` | **GOTO**: jump the cursor forward/back by `(duration | instrument << 8)` 4-byte packets. |
| `0xFD` | **TEMPO**: set `TempoMultiplier = duration`. |
| `0xFF` | **END**: stop playback, silence all channels, restore tempo to 1. |

- `LoadSong(addr)` (the `SONG` opcode) points the cursor at a packet stream and enables playback.
- `Step()` is called from the APU on a sample-accurate cadence (`1024 / TempoMultiplier` samples per step), so tempo is audio-clock driven and frame-rate independent.
- `Cursor` can be read/written via `GETSEQ`/`SETSEQ`. `MUTE` toggles `Enabled`.