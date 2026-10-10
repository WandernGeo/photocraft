# Headless frame retouch for aligned face-swap frames

This note is a recipe for driving `photocraft-cli` on a folder of PNG or JPEG frames (a StarJam-style face swap: about 450 frames for 15 seconds). It records what was built and run on Linux on 2026-10-10, on synthetic portraits only. No real person's photo was used.

**Verdict.** The CLI is usable today as a scripted compositor: match a sequence's colour to one reference frame, and paste a known eye or mouth rectangle from an aligned good frame onto specific bad frames. It is not a face retoucher. There is no landmark detector, no face-aware liquify, and no learned heal. Spot healing left a glassy iris glassy. A fixed rectangle misses as soon as the head moves.

## Build (Linux, no GUI)

The workspace `rust-version` is 1.95 and the edition is 2024. The machine image's default `rustc` 1.83.0 cannot compile it. This run used stable `rustc 1.99.0 (b940084d7 2026-09-28)`.

```sh
rustup toolchain install stable --profile minimal
rustup default stable
cargo build --release -p photocraft-cli
```

| | |
|---|---|
| Command | `cargo build --release -p photocraft-cli` |
| Jobs | `CARGO_BUILD_JOBS=2` on 4 CPUs |
| Wall time | 5 min 33 s (cargo), 334 s clock, cold registry |
| Crates compiled | 256 |
| `target/` for this build | 970 MB |
| Binary | `target/release/photocraft-cli`, 42,334,728 bytes, not stripped |
| After `strip` | 36,758,688 bytes |
| Dynamic libs | `libc`, `libm`, `libgcc_s`, the dynamic linker |
| Version line | `photocraft-cli 0.6.0 (dev build)` |

No GTK, Wayland, or X11 development packages were required. `ldd` does not list them. The desktop app's Linux packages in `docs/development.md` are for the windowed binary, not this one.

## What was exercised

All of the following exited 0 against a 640×480 synthetic portrait (`bad.png`, a cool colour cast, glassy irises, a shifted mouth, and a hard cheek patch) and a clean `good.png` of the same drawing.

| Mode | Result |
|---|---|
| `run` | Each retouch below, one process per output |
| `batch` | One action list on 30 PNGs, 30 succeeded |
| `info`, `convert` | JSON document info; PNG → JPEG |
| `serve` (stdio) | `doc.open` → `engine.execute` → `doc.save`, then the process exits on stdin EOF |
| `serve` (TCP `127.0.0.1`) | First line must be `auth`. An earlier call returns `authentication required` and the server closes that connection |
| `mcp` (stdio) | `initialize`, then `doc_open` → `command_run` → `doc_export` |

`run` and `batch` use a trusted local filesystem, so `file.openAs` and `file.placeEmbedded` can read paths. `serve` and `mcp` do not, unless you pass both roots:

```sh
photocraft-cli mcp \
  --automation-read-root /work/frames \
  --automation-write-root /work/frames
```

Paths may be relative to that root or absolute paths underneath it. Omitting a root refuses that direction. Inside `serve` and `mcp`, `file.placeEmbedded` and `file.openAs` are rejected:

```text
automation command `file.placeEmbedded` uses ambient filesystem paths and is disabled; use capability-scoped document methods
```

Open files with `doc.open` / `doc_open`. Copy between open documents with `edit.copy` and `edit.pasteSpecial.pasteInPlace`. That path was run over MCP and restored the iris (below).

Flat PNG and JPEG exports bake the composite. The CLI warns `2 layer(s) flattened; layers, masks and blend modes are not kept`. That is the right output for a video frame. The in-memory document can still report `dirty: true` after a PNG `doc.save`, because only a layered save (PSD, PSB, OpenRaster, `.pcraft`) clears the saved revision. The PNG on disk is the flattened composite.

## Retouch results on the synthetic portrait

Canvas 640×480, 8-bit RGB. Opaque files open as a locked Background; the paint commands still ran. Mean absolute error (MAE) is against `good.png`, in 8-bit levels, over the whole frame (bad frame starts at 12.60) or over the defect rectangle. Iris figures are the mean RGB of an 8×8 patch in the left iris. The good iris is about `[41, 35, 33]`. The glassy iris is about `[228, 246, 255]`.

