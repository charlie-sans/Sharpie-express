# CPU

**Source:** `src/Sharpie.Core/Hardware/Cpu.cs`, `Cpu.Partials.cs`, `Cpu.PrefixPartial.cs`, `Cpu.Ops.g.cs`

Sharpie's CPU is a custom 16-bit little-endian design. It is split across four partial classes: the core state machine in `Cpu.cs`, the main opcode implementations in `Cpu.Partials.cs`, the `ALT`-prefixed (extended) opcodes in `Cpu.PrefixPartial.cs`, and the generated opcode dispatch table + system opcodes in `Cpu.Ops.g.cs` (generated from `res/opcodes.json` by `Sharpie.Generator`).

## Registers

- **32 general-purpose 16-bit registers** (r0 - r31), banked into **two pages of 16**. The register bank being read/written is selected by bit 15 of the flag register:
  - Bit 15 = `0` -> registers r0 - r15
  - Bit 15 = `1` -> registers r16 - r31
- `FLIPR` opcode toggles the bank by flipping flag register bit 15.
- `r0` doubles as the **exit code** register (`GetExitCode()`).
- The **stack pointer** (`_sp`) is separate and initialized to `Memory.AudioRamStart` (`0xF800`), growing downward toward the sprite atlas.
- **Program counter** (`_pc`) is a separate 16-bit register.

### Flag Register (16-bit)

The flag register packs flags in the low nibble and the register-bank selector in the MSB (right to left):

| Bit | Flag | Value | Meaning |
| --- | --- | --- | --- |
| 0 | Carry | `0x01` | Unsigned overflow (result >= 65536 or underflow on subtraction). |
| 1 | Zero | `0x02` | Result was exactly zero. |
| 2 | Overflow | `0x04` | Signed overflow (e.g. positive + positive = negative). |
| 3 | Negative | `0x08` | Highest bit of the result is set. |
| 15 | Bank | `0x8000` | Selects register page 0-15 (`0`) vs 16-31 (`1`). |

## Execution Model

- `Cycle()` reads one opcode from memory at `_pc`, executes it (writing the instruction-length delta into `pcDelta`), then advances `_pc`. If the CPU is halted or awaiting vblank, `Cycle()` returns immediately.
- Up to 16000 `Cycle()` calls happen inside one `Motherboard.Step()` frame before the vblank interrupt / input poll runs.
- `Halt()` stops execution permanently. `AwaitVBlank()` stops execution until `Step()` completes its vblank (cleared by `Motherboard.Step()`).
- `Reset()` clears all registers, resets the PC and SP, reloads the default palette, and clears the halt/vblank wait states.
- `RequestReset()` is a deferred reset: the CPU finishes the current opcode, then resets before advancing the PC.

## Stack

The call stack shares the work-RAM region. The SP starts at `0xF800` and grows downward:

- **Overflow** (into the sprite atlas at `0xC7FF` or below): `CALL`/`PUSH` raise `SegfaultType.StackOverflow`.
- **Underflow** (POP/RET past `0xF800`): raises `SegfaultType.StackUnderflow`.

`SETSP` validates the new value against both bounds and segfaults on violations. Stack accesses use word (2-byte) granularity for `CALL`/`RET`/`PUSH`/`POP` and byte granularity in the `ALT` variants (`ALT PUSH` pushes one byte, `ALT POP` pops one byte).

## Instruction Categories (see `docs/ISA_REFERENCE.md`)

### Register-to-Register (encoded as a single byte operand `rX rY`)
- `MOV`, `LDP`, `STP`, `STA`, `LDS`, `STS` (memory addressing), `ADD`, `SUB`, `MUL`, `DIV`, `MOD`, `AND`, `OR`, `XOR`, `SHL`, `SHR`, `CMP`.

