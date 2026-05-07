# Hexsphere — packaging plan

Hand-off doc for the standalone hexsphere app. Lives in
`h:/GMT/workspace-gmt/random-hexasphere/` as a separate git repo (separate
Cloudflare Pages deploy). The plan was packaged on 2026-05-07; deployment
prerequisites done. See §3 for status of each roadmap item.

---

## 1. Repository layout (shipped 2026-05-07)

The runtime app is `index.html` in this folder, served as the site root.
Flag textures live at `flags/worldcup-flags/*.png` (48 files) and are
loaded relative to the HTML so the OBJ + MTL + flags can travel together
when an artist imports into Cinema 4D.

`.gitignore` excludes the algorithm test harness (`hex-test/`), the OBJ
backup snapshots (`backup/`), the studio sources next to the runtime
flags (`flagTextures*.{jpg,psd}`, `icosa-layout.json`), and the original
single-file backup `hexsphere - Copy.html`.

---

## 2. Current state — what works

The app is functional and being used in production work. Single file, vanilla JS + Three.js via importmap (no build step). To run locally:

```
cd h:/GMT/workspace-gmt/hexsphere
npx http-server -p 8000 -c-1
# open http://localhost:8000/
```

### 2a. Topology (the hard part — locked)
- 50-hex default. Range scanner sweeps ±25 around the current count.
- Fibonacci spiral → Thomson basin-hop multi-start (M=15 starts × k=5 perturbations). Selection lexicographically by `[nonHexCount, clumping, evenness]`. ~3–5s per generate. 100% clean topology (always exactly 12 pentagons), clumping ≈ 0.04.
- Pentagon distribution metrics live in HUD: cell-count breakdown, clumping, evenness, min angular spacing.

### 2b. Flag rendering
- 48 PNGs from `flags/worldcup-flags/`. One material per flag in the browser (real GPU materials, not a single atlas).
- UV mapping: per cell, picks edge most "world-down", builds a local 2D basis, **rotates basis 30° CCW** (`FLAG_ROTATION_DEG` constant), then maps via **aspect-preserving cover** — flag image sits at native ratio (read from each loaded texture), excess cropped centrally on the long axis.
- Flag assignment: greedy max-min (`assignFlagsSpaced`). Every flag used before any repeat; duplicates pushed maximally apart on the sphere.

### 2c. Geometry pipeline (per cell, in order)
Five-stage pipeline runs on every cell each render. Topology is cached separately so geometry changes don't trigger Thomson.
1. **Chamfer** — replaces each cell vertex with a small bevel via two chamfer points + `chamferSub` Bezier intermediates (control point at the original vertex). Hex 6→2N(chamferSub+2) verts; UVs interpolated linearly between adjacent perim UVs.
2. **Extrude** — translates the (chamfered) perimeter along the cell normal; original perimeter remains at the sphere as a "base ring". Adjacent cells separate at shared edges (face-extrude, not radial). `extrudeScale` controls top-vs-base lateral scale (100% = vertical walls). `extrudeSub` adds intermediate rings inside the wall.
3. **Side walls** — between consecutive extrude rings. Tagged as `sideFaces` (between original cell-edge perim verts) vs `chamferFaces` (within chamfer block) so they can take different materials.
4. **Inset** — shrinks the top toward the cell centre by `reach` over `subdiv+1` divisions, with `height` raising the inner cap and `curve` shaping the rise (power law: `q = 2^(-2*curve)`).
5. **Cap** — closes the top as a fan-from-centre (triangulated) or a single n-gon (n-gon export).

### 2d. Filler triangles (closes chamfer holes)
At every Voronoi vertex (where 3 cells meet) the chamfer leaves a small triangular gap. `buildFillerTriangles` fans from the on-sphere vertex through each cell's chamfer block to close it. Always triangulated. Uses the `chamferMat` setting in the renderer; falls back to `chamfer_dark` when chamferMat = "stretched flag" (no single flag possible at a 3-cell junction).

