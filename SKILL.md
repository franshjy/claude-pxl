---
name: pxl-pixel-art
description: Create pixel art (game sprites, characters, items, tiles, tilesets, icons, scenes, sprite sheets) with pxl, a compact JavaScript DSL that renders PNGs and lets the user watch the painting live in a browser. Use when asked to draw, design or iterate on pixel art or game assets.
metadata:
  opencode/autoinvoke: false
disable-model-invocation: true
---

# pxl: pixel art as code

You write a small `.px.js` file; pxl renders PNGs and upscaled previews you can look at. The DSL
is built to avoid writing pixels twice: mirroring, auto-outline, palette-swap variants, reusable
parts, primitives, brushes and tilemaps. Needs Node ≥ 22.18. No install step.

`PXL` below means `node <this skill's directory>/bin/pxl.js`.

## Workflow

1. Write `<subject>.px.js`, named after what it draws (`sword.px.js`), not `art.px.js` (start from
   the template below; examples in `examples/`). `art.px.js` below stands for that file.
2. Build only what you're working on: `PXL build art.px.js --only slime,heart`
3. **Look at it**: read `out/_preview/<name>.png` (upscaled, checkerboard = transparent, grid every
   pixel, darker every 8). For exact pixel fixes use `PXL ascii art.px.js <name>` (keys + x/y rulers);
   it's cheaper than an image.
4. Fix with small edits (a `grid(x, y, ...)` patch, one op) and rebuild. Repeat.
5. **Check before you report**: you can't watch a GIF play, so verify instead:
   - Game size: `PXL build art.px.js --only <name> --scale 3` (no grid below 6x), read the preview,
     then rebuild without `--scale`.
   - Size: the build line prints `WxH`; it must match what was asked (see outline below).
   - Animation: the build line prints the frame count and loop length; `PXL ascii` prints each
     frame with its ms. Compare frames to confirm only the intended pixels change.
   Tell the user what you checked and that they should watch the GIF for timing.
6. If the user wants to watch: start `PXL serve art.px.js` in the background and give them the URL.
   It rebuilds on every save and replays the edited asset's painting steps; it also writes the same
   `out/` files, so keep reading previews as usual. To show several files at once, serve the
   folder (`PXL serve <dir>`): each `.px.js` loads on its own, and the **gallery** toggle plays every
   sprite and anim together, grouped by name prefix (`bracelet`, `bracelet_shine`); the filter box
   takes globs (`bracelet*`). One server per port (default 4321; pass `--port` if busy). Stop it
   when the user is done.

## Files

- Output always goes to `<file dir>/out/{sprites,anims,scenes,_preview}/<asset name>.*`. Files are
  named by asset, not by source file, so asset names must be unique across every `.px.js` in a
  folder, or one build silently overwrites another's output.
- Keep a subject's variants, parts and animations in its own file. To view or sheet several files
  together, make an index file that `include`s them (each keeps its own palette).

Errors name the file:line:col, the asset/layer/op path, and suggest fixes ("did you mean 'r'?").
One broken asset doesn't stop the others from building.

## Template

```js
pal({ k: "#1b1630", g: "#8ccf63", G: "#4c8a44", t: "#2a5560", w: "#fff6e0", r: "#e2585e", R: "#a63a4c" })
ramps("ktGg", "kRr")                   // dark -> light, drives shade()/light()

// Symmetric: write the left half, mirror it, outline it.
sprite("slime", { mx: true, outline: "k" }, `
  ....ggg
  ..ggGGg
  .ggGwGg
  gggkwgg
  gggkkgg
  tgggggg
  .tttttt`)
sprite("slime_red", { base: "slime", swap: { g: "r", G: "R" } })   // variant: 1 line

sprite("_eye", "wk\nkk")                                            // "_" = part, not exported
sprite("chest", { w: 16, h: 12, outline: "k" },
  rect(0, 3, 16, 9, "R"), rect(0, 0, 16, 4, "r"), stamp("_eye", 7, 4), shade([0, 9, 16, 3]))

scene("meadow", { w: 160, h: 96, bg: "w" },
  layer("ground", { y: 64 }, rect(0, 0, 160, 32, "G"), noise(0, 0, 160, 32, "g", 0.1, 3)),
  layer("props", stamp("slime", 40, 54), stamp("slime_red", 70, 54, { flip: "x" })),
  layer("shadow", { shade: true }, poly([38, 64, 88, 64, 92, 67, 34, 67], "k")))   // cast shadow

sheet("chars", ["slime*", "chest"], { pad: 1 })
```

## Essentials

- **Colors**: one-char keys from `pal`, `"#hex"`, `"."` skip, `"_"` erase. Default palette is pico8
  (`0-f`); `use("sweetie16" | "db16" | "endesga32" | "gray8")` for others.
- **Items run in order**: grid strings (drawn at 0,0), ops, `(c) => { c.rect(...) }` functions,
  `layer(...)`s. Then mirror → swap → flip → rot → outline (grows 1px each side) → crop.
- **Exact sizes**: because `outline` grows the canvas, a 32×32 outlined sprite is drawn at 30×30:
  `sprite("gem", { w: 30, h: 30, outline: "k" }, ...)` → 32×32. Or draw an outlined `_part` and
  stamp it into an `anim`/`sprite` with a fixed `{ w: 32, h: 32 }`.
