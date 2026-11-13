# 01 Design Foundations & Theme Tokens

`src/new.css` is the style entry point (`@import styles/*` in a fixed order, `dark.css` last); the tokens themselves live in `src/styles/theme.css`. This document describes only the tokens and rules currently in use.

## 1. Theme structure

The theme is driven by `document.documentElement.dataset.theme`, with values `light` | `dark`. The corresponding selectors:

- `:root[data-theme="light"]`
- `:root[data-theme="dark"]`

Light and dark share the same component class names; only the CSS variables switch. Every color variable is registered with `@property` as an interpolable custom property, and a theme switch adds the `.theme-transition` class for a 0.35s page-wide color transition (removed afterwards, so everyday rendering never carries a global transition).

## 2. Core color tokens (styles/theme.css)

### Light

| Token | Value | Purpose |
|---|---|---|
| `--frame` | `#f0f3f6` | window base, TopBar |
| `--content` | `#f7f9fc` | main content area |
| `--card` | `#f7f9fc` | sidebar, cards |
| `--surface-inset` | `rgb(52 63 81 / 2%)` | drag placeholder-slot inset background |
| `--text-primary` | `#3d4552` | primary text |
| `--text-secondary` | `#3d4552` | secondary text |
| `--text-weak` | `#69707c` | weak text |
| `--border` | `#d9dee7` | regular borders |
| `--border-strong` | `#b3bbc8` | strong borders |
| `--curve-axis` | `#b3bbc8` | curve axis solid lines |
| `--curve-path` | `#009aa2` | curve body |
| `--hover` | `#e3e8ef` | hover background |
| `--track` | `#e3e7ee` | tracks / slots |
| `--theme-hover-bg` | `#c2cbd8` | theme three-state hover background |
| `--brand` | `#47c3d1` | brand cyan |
| `--brand-deep` | `#009aa2` | deep brand cyan |
| `--brand-soft` | `rgba(71,195,209,0.1)` | soft brand background |
| `--brand-bg` | `#e2f5f7` | view-switch thumb background |
| `--danger` | `#dc3d43` | danger / delete |
| `--danger-soft` | `rgba(220,61,67,0.1)` | soft danger background |
| `--on-accent` | `#ffffff` | text on dark/accent backgrounds |
| `--accent-bg` | `#3b3c3f` | active pill background, Toast background |
| `--thumb` | `#ffffff` | slider thumb |
| `--gs-thumb-border` | `#3d4552` | slider thumb border |
| `--gs-thumb-disabled` | `#ffffff` | disabled slider thumb (white in light mode) |
| `--ring-hover` | `rgba(15,23,42,0.14)` | input focus ring |
| `--scrollbar` | `#d9dce4` | scrollbar thumb |
| `--scrollbar-hover` | `#c2c7d1` | scrollbar hover |
| `--disabled-fill` | `#b8bec9` | disabled fill |
| `--tab-active-bg` | `#e9edf1` | active tab background |

### Dark

| Token | Value | Purpose |
|---|---|---|
| `--frame` | `#16181b` | window base, TopBar |
| `--content` | `#1d1f22` | main content area |
| `--card` | `#1d1f22` | sidebar, cards |
| `--surface-inset` | `rgba(147,197,253,0.02)` | drag placeholder-slot inset background |
| `--text-primary` | `#c1c6cd` | primary text |
| `--text-secondary` | `#969ca4` | secondary text |
| `--text-weak` | `#717882` | weak text |
| `--border` | `#373a3e` | regular borders |
| `--border-strong` | `#51555b` | strong borders |
| `--curve-axis` | `#7c7d7f` | curve axis solid lines |
| `--curve-path` | `#47c3d1` | curve body |
| `--hover` | `#25292e` | hover background |
| `--track` | `#22262a` | tracks / slots |
| `--theme-hover-bg` | `#363c45` | theme three-state hover background |
| `--brand` | `#00919e` | dark-mode base brand color |
| `--brand-deep` | `#006265` | dark-mode brand hover / deeper shade |
| `--brand-soft` | `rgba(71,195,209,0.14)` | soft brand background |
| `--brand-bg` | `#1d3e44` | view-switch thumb background |
| `--danger` | `#ea5556` | danger / delete |
| `--danger-soft` | `rgba(234,85,86,0.15)` | soft danger background |
| `--on-accent` | `#c1c6cd` | text on dark active backgrounds |
| `--accent-bg` | `#43474e` | active pill background, Toast background |
| `--shadow-ink` | `15, 18, 23` | shadow cool-grey near-black (dark mode) |
| `--thumb` | `#c5c9d0` | slider thumb |
| `--gs-thumb-border` | `#1d1f22` | slider thumb border (matches the card background) |
| `--gs-thumb-disabled` | `#6d7279` | disabled slider thumb (lighter than borders, darker than enabled) |
| `--ring-hover` | `rgba(255,255,255,0.14)` | input focus ring |
| `--scrollbar` | `#3f4348` | scrollbar thumb |
| `--scrollbar-hover` | `#4f5358` | scrollbar hover |
| `--disabled-fill` | `#32363b` | disabled fill |
| `--tab-inactive` | `#2f3135` | tab hover background |
| `--tab-active` | `#45474c` | active tab background |
| `--tab-active-bg` | `#45474c` | active tab background (used by `tab-btn`) |