### 2e. Materials
- **Per flag** (`flag_<country>`): textured `MeshBasicMaterial` (unlit) or `MeshStandardMaterial` (lit, `flatShading: true`, `roughness: 0.85`).
- **`pent_dark`**: `0x0a0a0a` for the 12 unavoidable pentagons + their walls + their chamfer.
- **`extrude_border`**: `0x0a0a0a` — used when `sideMode = "dark border"`.
- **`chamfer_dark`** / **`chamfer_light`** (`0x0a0a0a` / `0xeeeeee`) — used when `ch mat` is "dark" / "light" for hex chamfer walls + filler triangles.
- Stretched-flag modes (`sideMode = "stretched flag"`, `ch mat = "stretched flag"`) merge those faces into the cell's flag-textured mesh and re-use the perimeter UVs (flag's edge colour stretches across).

### 2f. Lighting
- **Default UNLIT** (per user). Toggle via `lit` checkbox.
- Three lights always in the scene (only affect `MeshStandardMaterial`): `HemisphereLight(0xffffff, 0x202020, 0.55)` + key `DirectionalLight` at `(2, 3, 2)` intensity 1.0 + fill `DirectionalLight` at `(-2, -1, -2)` intensity 0.25.
- `flatShading: true` keeps panel edges crisp instead of smoothing across shared verts. Vertex normals computed via `geometry.computeVertexNormals()` only when lit.

### 2g. UI controls (all reactive, no Thomson rerun)
- **Hex count + generate / scan / export OBJ buttons** (top-right).
- **Geometry sliders + paired number inputs** (right-side panel):
  - `chamfer 0–50%` / `ch sub 0–20`
  - `extrude -100..+100%` / `ex scale 0–200%` / `ex sub 0–20`
  - `inset 0–99%` / `sub 0–20`
  - `height -100..+100%` / `curve -100..+100%`
- **Dropdowns**:
  - `side` — "dark border" / "stretched flag"
  - `ch mat` — "dark" / "light" / "stretched flag"
- **Checkboxes** (custom-styled to match):
  - `lit` (default off)
  - `wireframe mesh preview` (default off)
  - `triangulate export` (default off — exports n-gons)

### 2h. Wireframe preview
Master toggle. When off, no wires at all. When on, every face edge of the rendered mesh — top faces + side walls + chamfer bevels + filler triangles. Respects `triangulate export` so n-gons read as clean polygons when that checkbox is unchecked. Wires lifted by `EDGE_LIFT = 1.002` to avoid z-fighting.

### 2i. OBJ + MTL export
- N-gon faces by default; triangulate-export toggle flips to fan-triangulation.
- `mtllib hexsphere.mtl`, per-flag `usemtl flag_<name>`, plus `pent_dark` / `extrude_border` / `chamfer_dark` / `chamfer_light` only when actually used.
- Texture paths in MTL: `flags/worldcup-flags/<name>.png` (relative to OBJ).
- Filler triangles included with `chamfer_*` material.
- Tested against C4D import.

---

## 3. Roadmap

### 3a. Hosting setup ✅ PARTIALLY SHIPPED (2026-05-07)
- ✅ `git init` + first commit done.
- ⏳ User to push to GitHub (suggested `gamazama/hexsphere`).
- ⏳ User to wire Cloudflare Pages (static site, no build, output `/`).
- ⏳ User to choose domain (`hexsphere.gmt-fractals.com` or
  `app.gmt-fractals.com/hexsphere/`).
- ✅ README mentions the file:// CORS gotcha + local-server recipe.

### 3b. About / sponsor links ✅ SHIPPED (2026-05-07)
Bottom-left footer in `index.html` with the same link set the GMT app
exposes through `app-gmt/HelpExtras.tsx`:
GMT homepage · Ko-fi · PayPal · GitHub source · copy-link share.
Plain HTML/CSS, monospace, matches existing panel aesthetic.

### 3c. ~~Lighting toggle~~ ✅ SHIPPED
`MeshStandardMaterial + flatShading`, three-light setup, default unlit.

