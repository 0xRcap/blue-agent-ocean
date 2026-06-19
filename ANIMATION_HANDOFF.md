# Open Sea — Animation Handoff

Everything needed to continue the **Open Sea** 3D map animation. Scope: the animation only
(the Harbor map visualization). Not the agent / Curator / protocol work.

---

## 1. What it is

An interactive WebGL "sea world" map for Sail. A flat animated ocean (the brand's hero sea
shader) seen from an aerial 3/4 view. On it:

- **Sailboats = SMAs (agent accounts).** Voxel-extruded 3D versions of the Sail pixel logo, glowing blue.
- **Islands = DeFi protocols** (AAVE, UNISWAP, MORPHO, GMX, AERODROME) — irregular dark landmasses with beacons.
- **Seas = networks** (Base, Arbitrum, Unichain, Ethereum) — organic, domain-warped tinted regions.
- Boats **sail to protocol islands** (currently mock activity), dock, then pick a new target.
- Click a boat → camera **follows** it + a glass **vessel panel** shows its bounded permissions.
- Left **fleet legend** filters boats; **search** by name; **add-your-ship** chrome with X-share.
- Navigation: **drag rotate, scroll zoom, right-drag/2-finger pan** (OrbitControls).

---

## 2. Files & how to run

| File | What |
|---|---|
| `harbor-preview/index.html` | The entire animation (single file, plain WebGL/Three, no build step) |
| `harbor-preview/boat-grid.js` | The Sail pixel-boat shape as a 0/1 grid (`window.BOAT_ROWS/BOAT_GRID/BOAT_W/BOAT_H`) |
| `harbor-preview/Logo-blue.png` | Source logo (white boat on blue) the grid was traced from — kept for reference |

**Run locally:**
```bash
cd harbor-preview
python3 -m http.server 8787
# open http://localhost:8787  (use Chrome — see Safari note below)
```
No bundler. Edit `index.html`, refresh. There's an on-page error overlay (red text) + a
try/catch around the render loop that prints the real error (see §7).

---

## 3. Tech stack / dependencies (all via CDN, no install)

- **three.js r128** — `cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js`
- **examples/js (UMD)** from `cdn.jsdelivr.net/npm/three@0.128.0/examples/js/...`:
  `controls/OrbitControls.js`, `postprocessing/EffectComposer.js`, `RenderPass.js`, `ShaderPass.js`,
  `UnrealBloomPass.js`, `shaders/CopyShader.js`, `shaders/LuminosityHighPassShader.js`
- Fonts: Google Fonts — JetBrains Mono + Instrument Sans.

Pinned to r128 because the examples/js UMD globals (attach to `THREE.*`) match that core. If you
upgrade three, switch to ES modules + import maps (examples/js is removed in newer versions).

---

## 4. Brand constraints (from the Sail "Electric blueprint" design manual)

Apply these to anything visual:
- **Dark only.** Near-black field. No light mode.
- **One accent hue:** Sail blue `#1990FF` (bright `#4DABFF`, pressed `#0E78EA`). Positive = blue, never green.
- **Sharp corners** (≤3px; chips/pills banned). **Hairlines** (1px, low-alpha white) not heavy borders.
- **Glow rationed** to the live elements (boats, beacons, moon, specular) — not everything.
- **Mono uppercase labels** (JetBrains Mono, wide tracking). Section labels = blue index + grey name.
- **No em-dashes** in visible copy. Use ` · `, commas, periods.
- Pixel-art boat stays hard-edged.

---

## 5. Architecture (inside `index.html`)

Scene graph (added to `sc` in this order — `sc.children[0]` is the sky dome, relied on in the loop):
1. **Sky dome** — big inverted sphere, `skyColor()` shader (night sky + stars + moon).
2. **Sea plane** — `PlaneGeometry(1200,1200,380,380)` rotated flat; ShaderMaterial:
   - vertex displaces `y += seaHeight(xz*WS)*AMPY`, computes normal from x/z finite differences.
   - fragment = hero-sea shading: fresnel→`skyColor(reflect)`, moon specular, deep-water color,
     **network tint** via domain-warped Voronoi (4 seed points = networks).
3. Lights (ambient + key + warm rim).
4. **Islands** (`PROTO` array) — `buildIsland()` irregular flattened icosphere + halo + beacon + HTML label.
5. **Sea lanes** — `Points` along `edges` between islands, additive, pulsing.
6. **Network labels** — HTML divs projected each frame.
7. **Boats** (`V` map → `makeBoat()`): per boat an `InstancedMesh` of the pixel cells + invisible
   hit-sphere (`hits[]` for raycast) + glow sprite + wake `Line`.

Key systems:
- **Hero sea shader** lives in the `CG` GLSL string (shared by sky + sea). Extracted verbatim from
  `SailLandingPage` / `sail-landing` `WaterFloatingUI.jsx` (the "Seascape" raymarch, adapted to a plane).
- **JS wave port** (`seaHJ`/`waveY`) replicates `seaHeight` so boats ride the *exact* rendered surface.
- **Pixel boat**: `cells[]` built from `BOAT_GRID`; `cellGeo` box per filled cell via InstancedMesh.
- **Interaction**: `cv` click → raycast `hits` → `show(name)` panel + `follow=true`.
- **Camera**: `OrbitControls`; `follow` lerps `controls.target` to the selected boat until the user drags.
- **Filter**: `activeFleet` + `searchStr` → `applyFilter()` sets `boat.userData.active`; inactive boats
  dim + shrink (handled in `updBoat`).