- **Shapes**: `px rect box line(…, width) hline vline circ ring ellipse poly fill wire text`.
- **Tone**: `dither grad noise`, `shade/light(region, steps, density)`, `recolor`, and
  `shade:`/`light:` layers for cast shadows and glows. Regions: `[cx,cy,r]`, `[x,y,w,h]`,
  polygon, or a function. `clip(region, ...items)` restricts any items (half shapes, slopes).
- **Reuse**: `stamp tile pattern tilemap scatter`. Names can be groups or globs
  (`group("rocks", ["rock_a", "rock_a", "rock_b"])`, `"tree_*"`). Each placement picks randomly but
  reproducibly. Use `rflip`, `rswap`, `seed`, and `reseed: true` for per-copy variety.
- **Brushes**: `scatter("_leaf*", x, y, w, h, 30, { on: "g", clip: true, rflip: "xy" })` paints
  texture with tiny part sprites, only on the given colors.
- **Animation**: `anim("cat", { outline: "k", fps: 10 }, grid, frame(), frame(grid(23, 0, "_K")))`:
  frames are patches over shared items or a `base`; or `frames: 6` + `(c) => ... c.phase ...` for
  procedural loops. Stamping an anim animates the scene (`rdelay: true` desyncs copies, `frame: n`
  freezes one). Check `_preview/<name>.png` (all frames side by side) or `pxl ascii` per frame.
- **Glow**: `glow("zgQ", { r: 6, strength: 0.5 })` directly in a sprite/scene body gives every pixel
  of those keys a halo (flames, potions, moons, neon); `ramp: 2` instead stays in the palette.
  Give glowing things their own keys so nothing else lights up, or limit it with `where`.

Full API with every option: [reference.md](reference.md).
Dynamic lighting (lamps, light blocked by solid layers, sky light, Terraria-style): read
[lighting.md](lighting.md) only when a scene needs lights.

## Habits that save tokens and give better art

- Write half of anything symmetric (`mx`) and let `outline` draw outlines. Never hand-draw either.
- Recolors are `base` + `swap`; repeated details are `_parts` + `stamp`; families are a `group` or a
  loop: `for (let i = 1; i <= 3; i++) sprite("tree_" + i, ...)`.
- Large areas: primitives, `grad`, `noise`, `pattern`, `tilemap`. Never big hand grids.
- Organic texture (leaves, grass, stones, rust): 2–3 tiny brush parts + `scatter` with `on`/`clip`.
- **No filler noise.** Don't sprinkle `noise` speckle over walls, floors or plaster to make them
  "less flat": it reads as random dots, not material. Every mark on a surface needs a cause you
  could name, and it should sit where that cause would put it:
  - built structure: siding/plank lines, brick or stone courses with offset joints, tile seams;
  - weathering: rain-stain streaks under ledges and sills, splash-back grime fading up from the
    ground, rust runs under metal brackets, paint flaking off boards (following the board lines);
  - clutter that tells the story: posters, pipes, vents, cracks from a corner.
  Flat color is fine; a clean wall beats a dotted one. `noise` is OK only where the material itself
  is granular (gravel, sand, asphalt, tar roofs, rough stone), and keep it sparse. On faces drawn
  at an angle (3/4 views), make the marks follow the face's slant.
- Lighting: define `ramps`, then `light`/`shade` regions; stack a `density: 0.5` pass for soft edges.
  Use `shade: true` layers with a `poly` for cast shadows across many objects.
- Use mnemonic keys (`g`/`G` light/dark green). Keep 2–4 shades per material for a clean pixel-art look.
- Build with `--only` while iterating; a full build also writes every preview.

## Art rules

- **Silhouette first**: it must read when filled with one color, and at 1x. Cut detail that vanishes
  at real size before adding more.
- **One light source** (default top-left): `light` the lit side, `shade` the far side and the
  bottom. Never darken every edge evenly (pillow shading).
- **Outlines**: pick one style for a whole set (none, outer `outline`, or outline plus internal
  lines) and keep it the same on every sprite.
- **Frames**: fewer key poses beat many in-betweens. Typical counts: idle 4, walk 6 (4 is the
  minimum), run 6, jump 3, attack 4, hurt 2, death 5.
  - Walk: contact (front leg out, arms opposite, body lowest), passing (leg under body, body
    highest), then the same two with the legs swapped. Write the 2nd half as `frame()` patches.
  - Attack: anticipation (1–2 frames, ~120ms), a fast action (1–2 frames, ~50ms), a held contact
    (~100ms), then a slower recovery (~80ms). Use `frame({ ms })` per phase.
- **Timing** per frame: 50–80ms impacts, 80–120ms run, 120–180ms walk, 200–300ms idle, 300–500ms
  ambient loops.
- **Tiles**: build them with `tile`/`pattern` in a test scene and check the seams before shipping.
- **In the engine** (when wiring sprites into a game): integer scaling only (2x, 3x, never 1.5x),
  crisp rendering (no smoothing, e.g. Phaser `pixelArt: true`), PNG only.
