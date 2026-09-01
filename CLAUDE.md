# Weaver — Project Standards

## Loading Screen Standard

All Mentifaber web apps use this enterprise loading screen pattern. Apply it to every new project.

### Visual Design
- Background: `#000005` (near-black)
- Title: `Orbitron` font, 11-15px, letter-spacing `.35em`, color `#00d4c0`, glow `text-shadow`
- Subtitle: `Space Mono` or `IBM Plex Mono`, ~8px, letter-spacing `.28em`, color `#1e4060`
- Phase dots: 4 dots, states: default (dark) / current (pulsing cyan border) / done (filled cyan)
- Phase labels: ASSETS · SHADERS · SCENE · ONLINE (or task-appropriate names)
- Progress bar: `width:min(260px,60vw)`, 2px height, `#000` track, gradient fill `#003a38 → #00d4c0 → #00f5e0`
- Glow dot on bar right edge, tracks the fill
- Bottom corner: project attribution e.g. `MENTIFABER · [PROJECT NAME]`

### Phase Controller
```javascript
window._lsPhase = function(step, msg, pct) {
  // step: 0=ASSETS, 1=SHADERS, 2=SCENE, 3=ONLINE
  // msg: current status label
  // pct: 0-100
};
```
Call between each external script load with incrementing percentages (5 → 18 → 34 → ... → 100).
Final call: `_lsPhase(3, 'ONLINE', 100)` then fade out with `ls.style.opacity='0'` after 400ms delay.

### Mobile Performance Rules
For any WebGL/Three.js app:
- Pixel ratio: `isMobile ? 1 : Math.min(window.devicePixelRatio, 2)` — biggest single win
- Antialias: `!isMobile` — saves GPU fill-rate
- Bloom resolution: half-size on mobile (`window.innerWidth/2`)
- Background particle count: ~35% of desktop count on mobile
- Mobile detection: `('ontouchstart' in window) || navigator.maxTouchPoints > 0 || window.innerWidth < 900`

### Preload Hints
For apps with multiple CDN scripts, add `<link rel="preload" as="script">` for every external script
in `<head>` — browser fetches all in parallel before parsing body.

### CSS Animations
```css
@keyframes ls-pulse {
  0%, 100% { box-shadow: 0 0 6px rgba(0,212,192,.3); }
  50%       { box-shadow: 0 0 14px rgba(0,212,192,.8); }
}
```

## Color Palette (Mentifaber DSC)
- `--void: #000005` — background
- `--cyan: #00d4c0` — primary accent
- `--amber: #ffaa00` — secondary accent  
- `--text: #6ea8c8` — body text
- `--text-dim: #1e4060` — muted text
- `--border: rgba(0,210,190,.14)` — default border
- `--border-hot: rgba(0,210,190,.55)` — active border

## Fonts
- Display: `Orbitron` (title, branding)
- Mono: `Space Mono` (UI, data)
- Alt mono: `IBM Plex Mono` (website pages)
