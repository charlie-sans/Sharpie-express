# Motherboard

**Source:** `src/Sharpie.Core/Hardware/Motherboard.cs`, `IMotherboard.cs`

The `Motherboard` is the central bus and facade that wires every hardware subsystem together: CPU, PPU, APU, Memory, OAM, and the Sequencer, plus the pluggable I/O drivers (display, audio, input, debug, save). It implements `IMotherboard`, the contract every subsystem uses to reach memory and peripherals. `SharpieConsole` is the public wrapper around it (see ARCHITECTURE.md).

## Ownership

The motherboard constructs and owns:

- `Cpu` - the 16-bit processor.
- `Ppu` - the compositor + VRAM.
- `Apu` (static) - the 8-channel synthesizer.
- `Memory _ram` - the full 64KB working memory (code, sprites, work RAM, audio RAM, palette).
- `Memory _biosRom` - the BIOS code region.
- `Memory _instrumentTable` - the 128-slot (512 byte) ADSR instrument table.
- `OamBank` - the sprite attribute table.
- `Sequencer` - the hardware music sequencer.

It also holds the injected device drivers: `IDisplayOutput`, `IAudioOutput`, `InputHandler`, optional `DebugOutput`, and optional `ISaveHandler`.

## BIOS Flag Region

The motherboard reserves a set of status addresses in the reserved space (see MEMORY.md):

| Address | Flag |
| --- | --- |
| `0xFA20` | Magic string header (4 bytes: `"SHRP"`). |
| `0xFA24` | BIOS version (word, `0x0004` = 0.4). |
| `0xFA26` | Cartridge verification state. |
| `0xFA28` | Cartridge loaded flag (`0x01` loaded, `0xFF` failed, `0x00` none). |
| `0xFA29` | Error / segfault code. |

## Boot Flow

1. **`LoadBios(byte[] biosData)`** copies the first 18 KiB into `_biosRom`, clears the cart-loaded flag, and splices the BIOS syscall table (`$FA2A` onward, minus the last 32 bytes of palette padding) into RAM. This lets BIOS syscalls be addressable from cartridge code.
2. The Motherboard constructor sets `IsInBootMode = true`, meaning reads below `SpriteAtlasStart` (`0xE7FF`) return BIOS ROM instead of cartridge RAM.
3. **`LoadCartridge(byte[] fileData)`** parses the 80-byte `"SHRP"` header (magic, version, 16 legacy palette colors), then loads:
   - First 18 KiB -> fixed ROM at `0x0000`.
   - Between 18 KiB and the last 8 KiB -> 32 KiB bank slices installed via `SetBanks`.
   - Last 8 KiB -> sprite atlas at `0xC800`.
   - On failure, sets the cart-loaded flag to `0xFF` and pushes the exception to debug output.
4. The BIOS writes `1` to the `CartVerificationState` flag (`0xFA26`), which the motherboard intercepts in `WriteByte`/`WriteWord` while in boot mode and triggers **`BootIntoCartridge()`**.
5. **`BootIntoCartridge()`** marks `IsInBootMode = false`, resets/reboots the audio & sequencer, halts + resets the CPU, applies the cartridge's legacy palette, and clears the OAM.

## Memory Read/Write (`IMotherboard`)

- **Reads** honor boot mode: while `IsInBootMode`, addresses at/below `0xE7FF` read from BIOS ROM.
- **Writes** to the protected region between `ReservedSpaceStart` (`0xF800 + 544`) and `ColorPaletteStart` (`0xFFE0`) raise `SegfaultType.ReservedRegionWrite` while a cartridge is running. Writing `0xFFFF` also segfaults.
- Writing `1` to `0xFA26 while booting kickstarts `BootIntoCartridge()`.

## Per-Frame Step

`Motherboard.Step()` is the heart of the emulator loop:

```
IsForcedYield = false
for i in 0..15999:
    if cpu.IsAwaitingVBlank or cpu.IsHalted: break
    cpu.Cycle()
GetInputState()     # poll input into ControllerStates
VBlank()            # Ppu.VBlank(oam): composite the frame
cpu.IsAwaitingVBlank = false
```

Any fatal exception is logged through `PushDebug` with the PC, dumped via `DebugOutput.LogAll()`, and rethrown to the caller (which the runners use to trigger the blue screen).

## Peripheral Methods (syscall backing)

| Group | Methods |
| --- | --- |
| Video | `ClearScreen`, `VBlank`, `SetTextAttributes`, `DrawChar`, `ReadVram`, `WriteVram`, `SetBlitterMode`, `GetVideoBuffer`, `MoveCamera`/`SetCamera`. |
| Sprites | `WriteSpriteEntry`, `ReadSpriteEntry`, `GetOamCursor`, `SetOamCursor`, `CheckCollision`. |
| Audio | `PlayNote`, `StopChannel`, `StopAllSounds`, `StartSequencer`, `ToggleSequencer`, `GetSequencerCursor`, `SetSequencerCursor`, `DefineInstrument`, `ReadInstrument`. |
| Input | `GetInputState`. |
| Storage | `SaveToDisk`, `LoadFromDisk`, `SaveRam` (span view for the save driver). |
| Misc | `SetCurrentBank`, `GetCurrentBank`, `SwapColor` (SWC), `StopSystem`, `TriggerSegfault`, `PushDebug`. |

## Palette

- `LoadDefaultPalette()` (used at boot/reset) maps each of the 31 palette slots to itself.
- `LoadPalette(byte[32])` (used at cartridge boot) remaps each slot to the cartridge's legacy palette (invalid entries stay identity).
- `SwapColor(old, new)` implements the `SWC` opcode: writes the target color into a palette slot.
- The static `IMotherboard.MasterPalette` is the fixed 32-entry (R, G, B) table used by `Ppu.GetFrame()` to convert VRAM indices into pixels.

## Font

`IMotherboard` embeds the built-in 8x8 BIOS font (`SmallFont`), indexed by glyph. Helpers:

- `GetCharacter(int index)` - returns the 8-byte pixel row array for a glyph (out-of-range maps to `?`).
- `AsciiToGlyphIndex(byte)` - maps ASCII (folded uppercase) to a glyph index.
- `GlyphIndexToAscii(byte)` - the reverse mapping.

## Segfaults (see SegfaultType.cs)

`TriggerSegfault(SegfaultType)` pushes a message to debug output, writes the error code into the reserved flag byte, then calls `ResetState()`, which halts the CPU, clears RAM/audio/OAM/text, and returns the system to boot mode so the BIOS can render its blue screen.

| Code | Meaning |
| --- | --- |
| `OamCursorOutOfBounds` (0x01) | OAM cursor set past `MaxEntries - 1`. |
| `ReservedRegionWrite` (0x02) | Write to the protected `$FA20`-`$FFFF` region. |
| `StackUnderflow` (0x03) | RET/POP with an empty stack. |
| `StackOverflow` (0x04) | CALL/PUSH past the stack bottom. |
| `ManualTrigger` (0xFF) | `ALT HALT` (intentional blue screen). |

## Instrument Table

`DefineInstrument(index, a, d, s, r)` stores a 4-byte ADSR profile, guarding out-of-range indexes (>= 512) with a segfault. `ReadInstrument` bounds-checks similarly. The APU reads these profiles to shape the envelope, and the `INST`/`INSTR` opcodes populate them (each nibble scaled by 17).

## Collision

`CheckCollision(srcIndex)` runs an AABB test of the source sprite (entry `srcIndex`) against every other entry, skipping `Background` and `Hud` flagged sprites. Returns the matched entry index, or `0xFFFF` if none. Backs the `COL` opcode.