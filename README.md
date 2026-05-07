# Hex Sphere

Real-time hexagonal sphere generator with national-flag textures, OBJ + MTL
export, and a turntable MP4 recorder. Single-file standalone — vanilla
JavaScript + Three.js loaded via importmap, no build step.

A [GMT](https://gmt-fractals.com) tool by Guy Zack, GPL-3.0.

## Run locally

CORS rules block flag PNGs from loading off `file://`, so a local web server
is required:

```
cd hexsphere
npx http-server -p 8000 -c-1
# then open http://localhost:8000/
```

## Features

- **Topology**: 50-hex default, 8–500 supported. Fibonacci spiral seed →
  Thomson basin-hop multi-start. Always exactly 12 pentagons; pentagon
  clumping ≈ 0.04 at standard quality.
- **Quality presets**: `fast` (~1s), `standard` (~4s), `max` (~12s) — trades
  search effort for pentagon distribution uniformity.
- **Range scanner**: tries every hex count in ±25 around the current value
  and ranks them by clumping + topology cleanness.
- **Geometry pipeline** (per cell): chamfer → extrude → side walls → inset
  → cap. Every parameter re-renders against the cached topology in real
  time — no Thomson rerun on slider drags.
- **Per-cell flag overrides**: click a hex to pick a different flag, rotate
  it in 30° steps, or scale the UV. Click background to deselect.
- **Global flag transforms**: rotate / scale all flags from a single pair
  of knobs.
- **Two flag-fit modes**: `cover` (default — flag fills the cell, edges
  cropped) and `extend` (flag aspect preserved, edges clamped across the
  rest of the cell — same trick as `icosa-flag-mapper.html`).
- **Lit + shadows + cyclorama**: toggle the `lit` checkbox for a soft
  studio-style backdrop with a shadow plane underneath.
- **Shareable URLs**: `?hex=N&seed=K&quality=fast` reproduces a build
  exactly. Click `copy link` in the footer to grab the current state.
- **OBJ + MTL export** with adjustable scale (Cinema 4D defaults to cm —
  the `obj scale` field multiplies vertices to match). Triangulated or
  n-gon faces; `illum` switches between matte and specular shading.
- **Turntable MP4 export** via WebCodecs + mp4-muxer. Configurable
  duration, fps, resolution, and bitrate. Chrome / Edge only — Firefox /
  Safari don't ship `VideoEncoder` yet.

## Keyboard / mouse

| Action | Effect |
|---|---|
| left-drag | orbit camera |
| scroll | zoom |
| click cell | select for flag override |
| click background | deselect |
| Enter (in hex count field) | regenerate |
| Shift-click `generate` | re-roll the seed at the same hex count |

## Files

- `index.html` — the whole app.
- `flags/worldcup-flags/*.png` — 48 national flags. Texture paths in the
  exported MTL are relative, so the OBJ + MTL + this folder must travel
  together when importing into Cinema 4D.
- `PLAN.md` — engineering notes, architecture rationale, roadmap.

## License

GPL-3.0 — see GMT-fractals on GitHub for the full text.