| Operation | Command | What happened |
|---|---|---|
| Spot heal, cheek | `paint.spotHealing` `contentAware` | Real inpaint, not a stub. Cheek MAE 64.1 → 40.2. The patch is still the wrong colour. |
| Spot heal, glassy eye | `paint.spotHealing` | Iris stayed `[219, 238, 255]`. Eye MAE 44.6 → 45.6 (slightly worse). Does not rebuild an iris. |
| Clone stamp, cheek | `paint.cloneStamp` | Copies pixels from a source point on the **same** frame. Cheek MAE 64.1 → 25.8. The copy still carries the frame's blue cast, because the source cheek is on the bad frame. |
| Healing brush | `paint.healingBrush` | Same source as the clone. Cheek MAE 64.1 → 40.0. Weaker than the clone here: it pulls the copy toward the surrounding (still blue) skin. |
| Patch | `paint.patch` | Selection plus `offset`. Cheek MAE 64.1 → 22.6. Feather on the selection is honoured. |
| Content-aware patch | `paint.patch` `contentAware: true` | Cheek MAE 64.1 → 19.9, the best same-frame texture fix. Only the 48×40 selection changed (1920 pixels), even though the result's `damage` box was larger. |
| Place + mask + feather | `file.placeEmbedded`, `select.rect` ellipses, `layer.layerMask.revealSelection`, `layer.layerMask.edit` | This is the eye/mouth repair. Left eye MAE 44.6 → 7.9, right eye 44.8 → 8.0, mouth 31.0 → 12.7. Iris `[47, 42, 40]` against `[41, 35, 33]`. Residual error is the feather mixing the good feature into the still-cool skin. |
| Levels (fitted) | `image.adjustments.levels` | Full-frame MAE 12.6 → 5.9. Mean RGB `[94.3, 92.1, 95.6]` against good `[94.4, 94.3, 102.1]`. Iris unchanged in structure. |
| Curves (fitted) | `image.adjustments.curves` | Full-frame MAE 12.6 → 5.6. Mean `[99.3, 92.1, 106.4]`. |
| Match Color | `image.adjustments.matchColor` | Full-frame MAE 12.6 → 4.5. Mean `[94.5, 94.4, 102.2]`. This is the one that tracks a reference without hand-fit curves. Glassy eyes stay glassy (`[233, 255, 231]`): the cast moves, the shape does not. |

`filter.liquify` runs. One `forwardWarp` stroke on the mouth changed 2032 pixels and reported `maxDisplacement` about 16 px in 8 ms. The strokes are coordinates you supply. Nothing detects a face or a smile.

`select.subject` on this portrait returned one region, `bounds` `[187, 69, 267, 333]`, covering the head. It did not return eyes or a mouth.

A wrong levels gamma makes a cast worse. In this engine a gamma above 1 lifts midtones. `blue.gamma: 1.35` on the cool frame moved mean blue from 125 to 145. Fit levels on a sample, or use Match Color.

## (a) Colour-match a sequence

`batch` opens each file in its own session, runs the action, and writes `<out>/<stem>.<ext>`. Match Color's `source` is a **document index in that session**, not a path. The action opens the reference as document 1, activates the frame (document 0) again, and matches. The reference is decoded once per frame; that cost is inside the timings below.

`actions-colormatch.json`:

```json
[
  ["file.openAs", {"path": "/absolute/path/to/reference.png"}],
  ["document.activate", {"document": 0}],
  ["image.adjustments.matchColor", {"source": 1, "luminance": 100, "intensity": 100, "fade": 0}]
]
```

```sh
photocraft-cli batch \
  --actions actions-colormatch.json \
  --in frames/ \
  --out frames-matched/ \
  --format png
```

JPEG output: `--format jpg --quality 92`. The same action works on a JPEG input folder.

Measured on this machine (4 CPUs), synthetic frames, reference reopened every file:

