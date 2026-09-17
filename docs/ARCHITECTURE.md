# Architecture Overview

This document explains how the Sharpie-express codebase fits together, and where each piece lives.

## Projects

| Project | Path | Purpose |
| --- | --- | --- |
| `Sharpie.Core` | `src/Sharpie.Core/` | The emulator proper: CPU, memory, PPU, APU, OAM, motherboard, sequencer, and the public `SharpieConsole` facade. Also hosts the driver interfaces. |
| `Sharpie.Assembler` | `src/Sharpie.Assembler/` | The Sharpie assembly toolchain: lexer, compiler, IR/region layout, and the cartridge exporter. |
| `Sharpie.CCompiler` | `src/Sharpie.CCompiler/` | The C -> Sharpie-asembly compiler (ClangSharp-backed parser + hand-written emitter + peephole optimizer). Ships the `sharpie.h` SDK headers and the C test fixtures. |
| `Sharpie.Cli` | `src/Sharpie.Cli/` | The `sharpie` command-line front-end that wires files into the assembler/compiler/exporter, and supports PNG -> sprite-atlas conversion. |
| `Sharpie.Generator` | `src/Sharpie.Generator/` | Code generator that turns `res/opcodes.json` into `Cpu.Ops.g.cs` (the opcode dispatch table). |
| `Sharpie.Tools` | `src/Sharpie.Tools/` | `ImageConverter` - converts PNG sprites into Sharpie asm/C sprite-atlas data. |
| `Sharpie.Runner` | `src/Sharpie.Runner/RaylibCs/`, `Headless/`, `Web/` | The three `SharpieConsole` hosts: desktop (raylib), headless (CI/testing), and browser (WASM). |
| `Sharpie.Tests` | (`SharpieCCompilerTests`) | xUnit test suite driving the CLI + headless runner over the C fixtures. |

## Data Flow

```
.strip {c0, cqA}
                                    ┌──────────────────────────────────────────────┐
   .c / .asm / .png  ──────────►   │   Sharpie.Cli (sharpie CLI)                 │
        source                     │   ├─ SharpieCCompiler (C → asm)              │
                                   │   ├─ SharpieAssembler (asm → machine bytes)  │
                                   │   └─ SharpieExporter (→ .shr / .bin)         │
                                   └──────────────────────────────────────────────┘
                                                              │
                                                              ▼
                                    ┌──────────────────────────────────────────────┐
   .shr cartridge + bios.bin  ────► │   SharpieConsole (public facade)             │
                                    │   └── Motherboard (bus)                      │
                                    │       ├── Cpu ◄── Memory ◄── banks / BIOS    │
                                    │       │        │                             │
                                    │       ├── Ppu ◄── VRAM ◄── OamBank           │
                                    │       ├── Apu ◄── AudioRAM ◄── Sequencer     │
                                    │       └── drivers: display / audio / input / │
                                    │                        save / debug          │
                                    └──────────────────────────────────────────────┘
                                                              │
                                  ┌───────────────────────────┼────────────────────┐
                                  ▼                           ▼                    ▼
                          RaylibCs runner              Headless runner        Web runner
                         (desktop window)              (exit code / tests)   (WASM + canvas)
```

## Core Emulation Loop

1. The runner constructs a `SharpieConsole(display, audio, input, debug, save)`.
2. `LoadBios(biosData)` installs the BIOS firmware.
3. `LoadCartridge(fileData)` parses the `.shr`, loads fixed ROM + banks + sprite atlas, and sets the cart-loaded flag.
4. The BIOS boot code hands off to the cartridge (see MOTHERBOARD.md).
5. Each 60 fps tick, the runner calls `SharpieConsole.Step()`, which:
   - Runs up to 16000 CPU cycles (stopping early if the CPU halted or awaits vblank).
   - Polls input into the controller states.
   - Runs `Ppu.VBlank()` to composite the frame.
6. The runner pulls `GetVideoBuffer()` (BGRA8888) into its display, and streams `FillAudioBufferRange` into `IAudioOutput`.

## Runtime Boundaries

| Boundary | Mechanism |
| --- | --- |
| CPU to memory | `IMotherboard.ReadByte/ReadWord/WriteByte/WriteWord` via `_mobo`. |
| CPU to video | `ReadVram/WriteVram` and the syscall methods (DRAW, CLS, CAM, BLITMODE...). |
| CPU to audio | `PlayNote`, `StopChannel`, `StartSequencer`, `DefineInstrument`, ... |
| CPU to peripheral | `GetVideoBuffer`, `FillAudioBufferRange`, `ControllerStates`, `ISaveHandler`. |
| PPU to main RAM | `IMotherboard` reads for sprite atlas, palette remap, and font. |
| APU to main RAM | `IMotherboard` reads for per-channel control blocks + instrument table. |

## Test Strategy

`SharpieCCompilerTests` (xUnit) drives the CLI to compile each fixture under `src/Sharpie.CCompiler/fixtures/`, then runs the headless runner over the output and asserts the exit code / behavior. A SHA-256 `fixture_cache.json` skips re-running unchanged fixtures.

## Build & Release

`.github/workflows/build-and-release.yaml` builds the CLI + runner for linux-x64, win-x64, win-arm64 (raylib compiled from source for ARM), plus a browser WASM Web runner published to GitHub Pages. Releases are tagged `v*` (a dash in the tag marks it as a prerelease).

## Gotchas Working Here

- **BIOS + palette overlap:** when loading the BIOS syscall table, skip the final 32 bytes (the palette region) or you will zero out the color palette (a footgun documented in `Motherboard.LoadBios`).
- **Sprite atlas is upside-down:** sprite N lives at `SpriteAtlasStart - 32*(N+1)`, growing downward from `0xE7FF`.
- **Register banking:** bit 15 of the flag register selects register page 0-15 vs 16-31; `FLIPR` toggles it.
- **Stack direction:** SP starts at `0xF800` and grows down; bounded by the sprite atlas below and audio RAM above.
- **The class typo** `RaylibDebugOutpug` is intentional-looking but has been left as-is across the Raylib and Web runners.
- **`Apu` is static** on the Motherboard, so audio state is process-global (fine for single-console runners, thread-safety only matters if you host multiple consoles).