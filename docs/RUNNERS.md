# Runners

**Source:** `src/Sharpie.Runner/`

Runners are the executables that host `SharpieConsole` and drive the emulation loop with real platform I/O. There are three: the desktop raylib runner, a headless runner for automation/tests, and a WASM web runner.

## Shared Pattern

Every runner:

1. Constructs the platform driver implementations (`IDisplayOutput`, `IAudioOutput`, `InputHandler`, `DebugOutput`, `ISaveHandler`).
2. Creates `SharpieConsole`, calls `LoadBios(embeddedBios)`.
3. Loads a `.shr` cartridge (from a CLI arg, a file dialog, drag-and-drop, or JS bytes).
4. Loops: `emulator.Step()` -> `GetVideoBuffer()` to the display -> `FillAudioBufferRange` to audio -> `LogAll()` for debug messages.
5. Returns `emulator.ExitCode` when done.

## Raylib Header

**Source:** `src/Sharpie.Runner/RaylibCs/`

Built on `Raylib_cs`. The main `Program.cs` is a top-level program:

- **Input**: cartridge path as `argv[0]` (must end in `.shr`); also supports clicking while in boot mode to open a native `FileDialog`, and drag-and-drop of a `.shr` file.
- **Save**: sets `RaylibSaveHandler.SavePath` to `<cartridge>.sav`.
- **Loop**: while the window is open, if a cartridge is loaded it runs the step loop; `logger.LogAll()` drains debug output each frame.
- **Exit**: returns the emulator's exit code on window close.

### Driver implementations (`Impl/`)

| File | Implements | Notes |
| --- | --- | --- |
| `RaylibVideoOutput.cs` | `IDisplayOutput` | Creates a resizable 512x512 (2x) window, point-filtered texture, `DrawTexturePro` scaling to a centered square. |
| `RaylibAudioOutput.cs` | `IAudioOutput` | Opens an audio stream (32-bit float, mono, 44100 Hz) with a callback (`[UnmanagedCallersOnly]`) that calls `SharpieConsole.FillAudioBufferRange` directly into the buffer; real-time, no managed buffers. |
| `RaylibInputHandler.cs` | `InputHandler` | Polls keyboard + gamepad into the `ControllerKeys` bitmask for 2 players. |
| `RaylibSaveHandler.cs` | `ISaveHandler` | File-backed save RAM at `SavePath`. |
| `RaylibDebugOutpug.cs` | `DebugOutput` | **Note the typo**: the class is deliberately named `RaylibDebugOutpug` and shared with the Web runner. Logs to console. |
| `BiosLoader.cs` | - | Reads the embedded `bios.bin` manifest resource (and the icon PNG on Windows). |
| `FileDialog.cs` | - | Native file open dialog for loading cartridges at runtime. |

## Headless Runner

**Source:** `src/Sharpie.Runner/Headless/`

Used by the tests/CI. Same loop, minimal drivers:

| File | Notes |
| --- | --- |
| `HeadlessVideoOutput` | No-op framebuffer; `ShouldCloseWindow` returns `emulator?.IsHalted` so the loop stops when the cartridge halts. |
| `HeadlessAudioOutput` | No-op. |
| `HeadlessInputHandler` | Returns `(0, 0)`: no input. |
| `HeadlessSaveHandler` | No-op. |
| `HeadlessDebugOutput` | `Console.WriteLine` each message. |

`Main(args)` validates a cartridge path argument exists, embeds `bios.bin` as a manifest resource, steps until halted, and returns the emulator's exit code. This is how the C compiler tests assert program behavior (e.g. a fixture that `return 42;` yields exit code 42).

## Web Runner (WASM)

**Source:** `src/Sharpie.Runner/Web/`

Compiles the same raylib drivers to browser WASM. Exposes a tiny JS interop surface:

| Export | Purpose |
| --- | --- |
| `UpdateFrame()` | One emulator step + push the framebuffer (called from the browser's `requestAnimationFrame`). |
| `IsInBootMode()` | Whether the BIOS boot screen is up. |
| `LoadCartridgeFromBytes(byte[] data)` | Load a `.shr` from JS bytes (file input / drop). |

The browser side (`main.js` + `_framework/dotnet.js`) wires cart loading and the animation loop. See the GitHub Pages workflow in `.github/` for how the WASM build is published.