| Job | Time |
|---|---|
| 30 × 640×480, Match Color | 6.04 s (30/30), about 0.20 s/frame |
| One 1920×1080 frame, Match Color (image resized, one shot) | 0.93 s |

Mean RGB after the 640×480 batch sat on the reference (`[94.4, 94.3, 102.1]`) for a clean frame, a mid-cast frame, and the most cast frame. A frame that also had the glassy eyes and cheek patch only fell from MAE 6.1 to 3.2: the colour moved, the defects stayed.

Arithmetic from the single 1080p timing, not a measured 450-frame run: 450 × 0.93 s is about 7 minutes for a 1080p colour pass on this machine. A 4K plate will be slower. Confirm on one real frame before launching the folder.

Hand-fit curves, when you already know the cast and do not want to reopen a reference (this cast only):

```json
[
  ["image.adjustments.curves", {
    "red": [[0, 0], [128, 148], [255, 255]],
    "blue": [[0, 0], [128, 108], [255, 230]]
  }]
]
```

Levels that reduced the same cast (MAE 12.6 → 5.9):

```json
{"red": {"inWhite": 230}, "blue": {"inBlack": 20, "gamma": 0.8}}
```

```sh
photocraft-cli run bad.png \
  --cmd image.adjustments.levels \
  --params '{"red":{"inWhite":230},"blue":{"inBlack":20,"gamma":0.8}}' \
  --out bad-levels.png
```

Those numbers are a fit to this fixture. On a real shot, sample the reference and the frame (or just use Match Color).

## (b) Paste a masked eye and mouth from a reference

This assumes the reference and the bad frame share a canvas and the face is already in the same place. `fit: false` keeps a same-size reference at 100% instead of scaling it down. Origin is the top-left. `x,y,width,height` are document pixels. Ellipses plus `feather` on the selection, then another feather on the layer mask, are what produced the eye MAE drop above.

`actions-paste-eyes.json` (coordinates from the synthetic portrait; replace them):

```json
[
  ["file.placeEmbedded", {"path": "/absolute/path/to/reference.png", "fit": false}],
  ["select.rect", {"x": 248, "y": 188, "width": 64, "height": 40, "ellipse": true, "feather": 8}],
  ["select.rect", {"x": 328, "y": 188, "width": 64, "height": 40, "ellipse": true, "feather": 8, "mode": "add"}],
  ["select.rect", {"x": 276, "y": 292, "width": 88, "height": 36, "ellipse": true, "feather": 6, "mode": "add"}],
  ["layer.layerMask.revealSelection", {}],
  ["layer.layerMask.edit", {"feather": 3}]
]
```

`batch` runs that on **every** file in the folder. It has no frame-number filter. For frames 12, 18, and 40, either move those files into their own folder or call `run`:

```sh
for n in 0012 0018 0040; do
  photocraft-cli run "frames/frame_${n}.png" \
    --cmd file.placeEmbedded --params '{"path":"/absolute/path/to/reference.png","fit":false}' \
    --cmd select.rect --params '{"x":248,"y":188,"width":64,"height":40,"ellipse":true,"feather":8}' \
    --cmd select.rect --params '{"x":328,"y":188,"width":64,"height":40,"ellipse":true,"feather":8,"mode":"add"}' \
    --cmd select.rect --params '{"x":276,"y":292,"width":88,"height":36,"ellipse":true,"feather":6,"mode":"add"}' \
    --cmd layer.layerMask.revealSelection \
    --cmd layer.layerMask.edit --params '{"feather":3}' \
    --out "out/frame_${n}.png"
done
```

Measured: the paste action on all 30 synthetic 640×480 frames took 4.39 s (about 0.15 s/frame, 30/30). Four named frames via the loop took 0.71 s, and the repaired iris mean was `[47, 42, 40]`. One 1920×1080 place-and-mask (one ellipse) took 0.40 s. 450 such frames would be on the order of 3 minutes from that single timing, only while the rectangle still covers the eyes.

Same-frame cheek repair, when the good skin is on the bad frame itself (clone), and when you would rather heal a selection from an offset (patch):