### Brand color rules

- Light: `--brand` is used for pill backgrounds / borders / themed text; `--brand-deep` for text and hover rings that need deeper contrast.
- Dark: `--brand` is the base brand color; `--brand-deep` covers the "deeper" cases such as on-state hover rings.
- The channel pills `.ch-pill.active`, the install pill `.install-btn`, the save-preset pill `.sel-action.save`, and the device-properties button `.dev-prop-btn:hover` all uniformly use `--brand` for border and text in dark mode.

## 3. Shadows

### Light

```css
--card-shadow: 0 1px 2px rgba(15, 23, 42, 0.02), 0 1px 3px rgba(15, 23, 42, 0.04);
--hover-shadow: 0 2px 4px rgba(15, 23, 42, 0.06), 0 4px 10px rgba(15, 23, 42, 0.08);
```

### Dark

Dark shadows avoid pure black and use the cool-grey near-black `--shadow-ink: 15,18,23`, so shadows never read as a glow:

```css
--card-shadow: 0 1px 3px rgba(var(--shadow-ink), 0.7), 0 4px 14px rgba(var(--shadow-ink), 0.4);
--hover-shadow: 0 4px 10px rgba(var(--shadow-ink), 0.8), 0 6px 16px rgba(var(--shadow-ink), 0.55);
```

## 4. Radius & spacing

### Radius

| Variable / class | Value |
|---|---|
| Cards, curve card, device properties card | `16px` |
| Sidebar, main content area | `26px` |
| Dialogs | `20px` |
| Pill buttons, tags, inputs | `9999px` / `8px` / `10px` |
| Tab buttons | `16px` |

### Spacing

- App shell padding left/right/bottom: `10px`
- Between sidebar and main content: `10px` (`--side-gap`)
- Card grid gap: `12px`
- Form row spacing: `8px`

## 5. Typography

- Font family: `"Plus Jakarta Sans", "Microsoft YaHei UI", system-ui, sans-serif`
- Numbers use `font-variant-numeric: tabular-nums`, so parameter inputs and slider readouts never jitter.
- Sizes: primary text 13px, card titles 13–16px, weak text 11–12px; preset pill group subtitle 12px.

## 6. Z-index hierarchy

| Layer | z-index | Content |
|---|---|---|
| Content | 1 | cards |
| Tab close / selected | 3–5 | elements inside the tab bar |
| Sidebar resizer | 20 | drag divider |
| Curve hover tip | 20 | frequency-response hover card |
| Bottom row | 20 | device properties + curve card |
| Selection toolbar | 30 | floating toolbar |
| Marquee rectangle | 40 | selection rectangle |
| Scrollbar track (outside dialogs) | 50 | content page / sidebar overlay scrollbars |
| Dialog overlay | 60 | Dialog overlay |
| Dialog content | 61 | Dialog content |
| Scrollbar track (inside dialogs) | 65 | install / save preset / import preview / uninstall progress |
| Toast | 70 | lightweight toast |
| Drag flight copy | 70 | `drag-fly.fly-anim`; portaled into the scroll content layer, landing inside `.device-body`'s stacking context (that context has `z-index:0` and sits entirely beneath the tab bar's 40); 70 only needs to beat the cards (1), the marquee box (40), and the tint layers (25–36) |
| Drag overlay | 75 | `drag-fly.overlay-fixed`; portaled into `.device-body` (not the scroll container), clipped to the content-area bounds by its `clip-path`; `position:fixed` viewport positioning that does not shift with scrolling (required for cursor lock) |
| Window resize handles | 9999 | resize handles |

## 7. Color mapping tables (TS)