### 3d. Turntable MP4 export ✅ SHIPPED (2026-05-07)
Modal dialog with duration / fps / width / height / bitrate.
WebCodecs `VideoEncoder` + `mp4-muxer` (loaded lazily via esm.sh on first
click). Encoder pattern mirrors `dev/engine/export/videoEncoder.ts` but
without the worker-style image-sequence path. Orbit controls are disabled
during capture and the renderer state (size, dpr, camera, target) is
saved/restored. Chrome/Edge only — Firefox and Safari don't ship
`VideoEncoder` yet; the dialog reports the missing API up-front.

### 3e. Texture loading progress ✅ SHIPPED (2026-05-07)
Centred overlay shows `loading flags N/48…` until decode completes; on
failures, the count of failed flags appears for 1.5s before the overlay
hides. Silent fast-path for the success case.

### 3f. Quality presets ✅ SHIPPED (2026-05-07)
Top-bar dropdown (`fast` / `standard` / `max`) maps to the basin-hop
constants from §4a. URL `?quality=fast` survives reloads.

### 3g. Shareable URL ✅ SHIPPED (2026-05-07)
`?hex=N&seed=K&quality=standard` encoded on every build via
`history.replaceState`. Footer "copy link" button writes the current URL
to the clipboard. Shift-clicking `generate` re-rolls the seed without
changing hex count, so the same hex count can yield a fresh arrangement.

