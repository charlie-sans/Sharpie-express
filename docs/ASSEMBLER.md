# Assembler

**Source:** `src/Sharpie.Assembler/`

The assembler turns Sharpie assembly text (or machine-code-adjacent source) into the byte arrays the emulator loads. It is a multi-pass, region-aware assembler that enforces the cartridge's physical memory layout while assembling.

## Public Entry Point

```csharp
public class SharpieAssembler
{
    public SharpieAssembler(bool isFirmware = false);
    public byte[] AssembleFromFile(string inputFilePath);   // parse a .asm path
    public byte[] AssembleFromText(string sourceText);      // parse an in-memory string
}
```

The `isFirmware` flag changes which region layout is emitted (see `IRomBuffer`).

## Pipeline

1. **`Assembler.Lexer.cs`** - tokenizes source text into `TokenLine`s (label/opcode/operand lines with file/line metadata).
2. **`Assembler.Compiler.cs`** - `SharpieRomEmitter` drives the multi-pass assembly:
   - Fold constants / enums, resolve labels and structure scopes.
   - Resolve opcodes via the generated `InstructionSet` lookup (`InstructionSet.g.cs`).
   - Emit bytes into region buffers.
3. **`Structuring/IRomBuffer.cs`** - the output is written into one of several **region-aware buffers** that each model a physical chunk of cartridge memory, with double-write and overflow protection.
4. **`Utilities/SharpieExporter.cs`** - wraps the assembled payload into a `.shr` cartridge (or passes raw bytes through for firmware).

## Region Buffers

Each buffer enforces its own size, tracks a cursor, marks every touched byte (writing to a byte twice throws `SharpieRomSizeException`), and maintains a scope stack for labels/data:

| Buffer | Size | Emits | Purpose |
| --- | --- | --- | --- |
| `FixedRegionBuffer` | 18 KiB | `0x0000 - 0x47FF` | The always-loaded fixed ROM. |
| `BankBuffer` | 32 KiB each | `0x4800 - 0xC7FF` | One buffer per switchable bank (`BANK`-able). Auto-numbered (`Bank 0`, `Bank 1`, ...). |
| `SpriteAtlasBuffer` | 8 KiB | `0xC800 - 0xE7FF` | Sprite tile bytes; `PositionCursor(spriteIndex)` places the cursor at `(8KiB-1) - 32*(spriteIndex+1)`, matching the emulator's downward-growing atlas. |
| `FirmwareBuffer` | 64 KiB | full space | Raw / non-cart form, used for BIOS and firmware builds. Sprites positioned at `(58KiB-1) - 32*(index+1)`. |

The shared `IRomBuffer` contract provides `WriteByte`/`WriteWord` with bounds + double-write checks, plus a `(NewScope/ExitScope/Scopes)` block-structure stack for handling scoped data.

## Scope System

- A global scope is always present at the bottom of each buffer's scope stack.
- `AssemblySyntaxException` is thrown if code tries to exit the global scope.
- `ScopeLevel` objects track block nesting so labels/constants can be resolved contextually across passes.

## Opcode Metadata

`Utilities/InstructionSet.g.cs` is **auto-generated** by `Sharpie.Generator` from `res/opcodes.json`. It exposes:

- `GetOpcodeLength(name)`, `GetOpcodeHex(name)`, `GetOpcodeWords(name)`
- `IsOpcodeFamily(name)` - whether the opcode occupies a 16-wide family (`0xXn`).
- `GetOpcodePattern(name)` - the operand pattern string for validation.
- `IsValidOpcode(name)`

## Cartridge Exporter (`.shr` format)

`SharpieExporter.CreateCartridge(compiledCode, isFirmware)` writes the binary payload. Firmware mode returns the raw bytes with no header. Otherwise it prepends an **80-byte header**:

| Field | Size | Notes |
| --- | --- | --- |
| Magic | 4 | ASCII `"SHRP"`. |
| Title | 24 | Space-padded ASCII. |
| Author | 14 | Space-padded ASCII. |
| Version | 2 | Little-endian `ushort` (major << 8 | minor). |
| Padding | 4 | `0xFF` x4. |
| Legacy palette | 32 | `0xFF` x32 (palettes are being removed in v0.4; retained for compatibility). |
| Code | variable | The assembled payload (fixed ROM + banks + sprite atlas). |

## Errors

- `SharpieRomSizeException` - writing past a region's end, or double-writing the same address.
- `AssemblySyntaxException` - bad tokens, unclosed/over-closed scopes, invalid sprite indices.
- Both are surfaced by the CLI as "Build failed" messages with elapsed time.