```sh
photocraft-cli run bad.png \
  --cmd paint.cloneStamp \
  --params '{"points":[[372,248],[384,256],[396,264]],"source":[244,252],"size":42,"hardness":70}' \
  --out cheek-clone.png

photocraft-cli run bad.png \
  --cmd select.rect --params '{"x":360,"y":236,"width":48,"height":40,"feather":2}' \
  --cmd paint.patch --params '{"offset":[-128,4],"mode":"source"}' \
  --out cheek-patch.png
```

`source` is the document point sampled under the first brush point. `offset` is how far the patch selection is dragged, in pixels. `mode: "destination"` heals the destination instead of the selection. `contentAware: true` with `structure` 1..7 and `color` 0..10 is the content-aware patch; it was the best same-frame cheek fix here and is still not an eye rebuild.

## MCP tool calls

Newline-delimited JSON-RPC on stdin. Start:

```sh
photocraft-cli mcp \
  --automation-read-root /work/frames \
  --automation-write-root /work/frames
```

Handshake, then a levels round trip that was run against `bad.png` (export MAE matched the CLI levels result, 5.89):

```json
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"starjam","version":"0"}}}
{"jsonrpc":"2.0","method":"notifications/initialized"}
{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"doc_open","arguments":{"path":"/work/frames/bad.png"}}}
{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"command_run","arguments":{"id":"image.adjustments.levels","params":{"red":{"inWhite":230},"blue":{"inBlack":20,"gamma":0.8}}}}}
{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"doc_export","arguments":{"path":"/work/frames/out/bad-levels.png","format":"png"}}}
```

`doc_open` returned `{"index":0,"width":640,"height":480,"layers":1,...}`. `command_run` returned `isError: false`. `doc_export` returned the path and an empty warning list for a single-layer levels edit.

Match Color over MCP, without `file.openAs`: open the reference, open the frame (it becomes active), pass the reference index.

```json
{"jsonrpc":"2.0","id":5,"method":"tools/call","params":{"name":"doc_open","arguments":{"path":"/work/frames/reference.png"}}}
{"jsonrpc":"2.0","id":6,"method":"tools/call","params":{"name":"doc_open","arguments":{"path":"/work/frames/bad.png"}}}
{"jsonrpc":"2.0","id":7,"method":"tools/call","params":{"name":"command_run","arguments":{"id":"image.adjustments.matchColor","params":{"source":0,"luminance":100,"intensity":100,"fade":0}}}}
{"jsonrpc":"2.0","id":8,"method":"tools/call","params":{"name":"doc_export","arguments":{"path":"/work/frames/out/bad-matched.png"}}}
```

`source` is the index from the first `doc_open`. If other documents are already open, use the index in that reply, not a hard-coded 0.

Masked paste over MCP (`file.placeEmbedded` is disabled here). Copy the whole reference, paste it in place on the bad frame, then mask. This was run; the exported iris matched the CLI place-and-mask result (`[47, 42, 40]`).

```json
{"jsonrpc":"2.0","id":10,"method":"tools/call","params":{"name":"doc_open","arguments":{"path":"/work/frames/reference.png"}}}
{"jsonrpc":"2.0","id":11,"method":"tools/call","params":{"name":"command_run","arguments":{"id":"edit.copy","params":{}}}}
{"jsonrpc":"2.0","id":12,"method":"tools/call","params":{"name":"doc_open","arguments":{"path":"/work/frames/bad.png"}}}
{"jsonrpc":"2.0","id":13,"method":"tools/call","params":{"name":"command_run","arguments":{"id":"edit.pasteSpecial.pasteInPlace","params":{}}}}
{"jsonrpc":"2.0","id":14,"method":"tools/call","params":{"name":"command_run","arguments":{"id":"select.rect","params":{"x":248,"y":188,"width":64,"height":40,"ellipse":true,"feather":8}}}}
{"jsonrpc":"2.0","id":15,"method":"tools/call","params":{"name":"command_run","arguments":{"id":"select.rect","params":{"x":328,"y":188,"width":64,"height":40,"ellipse":true,"feather":8,"mode":"add"}}}}
{"jsonrpc":"2.0","id":16,"method":"tools/call","params":{"name":"command_run","arguments":{"id":"select.rect","params":{"x":276,"y":292,"width":88,"height":36,"ellipse":true,"feather":6,"mode":"add"}}}}
{"jsonrpc":"2.0","id":17,"method":"tools/call","params":{"name":"command_run","arguments":{"id":"layer.layerMask.revealSelection","params":{}}}}
{"jsonrpc":"2.0","id":18,"method":"tools/call","params":{"name":"command_run","arguments":{"id":"layer.layerMask.edit","params":{"feather":3}}}}
{"jsonrpc":"2.0","id":19,"method":"tools/call","params":{"name":"doc_export","arguments":{"path":"/work/frames/out/bad-eyes.png"}}}
```