---

## 6. Tuning knobs (where to change look/feel)

| Want to change | Where |
|---|---|
| Wave size / choppiness | `WS` (0.2, wavelength) and `AMPY` (1.3, height) — **must match shader + JS** |
| Boat size | `makeBoat()` `b.scale.setScalar(0.8)` and the `tsc` in `updBoat` |
| Boat glow / wake | glow sprite scale + `wmat` opacity in boat creation; `updBoat` glow/wake |
| Camera default | `cam.position.set(0,130,165)` + `controls.target.set(0,0,-25)` + min/maxDistance/maxPolarAngle |
| Bloom strength | `UnrealBloomPass(res, 0.6, 0.65, 0.3)` (strength, radius, threshold) |
| Network seas | the 4 `s0..s3` seed points + `c0..c3` tints + warp amount (80.0) in the sea fragment |
| Islands | `PROTO` array (name,x,z), `buildIsland(13)` shape, halo/beacon sizes |
| Sea lanes | `edges` array + `laneMat` |
| Activity speed / dock time | `updBoat`: `u.spd`, dock threshold `16`, dock duration `2.5` |

---

## 7. Gotchas / lessons (READ before editing — these cost real time)

1. **Safari crashes, Chrome is fine.** Safari's WebGL chokes on the bloom `EffectComposer` render
   targets and `RoomEnvironment`/PMREM → context loss → `createShader returns null` →
   `"shaderSource must be an instance of WebGLShader"`. For Safari support: feature-detect and
   **drop bloom + lower pixel ratio**. Currently targeted at Chrome.
2. **Cross-origin error masking.** Errors thrown inside CDN three.js report as `"Script error."` to
   `window.onerror`. The render loop is wrapped in try/catch so it can read the *real* message — keep that.
3. **The OCT_M transpose bug (the big one).** GLSL `uv *= OCT_M` is `vector * matrix` = the
   **transpose** of `matrix * vector`. The JS port (`seaHJ`) MUST use `nx=1.6*ux+1.2*uy; ny=-1.2*ux+1.6*uy;`
   If you get it wrong, boats float for ~2s then drift off the water (error grows with the time term). Both
   sea height functions (shader + JS) must stay identical.
4. **`mesh.position` is read-only** — never `Object.assign(mesh,{position:...})`; set `mesh.position.y=...`.
5. **`BufferGeometryUtils.mergeBufferGeometries` was flaky** — boats use `InstancedMesh` instead. Keep it.
6. **Boats on a sphere tip at the limb** (a boat on the side of a ball points sideways). We pivoted to a
   **flat map** to kill this. Don't go back to a sphere unless you fade boats at the limb.
7. **Boats billboard toward the camera** (yaw only) so they always read; heading drives movement, not facing.
8. **Performance:** cap `pixelRatio` at 2, water at ~380² segments, use InstancedMesh, keep bloom modest.
   The water vertex shader is heavy (multiple `seaHeight` calls/vertex) — don't raise segments much.

---

## 8. Mock → live data (what to wire when real data exists)

Everything visual is driven by placeholder data; replace these:

| Mock today | Live source |
|---|---|
| `V{}` map (8–10 hardcoded ships, fleet/network/permissions/activity) | on-chain SMA registry (scan SailKernel) + opt-in metadata DB |
| Boat `tgt` = random island | the protocol the agent is actually transacting with (live activity feed) |
| `PROTO` islands (5 hardcoded) | real protocol set + usage/TVL (size/lit by activity) |
| Network seas (4 fixed seeds) | real networks + volume/agent counts |
| `a` (voyages) / treasury | real on-chain stats |
| Add-your-ship form | connect wallet → SIWE sign → claim flow |

The killer feature when live: **boats sail to the island of the protocol their agent just used** — the
map becomes a real-time view of the whole fleet's on-chain activity. The `updBoat` targeting is already
structured for this (just set `u.tgt` from real events instead of random).

---

## 9. Next steps (animation roadmap)

- **Wire live activity** (ship → protocol targeting from real on-chain events). Highest leverage.
- **Cinematic intro**: blueprint-on → title → settle into the interactive map.
- **Craft polish** (needs a real-time loop / 3D dev, hard to do blind): water reflections of islands/boats,
  depth of field on the focused ship, bow-spray foam, coastline foam, god-rays.
- **Mobile + reduced-motion + 2D fallback** (card-grid Harbor sharing the same data model) for low-end/Safari/SEO.
- **Selection polish**: sonar ring on select, smooth camera dolly-in.
- **Instancing/LOD** for hundreds of ships.

---

## 10. Source references

- Hero sea shader: `WaterFloatingUI.jsx` ("Seascape" raymarch) in repos
  `github.com/sail-money/SailLandingPage` and `github.com/0xRcap/sail-landing`.
- Real 3D boat geometry (not used now, but reference): `SailboatScene.jsx` in `0xRcap/sail-landing`.
- Pixel boat traced from `Logo-blue.png` (white boat on blue).
- Design language: Sail "Electric blueprint" design manual (dark, one blue hue, hairlines, sharp corners).

> Build/iterate in Chrome. To debug a blank screen, check the red on-page error overlay first; if it
> says "Script error", the real message is being printed by the loop's try/catch — read that line.
