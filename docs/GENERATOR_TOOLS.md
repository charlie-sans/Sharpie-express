# Generator

**Source:** `src/Sharpie.Generator/Program.cs`

A code generator that consumes the single source of truth for the ISA - `src/Sharpie.Generator/res/opcodes.json` - and emits four artifacts. Run it whenever opcodes change, then commit the regenerated files.

## Input

`opcodes.json` is a JSON array of opcode records:

| Field | Meaning |
| --- | --- |
| `hex` | Opcode byte(s) as hex string. |
| `name` | Mnemonic (maps to `Execute_<name>`). |
| `len` | Total instruction length in bytes. |
| `family` | If true, the opcode occupies a full 16-wide family (`0xXn`), and the CPU dispatch treats it as `case >= 0xXn and <= 0xXf`. |
| `logic` | Optional inline C# to emit instead of a method call (for trivially inline-op codes). |
| `words` | Operand word count for the assembler's validation. |
| `pattern` | Assembly-language operand pattern string (e.g. `rx, ry`). |
| `desc` | Human-readable description (feeds the ISA doc). |
| `alt` | Description of the ALT-prefixed variant of this opcode (feeds the ISA doc). |

## Outputs

| Artifact | Path | Purpose |
| --- | --- | --- |
| `Cpu.Ops.g.cs` | `src/Sharpie.Core/Hardware/` | The CPU's `ExecuteOpcode` switch (case ranges, per-opcode `pcDelta`, inline logic or `Execute_<name>` call, unknown-opcode halting) plus `partial void Execute_<name>` declarations. |
| `InstructionSet.g.cs` | `src/Sharpie.Assembler/Utilities/` | `InstructionSet` lookup table: opcode length/hex/words/family/pattern accessors used by the assembler. |
| `docs/ISA_REFERENCE.md` | `docs/` | The hand-consumable assembly reference table, regenerated from the same source. |
| `ext/nvim/syntax/sharpie.vim` | `ext/nvim/` | Neovim syntax file (opcode keyword list + register/hex/note/label/directive rules). |
| `ext/vscode/` | `ext/vscode/` | VS Code TextMate grammar, `package.json`, and language configuration. |

## Running

```
dotnet run --project src/Sharpie.Generator
```

It reads `./src/Sharpie.Generator/res/opcodes.json` and writes all artifacts relative to the repo root. Because generated docs and editor syntaxes are committed, a diff review should include the regenerated files.

# ImageConverter

**Source:** `src/Sharpie.Tools/ImageConverter.cs`

Converts PNG sprite sheets into Sharpie sprite-atlas data, emitted either as assembly or C source. Invoked by the CLI when the input extension is `.png`:

```
sharpie spritesheet.png              # → .c source
sharpie spritesheet.png -S           # → .asm source
```

The converter packs 4-bit indexed pixels two-per-byte to match the PPU's sprite-atlas format (4 bytes per row, 32 bytes per 8x8 sprite), flips/positions them for the downward-growing atlas layout, and emits preformatted `.asm`/`.c` for `#include` into your cartridge.