### Register + Immediate
- `LDI`, `LDM` (load from address), `STM` (store to address), `IADD`, `ISUB`, `IMUL`, `IDIV`, `IMOD`, `IAND`, `IOR`, `IXOR`, `ICMP`, `INC`, `DEC`, `NOT`, `NEG`, `SIGEX`.

### Control Flow
- `JMP`, `JEQ`, `JNE`, `JGT`, `JLT`, `JGE`, `JLE`, `JC`, `JNC`, `CALL`, `RET`.

### VRAM Pixel Access
- `LDV` / `STV` - read/write a pixel indexed by a `Y<<8 | X` packed coordinate (each 256x256).

### System / BIOS
- `DRAW` (write OAM sprite entry), `CLS`, `VBLNK`, `PLAY`, `STOP`, `INPUT`, `RND`, `TEXT`, `PRNT`, `ATTR`, `SWC` (palette swap), `BANK`, `BLITMODE`, `SONG`, `MUTE`, `FLIPR`, `CAM`, `GETOAM`, `SETOAM`, `GETSEQ`, `SETSEQ`, `COL` (collision), `OAMPOS`, `OAMTAG`, `SETCRS`, `CRSPOS`, `SAVE`, `INSTR`, `OUT_R`, `OUT_B`, `OUT_W`.

### ALT-Prefixed (extended) Opcodes

The `ALT` opcode reads the sub-opcode byte that follows and can re-interpret many instructions:

- **Byte-width memory ops:** `ALT LDM`, `ALT LDP`, `ALT STM`, `ALT STP`, `ALT STA`, `ALT LDS`, `ALT STS`.
- **Carry-aware arithmetic:** `ALT ADD`, `ALT SUB` (add/subtract the carry/borrow bit), `ALT CMP`, `ALT MUL` (keeps the **high** word).
- **Register-indirect jumps:** `ALT JMP`, `ALT JEQ`, `ALT JNE`, `ALT JGT`, `ALT JLT`, `ALT JGE`, `ALT JLE`, `ALT JC`, `ALT JNC` (jump to the address held in a register).
- **Register-indirect CALL:** `ALT CALL`.
- **Pointer arithmetic:** `ALT IADD`, `ALT ISUB`, `ALT IMUL`, `ALT IDIV`, `ALT IMOD`, `ALT IAND`, `ALT IOR`, `ALT IXOR`, `ALT ICMP` (operate on the word at the address in a register).
- **Misc:** `ALT CLS`, `ALT SETCRS` (delta cursor move), `ALT CRSPOS` (delta), `ALT LOAD` (disk load), `ALT PUSH`/`ALT POP` (byte stack ops), `ALT RND`, `ALT OAMTAG` (tile id), `ALT VBLNK`, `ALT TEXT` (print a register's decimal), `ALT MUTE`, `ALT CAM` (set camera), `ALT BANK` (get bank), `ALT HALT`.

## Flag Semantics

- `UpdateFlags` computes carry (unsigned), zero, overflow (signed XOR trick), and negative for arithmetic.
- `UpdateLogicFlags` only touches zero and negative (no carry/overflow for logic ops).
- `DIV`/`MOD` and their immediate variants set `Zero` + `Overflow` on divide-by-zero and leave the destination at 0.
- The signed conditional jumps (`JGT`, `JLT`, `JGE`, `JLE`) use the standard `overflow XOR negative` signed-compare logic.

## Text Cursor

The CPU tracks a hardware text cursor (`CursorPosX`/`CursorPosY`). Both are **wrapped automatically**:
- X wraps within `0..31` and propagates overflow into Y.
- Y wraps within `0..31` using a negative-safe modulo.

`TEXT`/`PRNT` draw at the current cursor then advance it. `SETCRS`/`CRSPOS`/`ALT SETCRS`/`ALT CRSPOS` position it (absolute or delta).

## Debugging

`ToString()` dumps the PC, SP, flag register, and all 32 registers. `GetExitCode()` drives the headless runner's process exit code (register r0).