### 3h. C4D export tweaks ✅ SHIPPED (2026-05-07)
- `obj scale` input added (default 100 — Cinema 4D's cm units).
- `obj illum` dropdown switches the MTL between `illum 1` (matte —
  default, cleanest for flag textures) and `illum 2` (with specular).
- Header comment in OBJ records the chosen scale + illum for traceability.

### 3i. Optional polish that emerged during prototyping
- The "fan-line clutter" in the filler triangles is visible in wireframe-mesh mode when `triangulate export` is on. Could add an n-gon mode to `buildFillerTriangles` that emits a single polygon around each Voronoi vertex (no centre vertex, no fan lines) — boundary already drawn by adjacent cells. Skipped for now since dev iteration didn't ask for it.
- Pentagon chamfer walls always render `pent_dark` (regardless of `ch mat`). Could expose this as a separate option but adds UI noise; current behaviour is intentional to keep the unavoidable Euler-defect cells visually distinct.

---

## 4. Architecture notes (critical — read before changing)

### 4a. Why basin-hopping Thomson, not Lloyd
Lloyd's relaxation gives **0% clean topology** (always heptagons + pentagon clumps). Thomson gives 100% clean. Basin-hopping (perturb best result + re-relax) cuts clumping ~24% over single-shot Thomson at moderate compute cost. **Do not revert to Lloyd-only** for the seed relaxation — it was the very first thing we tried and the results were unusable. See `hex-test/test*.mjs` for the comparison data.

### 4b. Why selection by clumping (not evenness)
`evenness = minPairAngle / icosahedral_ideal` peaks at 1.0 only for Goldberg numbers (N where 12 pentagons can sit at icosahedron vertices). For most counts in the 50-hex range it caps below 1.0 even with perfect uniform spacing. `clumping = stddev/mean` of pentagon nearest-neighbor distances directly measures "are pentagons spread out evenly", which is what the user actually sees. **Lexicographic order is `[nonHexCount, clumping, evenness]`** — clean topology first, then visual uniformity, then icosahedral closeness as final tiebreak.

### 4c. UV pipeline — order of operations matters
Flag aspect ratios are read from the loaded textures. UVs must be recomputed *after* the flag-assignment step, because each cell's UV depends on its assigned flag's aspect ratio. The current order in `build()` is: relax → identify hexes → assign flags → compute UVs. Don't swap.

### 4d. Geometry pipeline — chamfer happens FIRST
The five-stage pipeline (chamfer → extrude → side walls → inset → cap) only works if chamfer runs first. The chamfered perim has a fixed `blockSize = chamferSub + 2`; downstream code uses `i % blockSize !== blockSize - 1` to detect chamfer-bevel edges vs original cell edges so they can take different materials. If you re-order any stages, this tagging breaks.

### 4e. Cached topology, parameter-driven re-render
`build()` runs Thomson + assigns flags + computes UVs and stores everything in `lastBuildSnapshot`. `renderSnapshot()` only rebuilds geometry from the cached snapshot — no Thomson rerun. **All UI controls except hex count call `renderSnapshot()` directly** for real-time response. Don't accidentally call `build()` on a slider change; it'll re-Thomson and stutter for 5s.

### 4f. n-gon export — preserve face structure
Each cell exports as one n-gon face per top + one per side wall + one per chamfer bevel + n-gon filler at vertices, NOT a triangle fan. The internal triangulation is left to the host app (C4D). The `triangulate export` checkbox flips to fan-triangulation explicitly.

### 4g. Greedy flag spacing
`assignFlagsSpaced` enforces `maxUses = ceil(N / 48)` so every flag is used before any repeat. For 50 hexes, 46 flags get one use and 2 flags get two. The two duplicates are placed maximally apart (typically antipodal). Shuffle order seeded by `hexCount * 31 + 7` so same-count rebuilds give the same arrangement; change the seed if you want a re-roll.

### 4h. Filler triangles need cellGeoms (not the raw cell info)
`buildFillerTriangles` walks `cellGeoms[ci].verts[blockStart..blockStart + blockSize - 1]` to get each cell's chamfer-block at every Voronoi vertex. The chamfer Bezier intermediates are baked in, so the filler fan automatically follows the curved bevel. If you ever decouple the chamfer compute from `buildCellGeometry`, the filler builder needs equivalent access to the chamfered perim positions in 3D.

---

## 5. User feedback — extras (all SHIPPED 2026-05-07)

- ✅ Lighting with shadows + infinity-plane backdrop. The "lit" checkbox
  now also enables a soft cyclorama (BackSide sphere with a subtle
  vertical gradient) plus a transparent shadow-receiving floor plane at
  `y = -1.45`. Renderer toggles `shadowMap.enabled` based on the checkbox;
  `keyLight.castShadow` follows. ACESFilmic tone mapping for the lit path.
- ✅ Per-cell UV edits via click-to-select. The flag panel's "cell"
  section drives a `cellOverrides` map keyed by topology cell index:
  per-cell flag dropdown (all 48 alpha-sorted), rotate buttons (±30°,
  text input snaps to 30° increments), uniform UV scale slider. Yellow
  ring traces the selected cell on-sphere as visual confirmation.
- ✅ Global flag rotate / scale knobs at the top of the flag panel — fold
  into the per-cell transform so the user can punch up "all flags 90°,
  this one back 30°" combos.
- ✅ Flag-edge extension via the `fit` dropdown: `cover` (flag fills the
  cell, edges cropped) vs `extend` (flag fits inside the cell, edges
  clamp across the rest — same idea as `icosa-flag-mapper.html`'s
  drawImage trick). Texture wrapping on every flag is set to
  `ClampToEdgeWrapping` for the extend path. A 0.5% UV inset skips the
  PNG anti-aliased border so the clamped edge takes the inner pixel
  colour, not a dark hairline.
- ✅ Unbounded sliders. Number inputs no longer carry `min`/`max`; only
  the paired range inputs do. Type any value past the slider range and
  the renderer honours it; the slider thumb sticks at its boundary.

### Selection / interaction subtleties

- Picking is CPU-side: ray hits a slightly-inflated bounding sphere, then
  the nearest seed-point centre wins. No raycaster against meshes ⇒ works
  through any extrude / inset / chamfer combination without a separate
  picking mesh.
- Drag vs click is gated on a 4px movement threshold so OrbitControls
  drags don't deselect.
- Rebuild (different hex count) clears all overrides + selection because
  cell indices aren't stable across topology changes. Slider tweaks
  preserve them.

---

## 6. Reference files

- **Topology test data:** `h:/workspace-random/hex-test/` (test1.mjs through test9_compute.mjs).
- **GMT engine codec (for MP4):** `h:/GMT/workspace-gmt/dev/engine/codec/`.
- **GMT about menu:** check `h:/GMT/workspace-gmt/dev/engine/plugins/Help.tsx` and `dev/components/`.
- **GMT landing site (for visual style cross-reference):** `h:/GMT/workspace-gmt/landing/`.