The group/preset colors' light hover, dark base, and dark hover are precomputed in `src/lib/blocks.ts` + `src/lib/accent.ts`:

- `ACCENT_HOVER_HEX`: light hover colors
- `ACCENT_DARK_HEX`: dark base colors and dark hover colors

Computation:

1. Convert the base color to OKLCH.
2. Light hover: `L * 0.75` with `C` unchanged; if the RGB-perceived green share is high, nudge `H` slightly toward the OKLCH green base hue `142.5°`.
3. Dark base: `L = 0.6`, `C`/`H` unchanged.
4. Dark hover: run the light-hover algorithm once more on top of the dark base.
5. OKLCH→sRGB uses gamut mapping: keep L/H and binary-search `C` down into the sRGB gamut, so oranges never drift red.

## 8. Scrollbar tokens

- Custom overlay scrollbar: transparent track, solid thumb `--scrollbar` (5px wide, 9999px radius), hover `--scrollbar-hover`; fade in/out via `opacity 0.25s ease`, auto fade-out 1.2s after scrolling stops.
- Native scrollbar (fallback, `fluentOverlay` style): thumb also uses `--scrollbar` / `--scrollbar-hover`, hover transition `0.32s cubic-bezier(0.4,0,0.2,1)`.

## 9. Glass ring (edge highlight) tokens (since 2026-09)

The glass ring is the lit outline around the glass edge of a card/panel, built from three layers: an outer `conic-gradient` directional highlight, a bottom canvas shade layer (`.vx-edge-shade`), and the card's inner light "leaving a gap" where the shade sits.

| Token | Meaning | Light | Dark |
|---|---|---|---|
| `--ring-g1` / `--ring-g2` | diagonal highlight strength (top-right / bottom-right) | `0.42` / `0.24` | `0.16` / `0.09` |
| `--ring-s1` / `--ring-s2` | side highlight strength | `0.3` / `0.28` (`@property` initial) | `0.12` / `0.28` (`@property` initial) |
| `--ring-l` / `--ring-r` | left / right edge brightness | `0.08` / `0.04` | `0.035` / `0.018` |
| `--ring-t` / `--ring-b` | top / bottom edge brightness | `0.32` / `0.14` | `0.115` / `0.05` |
| `--ring-m` | bottom shade strength | `0.22` (`@property` initial) | `0.22` (`@property` initial) |
| `--ring-base` | base outline color | `rgba(217,222,231,0.25)` | `rgba(255,255,255,0.035)` |
| `--edge-shade` | canvas shade layer color | `rgba(126,136,152,0.9)` | `rgba(8,10,13,0.9)` |

### 9.1 Geometry injection (`useGlassRing.ts`)

- `applyRingAngles()` measures width/height with `getBoundingClientRect()`, computes the four corner angles inside the `conic-gradient` via `atan2`, and injects `--ang-tl` / `--ang-tr` / `--ang-br` / `--ang-bl`, so the ring's lit falloff actually lands on the four corners.
- The top highlight line's end fade is converted from pixels to angles and written to `--ring-fade`: default `13px`, curve card `30px`; the corner radius is taken as `16px` (`fx-toolbar` uses `14px`), so wide and narrow cards read with a consistent visual length.
- The bottom converges independently along the bottom-left corner tangent, avoiding the inconsistent feel across card widths that reusing the top-right corner caused.

### 9.2 Theme interpolation and the shade layer

- `.theme-transition` interpolates all numeric `--ring-*` tokens and `--edge-shade` over `0.35s cubic-bezier(0.4,0,0.2,1)`, so ring colors transition smoothly on a theme switch.
- The canvas shade layer `.vx-edge-shade`: on a theme switch it is switched off with `0.15s` first and fades back over `0.6s` once the transition ends.
- The shade no longer draws its own black line: `shade` controls where the inner light "leaves a gap" (`lit=0`), letting the base highlight style's own shade show through — where the inner light is weak, the shade is naturally weaker.

### 9.3 Edge-tint sampling layer (`useEdgeTintLayer.ts`)

- The device card / curve card ring tint is painted on a low layer (`z-index: 25`); the floating toolbar's own ring is painted on a high layer (`z-index: 35`).
- Color is sampled only from the parts of a card that actually carry theme color (borders / chips / dots); black-and-white areas contribute no hue; the curve SVG path participates as discrete point lights in the higher-layer toolbar tint.
- Light sources are bucketed on a 64px grid; ring events are throttled with dirty-rect redraws; idle periods run no standing repaint loop and no fixed full polling (event-driven refresh).
