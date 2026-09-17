# PPU (Pixel Processing Unit)

**Source:** `src/Sharpie.Core/Hardware/Ppu.cs`, `Ppu.Bridge.cs`

The PPU is responsible for compositing the 256x256 display framebuffer every vblank. It combines three layers: **OAM sprites**, **HUD sprites**, and the **text grid**.

## Display & World

- **Display:** 256 x 256 pixels.
- **World:** 65536 x 65536 pixels (full 16-bit coordinate space).
- **Camera:** `CamX` / `CamY` are clamped to `(0, WorldSize - DisplaySize)` so the camera never leaves the world bounds. The camera is moved with the `CAM` opcode (`MoveCamera` relative or `SetCamera` absolute).
- **VRAM:** A dedicated `Memory` instance backing the framebuffer. Each byte is a color index. The PPU exposes `ReadByte`/`WriteByte` for the `LDV`/`STV` pixel-access opcodes.

## Sprite Storage

Sprites are 8x8 tiles drawn from the sprite atlas in main memory (`0xE7FF` growing downward). Each sprite occupies 32 bytes:

- 4 bytes per row (8 rows) = 32 bytes total.
- Each byte packs **two 4-bit pixels**: high nibble = left pixel, low nibble = right pixel.

Sprite index `N` is stored at address `SpriteAtlasStart - (32 * (N + 1))`.

## Sprite Flags (see OAM.md for full details)

| Bit | Flag | Effect |
| --- | --- | --- |
| 0 | FlipH | Mirror horizontally. |
| 1 | FlipV | Mirror vertically. |
| 2 | Hud | Screen coordinates, no camera applied. |
| 3 | Background | No collision (handled by motherboard, not PPU). |
| 4 | AlternatePalette | Offset all color indices by +16. |

## The Blitter Pipeline (`VBlank`)

Every frame, `Motherboard.Step()` calls `Ppu.VBlank(oam)` which:

1. **Clears the buffer** by filling VRAM with `BackgroundColorIndex`.
2. **Processes OAM sprites** (`ProcessOam`) - decodes each sprite, culls it against the camera viewport, and blits the visible 8x8 region.
3. **Processes HUD sprites** (`ProcessHud`) - HUD-tagged sprites were collected into `_hudSprites` during OAM processing; they render with raw screen coordinates (no camera).
4. **Processes the text grid** (`ProcessText`) - walks the 32x32 text grid and blits each character glyph in the current font color.

### Camera Culling

For each non-HUD OAM entry, the PPU converts world coordinates to screen coordinates (`localX = x - CamX`, `localY = y - CamY`) and:

- Skips sprites entirely outside the 256x256 viewport.
- Computes a clipped blit rectangle (`startX/endX/startY/endY`) so partially-visible sprites only write pixels within bounds.
- Returns early for fully-offscreen sprites (space-efficient hardware-camera semantics).

### Palette Remapping at Blit Time

Every drawn pixel color index goes through the hardware palette table before being written to VRAM:

1. The decoded sprite pixel value (color 0-15, or 16-31 in the alternate palette).
2. Color index 16 gets mapped to 0 (transparent) inside the alternate palette.
3. The index is then indexed into the `ColorPaletteStart` remap table and reduced `% 32`.
4. `WritePixel` also drops any index that equals 0 (transparency), and bounds-checks the destination.

This means sprites store **logical** color slots; the real colors are resolved through the remappable palette at draw time (`SWC` opcodes allow swapping palette slots at runtime).

## Text Layer

The text grid is a 32x32 array (each cell = 8x8 pixels = full screen of 256x256). Cell value `0xFF` means empty. Otherwise it's a glyph index into the BIOS font.

- Glyphs come from `IMotherboard.GetCharacter(index)`.
- `ProcessText` looks up the current `FontColorIndex`, remaps it through the palette, and blits each glyph via `BlitCharacter`.
- Characters are rendered 1-bit per pixel: a set bit draws the font color, an unset bit leaves the background.

## Blitter Modes

`BlitterMode` (set via the `BLITMODE` opcode) controls which layers are composited:

| Value | Mode | Layers disabled |
| --- | --- | --- |
| 0 | `Default` | None. |
| 1 | `NoText` | Text layer off. |
| 2 | `NoOam` | OAM + HUD sprites off. |
| 3 | `None` | Everything; buffer is never even implicitly cleared (reserved for raw VRAM drawing). |

The cube demo uses `BlitterMode.None` to push raw pixels through `STV`.

## Framebuffer Bridge (`Ppu.Bridge.cs`)

`GetFrame()` converts the VRAM index buffer into a **BGRA8888** framebuffer (256x256x4 = 262144 bytes) ready for the display driver:

- Each VRAM byte is looked up in `IMotherboard.MasterPalette` -> (R, G, B).
- Color index 0 produces alpha 0 (fully transparent / background).
- Every other index produces alpha 255.
- The result is cached in `_framebuffer` and returned directly to the runner.

## Key Fields

| Member | Purpose |
| --- | --- |
| `DisplayWidth/Height` (const) | 256x256. |
| `Mode` | Active blitter mode. |
| `CamX` / `CamY` | Hardware camera position, clamped. |
| `BackgroundColorIndex` | Color used by `ClearBuffer` and `CLS`. |
| `ReadByte` / `WriteByte` | VRAM (framebuffer index) access for `LDV`/`STV`. |