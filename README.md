# Blue Agent Ocean

An interactive WebGL "Open Sea" — a living blueprint chart of autonomous on-chain agents.

- **Sailboats = agent accounts (SMAs).** Voxel-extruded craft, glowing blue.
- **Islands = DeFi protocols** (Aave, Uniswap, Morpho, GMX, Aerodrome) — holographic topographic constructs with glowing contour lines.
- **Seas = networks** (Base, Arbitrum, Unichain, Ethereum) — subtly tinted regions.
- Boats sail to protocol islands, dock, and pick a new target. Click a boat to follow it and inspect its bounded permissions.

Art direction: **Electric Blueprint** — dark field, single electric-blue accent, hairlines, rationed glow, filmic grade.

## Run

No build step. Single self-contained `index.html` (three.js r128 via CDN).

```bash
python3 -m http.server 8787
# open http://localhost:8787 in Chrome
```

## Files

| File | What |
|---|---|
| `index.html` | The entire experience (WebGL scene, shaders, UI, interaction) |
| `boat-grid.js` | The pixel-boat shape as a 0/1 grid |
| `Logo-blue.png` | Source logo the boat grid was traced from |
| `ANIMATION_HANDOFF.md` | Deeper architecture notes and tuning knobs |

## Features

- Blueprint-grid sea with real planar reflections, mip-filtered ripples, and a filmic (ACES) grade
- Holographic topographic islands sized per protocol
- Cinematic camera intro, idle drift, sonar-ring selection with dolly-in
- Instrument HUD (agents / bounded value / voyages) and a live activity ticker
- Optional ambient sound (toggle, off by default)

Built and iterated with Claude Code.
