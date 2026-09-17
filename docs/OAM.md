# OAM (Object Attribute Memory)

**Source:** `src/Sharpie.Core/Hardware/OamBank.cs`

The OAM holds the sprite metadata table for all objects in the world. It is a fixed 8192-byte buffer divided into fixed 7-byte entries, exposed as a ring buffer through a cursor.

## Layout

| Constant | Value |
| --- | --- |
| `OamEntrySize` | 7 bytes |
| `Size` | 8192 bytes |
| `MaxEntries` | 8192 / 7 = 1170 entries |
| `MaxHudEntries` | 512 |

The class was deliberately NOT made an interface implementation: virtual dispatch made the emulator too slow, so it is a concrete class with direct array access.

## Entry Format (7 bytes)

| Offset | Size | Field |
| --- | --- | --- |
| +0 | word (LE) | X position (16-bit world coordinate). |
| +2 | word (LE) | Y position (16-bit world coordinate). |
| +4 | byte | Tile (sprite) ID. |
| +5 | byte | Attributes (see flags below). |
| +6 | byte | Type / tag (user-defined; used for collisions and `OAMTAG`). |

## Cursor

`Cursor` is the auto-incrementing write index. `WriteEntry` places the next sprite at `Cursor * OamEntrySize` and advances the cursor. The setter wraps to 0 if the value exceeds `MaxEntries`.

- `SetOamCursor` (via the `SETOAM` opcode and CPU) validates against `MaxEntries` and raises a `SegfaultType.OamCursorOutOfBounds` if too large.
- `CLS` resets the cursor to 0 so the next frame redraws from the top of the table.

## Sprite Flags (attributes byte)

| Bit | Mask | Flag | Meaning |
| --- | --- | --- | --- |
| 0 | `0x01` | `FlipH` | Mirror the sprite horizontally when blitting. |
| 1 | `0x02` | `FlipV` | Mirror the sprite vertically when blitting. |
| 2 | `0x04` | `Hud` | HUD sprite: rendered with raw screen coordinates, ignores the camera and doesn't participate in collisions. |
| 3 | `0x08` | `Background` | Treated as background geometry: excluded from collision checks (handled by the motherboard's `CheckCollision`). |
| 4 | `0x10` | `AlternatePalette` | Add +16 to every pixel's color index (selects the alternate color bank). |

## API

| Method | Purpose |
| --- | --- |
| `WriteEntry(ushort x, ushort y, byte tileId, byte attr, byte type)` | Appends a sprite at the current cursor and advances it. |
| `ReadEntry(int index)` | Reads a full entry as a tuple `(X, Y, TileId, Attr, Type)`. |
| `Invalidate(int from, int to)` | Fills a range with `0xFF` (marks entries empty; comprehensive-range version used by `ClearScreen`). |
| `InvalidateAll()` | Fills the entire buffer with `0xFF`. The PPU treats an all-`0xFF` entry as empty and skips it. |

## Consumers

- **PPU** (`Ppu.cs`) iterates `MaxEntries` every vblank, skipping all-`0xFF` entries, caching HUD-tagged sprites for the HUD pass, and blitting world sprites against the camera viewport.
- **Motherboard collision** (`CheckCollision`) does AABB checks over entries for `COL` (collision) opcode queries, skipping `Background` and `Hud` flagged sprites.
- **CPU** writes entries via `DRAW` (5 registers: x, y, sprite id, attrs, type) and reads them via `OAMPOS` (position) and `OAMTAG` (attrs/type).