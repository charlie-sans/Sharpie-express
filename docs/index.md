# Sharpie-express Documentation

Developer-facing docs for the Sharpie-express codebase (a port of the Sharpie 16-bit fantasy console to Stardust by Finite).

## Getting Oriented

Start with the **Architecture** doc, then drill into whichever subsystem you're touching.

| Doc | Covers |
| --- | --- |
| [ARCHITECTURE.md](ARCHITECTURE.md) | Project layout, data flow, emulation loop, runtime boundaries, gotchas. |
| [MEMORY.md](MEMORY.md) | Memory map, bank switching, palette region, read/write API. |
| [CPU.md](CPU.md) | 16-bit CPU, registers, flag register, banking, opcode families, stack, text cursor. |
| [PPU.md](PPU.md) | 256x256 framebuffer, camera, sprite decode, blitter modes, text layer, palette remap. |
| [APU.md](APU.md) | 8-channel synth, 4-byte channel blocks, ADSR instruments, waveforms, sequencer heartbeat. |
| [OAM.md](OAM.md) | Sprite attribute table, 7-byte entries, cursor, sprite flags. |
| [MOTHERBOARD.md](MOTHERBOARD.md) | Bus wiring, boot flow, BIOS flags, segfaults, per-frame step, peripherals. |
| [DRIVERS.md](DRIVERS.md) | Platform I/O interfaces (display, audio, input, save, debug) + the Sequencer. |
| [ASSEMBLER.md](ASSEMBLER.md) | Assembly pipeline, region buffers, scopes, opcode metadata, `.shr` exporter. |
| [CCOMPILER.md](CCOMPILER.md) | C -> Sharpie compiler, clang parsing, register conventions, optimizer, fixtures. |
| [CLI.md](CLI.md) | `sharpie` command-line tool and its pipeline dispatch. |
| [RUNNERS.md](RUNNERS.md) | Desktop (raylib), headless (tests), and web (WASM) runner hosts. |
| [GENERATOR_TOOLS.md](GENERATOR_TOOLS.md) | Opcode generator (opcodes.json -> code/docs/editor syntax) + PNG ImageConverter. |

## Reference

These are pre-existing reference docs, regenerated from `opcodes.json`:

- [ISA_REFERENCE.md](ISA_REFERENCE.md) - the full assembly opcode table (mnemonic, args, length, description, ALT behavior).
- [SYSCALL_REFERENCE.md](SYSCALL_REFERENCE.md) - the BIOS syscall table at `$FA2A`-`$FB4E`.

## Quick File Map

| Want | Look in |
| --- | --- |
| Public emulator API | `src/Sharpie.Core/SharpieConsole.cs` |
| CPU core | `src/Sharpie.Core/Hardware/Cpu*.cs` |
| Graphics | `src/Sharpie.Core/Hardware/Ppu.cs` + `Ppu.Bridge.cs` |
| Audio | `src/Sharpie.Core/Hardware/Apu.cs` + `Drivers/Sequencer.cs` |
| Sprites | `src/Sharpie.Core/Hardware/OamBank.cs` |
| Memory | `src/Sharpie.Core/Hardware/Memory.cs` |
| Bus + peripherals | `src/Sharpie.Core/Hardware/Motherboard.cs` + `IMotherboard.cs` |
| Toolchain | `src/Sharpie.Assembler/`, `src/Sharpie.CCompiler/`, `src/Sharpie.Cli/` |
| Runners | `src/Sharpie.Runner/{RaylibCs,Headless,Web}/` |
| BIOS source | `assets/bios/*.asm` |
| Example cartridges | `samples/*.asm`, `samples/*.c` |