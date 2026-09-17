# C Compiler

**Source:** `src/Sharpie.CCompiler/`

Sharpie ships a C compiler that compiles a C99-ish dialect down to Sharpie assembly, which is then fed to the assembler. It uses ClangSharp (libclang) to *parse* the C source, then a hand-written emitter translates the AST to Sharpie asm.

## Supported C

- Full GNU11 parse via libclang with `-std=gnu11 -target msp430`.
- `long` (32-bit) integers, opt-in via the `--allow-long` CLI flag (slower codegen).
- `struct`s and `union`s.
- Pointers, arrays, and pointer arithmetic.
- Strings and string pooling (deduplicated in a data section).
- `#pragma BANK(n)` and a `BANK()` attribute for cross-bank calls via `SYS_FAR_CALL`.
- `#include <sharpie.h>` and friends from the bundled SDK headers (`libs/`).

## Register Conventions

| Registers | Role |
| --- | --- |
| `r1` - `r7` | Scratch / temporaries. |
| `r8` - `r14` | Callee-saved locals. |
| `r15` | Frame pointer. |
| Function args / return | Per the Sharpie SysV-ish ABI the emitter implements. |

## Pipeline

1. `SharpieCC.Compile(inputs, optimize, allowLong)` runs libclang over the input `.c` files (multiple translation units allowed).
2. A set of `Emitter` classes walk the clang AST and emit Sharpie assembly text.
3. If `optimize` is set, the emitted code passes through the `Optimizer` (a ~1463-line def-use peephole engine - `DefMnemonics`/`UseMnemonics` analysis) before being returned.
4. The CLI hands the resulting assembly to `SharpieAssembler`.

## Intrinsics

`Intrinsics.cs` maps the SDK helper functions in `sharpie.h` (drawing, input, camera, audio, sprite ops) directly onto the underlying Sharpie opcodes so they compile efficiently.

## SDK Headers (`libs/sharpie/`)

- `sharpie.h` - umbrella header.
- `defs.h` - base types (`uint16_t`, `bool`, `Vector2`, `Body`, colors, constants, `ATTR_*` flags, button enums, etc.).
- `graphics/hardware.h` - engine/rendering helpers, `Color`, `Vector3`, sprite/blit APIs.

These headers are also used by the fixtures and the samples (`samples/*.c`).

## Fixtures & Testing

`fixtures/` holds 40+ C test programs grouped by feature:
- `8bit`, `32bit` - integer widths.
- `structs` - struct/union layout.
- `functions` - call/return conventions.
- `control_flow` - branches, loops, switch.
- `data` - static/global data, arrays.
- `internals` - compiler internals edge cases.
- `headers` - SDK-header compilation checks.

The xUnit test project (`SharpieCCompilerTests`) drives the CLI to compile each fixture, runs the result with the headless runner, and asserts exit codes/behavior. A SHA-256 `fixture_cache.json` in the test project lets CI skip fixtures whose sources haven't changed.

## Optimizer

The `Optimizer` performs peephole passes over the emitted assembly:

- **Def-use analysis** (`DefMnemonics` / `UseMnemonics`): which instructions define registers vs consume them, enabling dead-store elimination and redundancy removal.
- Constant folding and strength reductions where the target ISA allows.

It is enabled with `-O`/`--optimize`.