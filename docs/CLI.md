# CLI (sharpie)

**Source:** `src/Sharpie.Cli/Program.cs`

The `sharpie` command-line tool is the single front-end for the whole build pipeline: it dispatches C source, assembly, or PNG sprite images through the appropriate toolchain and emits a cartridge.

## Usage

```
sharpie <inputs...> [options]
```

## Options

| Flag | Meaning |
| --- | --- |
| `-o, --output <path>` | Output file path (`.shr`, `.bin`, or `.asm`). Defaults to the input basename. |
| `-O, --optimize` | Enable the C compiler optimizer. |
| `-S, --asm-only` | Stop after C compilation, emit assembly text instead of a ROM. |
| `-OS` / `-SO` | Optimize AND stop at assembly. |
| `--allow-long` | Enable 32-bit `long` support in the C compiler. |
| `-f, --firmware` | Assemble as raw firmware (no `.shr` header). |
| `-t, --title <name>` | ROM title (default: `Untitled`). |
| `-a, --author <name>` | ROM author (default: `Anonymous`). |
| `-h, --help` | Print the help screen. |

## Dispatch Logic

1. **No args / `-h`** prints help.
2. **All `.png` inputs**: routes to `ImageConverter`. Produces sprite-atlas data as assembly (`-S`) or C source; `-t`/`-a`/`-f` are ignored with a warning.
3. **`.c` inputs** (one or many): 
   - `SharpieCC.Compile(inputs, optimize, allowLong)` -> assembly text.
   - With `-S`: writes the `.asm` to `-o` (or `<input>.asm`) and stops.
   - Otherwise the assembly is handed to `SharpieAssembler`.
4. **`.asm` input**: assembled directly. `-O`/`-S` are rejected with an error.
5. **Anything else / mixed extensions**: rejected. Multi-file input is only allowed for `.c` and `.png`.
6. **Firmware mode** uses a `SharpieAssembler(true)` and `SharpieExporter.CreateCartridge(code, isFirmware: true)` (raw bytes, no header). Metadata flags are rejected in firmware mode.

## Output

The final payload is written by `SharpieExporter` as either a `.shr` cartridge (80-byte `"SHRP"` header + code + banks + sprite atlas) or a raw `.bin` for firmware. Exit codes: `0` success, `1` failure; all errors are printed in red with an elapsed-time summary.

## Wiring

- `CliOptions` (in `Sharpie.Cli/Options/`) holds parsed flags.
- Pixel pipeline shares `OutputFormat` (Assembly/C) between CLI and the ImageConverter.
- The CLI itself is never a runner: it only produces binaries. Execution happens in the runners (desktop/headless/web).