Copying a feathered eye selection alone, then `edit.pasteSpecial.pasteInPlace`, also restored the iris (`[44, 39, 37]`) and left the rest of the frame untouched. Use that when you do not want a full-frame layer.

`serve` speaks a smaller envelope, `{"id","method","params"}` → `{"id","ok","result"|"error"}`. Same roots. Useful methods include `doc.open`, `doc.save`, `doc.select`, `engine.execute`, `session.list`, `batch`, `methods`.

```json
{"id":1,"method":"doc.open","params":{"path":"/work/frames/bad.png"}}
{"id":2,"method":"engine.execute","params":{"command":"image.adjustments.matchColor","params":{"source":0}}}
{"id":3,"method":"doc.save","params":{"path":"/work/frames/out/bad.png","format":"png"}}
```

TCP (`photocraft-cli serve --port 9876 --control-token <64 hex> --automation-read-root ... --automation-write-root ...`) requires `{"id":"auth","method":"auth","params":{"token":"..."}}` before any other method.

## What is missing for this job

These are absent. They are not hidden behind a flag that this run failed to find.

- **No face landmarks.** Nothing returns eye corners, iris centres, or lip contours. Rectangles are yours to measure.
- **No face-aware liquify.** `filter.liquify` replays brush strokes (`forwardWarp`, `pucker`, `bloat`, `twirlCw`, and the rest) at the points you pass. A warped mouth stays warped unless you author the stroke.
- **No learned heal.** `paint.spotHealing` implements `contentAware` (classical completion, with a membrane fill fallback), `createTexture`, and `proximityMatch`. On the glassy iris, `contentAware` did not invent a pupil or a catchlight.
- **No tracking or temporal model.** Each file is independent. Neighbouring frames stay consistent only when you apply the same reference, the same curves, and the same rectangles. There is no optical flow.
- **`batch` cannot take a frame list.** One action, every image in the directory. Named frames are a shell loop or a separate folder.
- **`serve` / `mcp` cannot place or open by engine path.** `file.placeEmbedded` and `file.openAs` are refused. Use `doc_open` plus copy/paste, and Match Color's document index.
- **Same-frame clone and patch cannot borrow another frame's eye.** They sample the document they are editing.
- **`select.subject` is not a face parser.** On this portrait it selected the head as one blob.

`paint.healingBrush`, `paint.cloneStamp`, `paint.patch`, `paint.spotHealing`, `image.adjustments.levels`, `image.adjustments.curves`, `image.adjustments.matchColor`, `file.placeEmbedded`, and the layer-mask commands are implemented. They returned damage rectangles or changed the pixels as in the table. They are not stubs.

## Should you use it for StarJam frames right now?

Use it for two narrow jobs, with a person or an upstream detector supplying rectangles:

1. Pull a sequence's colour toward one good frame (`batch` + Match Color). That part works, and a 1080p frame was under a second here.
2. Cover a bad eye or mouth with the same region of an aligned good frame (place or paste, elliptical selection, mask feather). On this fixture that restored the iris. It will miss when the face is not in the same place.

Do not expect a folder pass to repair glassy eyes, a warped mouth, or a face patch that disagrees with the next frame. Spot healing, the healing brush, and Match Color all left the glassy iris a bright disc. Liquify will not find the mouth for you. Anything that depends on landmarks, identity, or a generative fill has to live outside this binary.
