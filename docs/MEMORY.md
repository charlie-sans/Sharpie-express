# Memory Subsystem

**Source:** `src/Sharpie.Core/Hardware/Memory.cs`

The `Memory` class is the raw backing store for the Sharpie console. It manages a single flat 64KB array plus an optional set of switchable ROM banks. The `Motherboard` wraps this class and routes reads/writes from the CPU, PPU, and other subsystems through it.

## Memory Map

| Region | Address Range | Size | Description |
| --- | --- | --- | --- |
| Fixed ROM | `0x0000` - `0x47FF` | 18 KiB | Always-loaded cartridge code and BIOS. |
| Switchable ROM | `0x4800` - `0xC7FF` | 32 KiB | Swappable ROM bank, selected via `SelectBank` / `BANK` opcode. |
| Sprite Atlas | `0xC800` - `0xE7FF` | 8 KiB | Sprite tile data. Grows **downward** from `0xE7FF`. Always loaded. |
| Work RAM | `0xE800` - `0xF7FF` | 4 KiB | General-purpose scratch RAM. |
| Audio RAM | `0xF800` - onward | ~544 bytes | Per-channel audio control blocks (4 bytes per channel, 8 channels). |
| Reserved Space | `$FA20` - `$FFDF` | | BIOS flags, instrument table, protected region. Writes trigger a `ReservedRegionWrite` segfault. |
| Color Palette | `0xFFE0` - `0xFFFF` | 32 bytes | 32-entry remappable color palette. |

The `Memory` class exposes the region boundaries as constants:

- `RomStart = 0x0000`
- `FixedRomEnd = 0x47FF`
- `SwitchableRomStart = 0x4800`
- `SwitchableRomEnd = 0xC7FF`
- `SpriteAtlasStart = 0xE7FF` (top address; atlas grows downward)
- `SpriteAtlasBottom = 0xC800`
- `WorkRamStart = 0xE800`
- `AudioRamStart = 0xF800`
- `ReservedSpaceStart = AudioRamStart + 544`
- `ColorPaletteStart = 0xFFE0`

## Bank Switching

The cartridge format supports multiple 32 KiB ROM banks that can be swapped into the switchable region at runtime:

- `SetBanks(byte[][] banks)` - installs the bank set and resets the current bank index to 0.
- `SelectBank(int index)` - switches the active bank. Safely no-ops if banks have not been initialized.
- `GetBank()` - returns the currently selected bank index.
- `BankCount` - total number of installed banks.

Any byte read or write into the `0x4800 - 0xC7FF` window is redirected into `_banks[_currentBankIndex]` instead of the flat `_contents` array. Read and write both reroute, so bank regions are directly writable during execution.

## Read / Write API

| Method | Description |
| --- | --- |
| `ReadByte(ushort/int address)` | Reads a single byte, honoring bank switching. |
| `WriteByte(ushort/int address, byte value)` | Writes a single byte, honoring bank switching. |
| `ReadWord(ushort/int address)` | Reads two bytes little-endian (low byte first). |
| `WriteWord(ushort/int address, ushort value)` | Writes two bytes little-endian. |
| `LoadData(ushort startAddress, ReadOnlySpan<byte> data)` | Bulk-copies data into the flat array (does **not** go through bank routing). Used to load BIOS, cartridge code, and sprite atlas. |
| `Fill(byte value)` | Fills the entire 64KB array. |
| `FillRange(int startIndex, int amount, byte value)` | Fills a bounded range. |
| `ClearRange(int from, int amount)` | Zeroes a bounded range. |
| `Slice(int from, int amount)` | Returns a writable `Span<byte>` view. |
| `View(int from, int amount)` | Returns a read-only view; prints a message and returns a zeroed span instead of throwing on out-of-range access. |

## Notes

- `View()` deliberately does not throw on out-of-range access. It prints a console warning and returns an empty span, which the save system relies on for graceful degradation.
- The sprite atlas region grows **downward** from `0xE7FF`: sprite index 0 is at `0xE7FF - 32`, sprite index 1 at `0xE7FF - 64`, etc.
- The reserved region `$FA20`-`$FFFF` is guarded by the `Motherboard` (not `Memory` itself) which raises a `SegfaultType.ReservedRegionWrite` on attempted writes while a cartridge is running.