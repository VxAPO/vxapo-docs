# 05 Dialogs & Selection Toolbar

## 1. Dialog basics

Radix Dialog. Base classes:

- `.vx-dialog-overlay`: `position:fixed; inset:0; background:rgba(10,13,18,0.45); z-index:60`, **plus a layer of `backdrop-filter: blur(2px)`** (with the `-webkit-` prefix): the curve and cards behind collapse into color blocks and the eye lands on the dialog. 2px only seeks "defocus" — do not increase it; larger values smear the UI behind into a blur.
  **Enter and exit are CSS animations** (`[data-state="open"]` / `[data-state="closed"]`, durations in 06's motion table): these two animations are Radix's only anchor for the exit; switch to a `transition`, or give the closed state no animation name, and the element unmounts immediately — the exit is never seen.
- `.vx-dialog-content`: centered, `width:min(420px, calc(100vw - 48px))`, background `var(--card)`, border `1px solid var(--border)`, radius 20px, shadow `var(--modal-shadow)`, four layers: `inset 0 0 0 1px var(--dsw-alias-border-inverted)`, `0 0 1px 0 rgba(0,0,0,.2)`, `0 0 4px 0 rgba(0,0,0,.02)`, `0 8px 20px 0 rgba(0,0,0,.08)`. The design file wrote `inset 1px solid …` — `solid` is border syntax and does not exist in box-shadow; the token is currently `transparent`, so that inset ring is invisible; it is kept to stay aligned with the design token (changing the token would reveal it).
- **Why the longest layer is 20px instead of the design's 32px**: this family of low alphas gets quantized into equal-value bands under 8-bit output. Measured (flat base color, 1× scale, sampling pixels straight down from the bottom edge): at `0 12px 32px 0.08` the whole shadow spans only 9 levels with 4–5px plateaus — visibly "banded"; at `0 8px 20px 0.08` the **depth is unchanged** (still 125→134, same weight) but the spread shortens from 24px to 16px, the plateaus tighten to 2–3px, and the banding is basically invisible.
  **Do not** fix the banding by raising the alphas — that weighs the whole shadow down. This is a deliberate deviation from the design file (a design tool's canvas renders at higher precision and does not show the quantization). Also, the dim layer's `blur(2px)` flattens the background texture, removing the "camouflage" that would mask the bands — so the deviation is necessary.
  Both themes' values live in `theme.css`'s `:root` (dark swaps in `--shadow-ink` and raises the alphas — light's 0.08 black is invisible on a dark backdrop), and `dark.css` no longer carries its own geometry, avoiding two independently written copies.
  **The transparent variant inside `theme-transition` must be layer-for-layer isomorphic** (same count / offsets / blur, only the colors transparent), otherwise `box-shadow` cannot interpolate and the shadow snaps instead of fading on theme switches.
- Header `.vx-dialog-head`, title `.vx-dialog-title`, close `.vx-dialog-close`, body `.vx-dialog-body`.

## 2. Settings dialog

- Row `.vx-setting-row`: name + description + value control.
- Theme setting, three states: Light / Dark / Follow system (`.theme-seg`, 240px wide, thumb slides smoothly).
- Language setting: 简体中文 / English.

## 3. Install device dialog

- `.install-dialog` / `.install-body` / `.install-list` (`.os-scroll` + OverlayScrollbar).
- Each item `.install-item`: device name, format info, install button.
- Install button `.install-btn`: rounded pill, background `var(--brand-soft)`, text `var(--brand-deep)` (light) / `var(--brand)` (dark), border `1px solid rgba(71,195,209,0.45)` (dark: `var(--brand)`); transition `background-color / color / border-color 0.18s ease`.
- Hover: background `rgba(71,195,209,0.2)`.
- Disabled: `opacity:0.6; cursor:wait`.

### 3.1 Install verification progress panel (the `install --verify` loop)

- Clicking install switches to the progress view `.install-progress` (one device installs at a time).
- State machine: `installing → restarting ↔ verifying → done | failed`; retries are transparent to the user (restarting/verifying advance with the event stream; retry only changes the copy).
- Data source: the App backend runs the CLI `install --verify` asynchronously via `install_device`, parses the JSON events line by line and `emit("install-progress")`es them; the frontend subscribes via `onInstallProgress`.
- Display:
  - `.install-progress-head`: device name + spinning spinner (while running).
  - `.install-progress-track` / `.install-progress-bar`: transparent track, only the bar shows; installing/verifying pulse, restarting holds a fixed 45%, done/failed 100%.
  - `.install-progress-text`: stage copy (write config / stop service / start service / verify / retry / success / failure).
- Terminal states: done → a "Done" button closes and refreshes the device list; failed → "Retry / Done", where retry reruns `handleInstall` and Done closes; both terminal states trigger a device-list refresh (on failure the best config is already written, so the device appears as installed; `rollback_install` backs up a rollback).
- Return contract: `InstallResult { success, mode, score, attempts, best_mode, best_score }`.

## 4. Uninstall dialog

- Dangerous-operation confirmation + progress list (`.op-progress`, `.os-scroll` + OverlayScrollbar).
- Buttons: Cancel + Confirm uninstall (danger color).

## 5. Save custom preset dialog

- Class `.vx-dialog-wide`: `width:min(520px, calc(100vw - 48px))`.
- Field `.vx-field`: label + input.
  - Name `.vx-text-input`: defaults to the name derived from the current marquee selection; may be empty (falls back to a placeholder name on save).
  - Color `.preset-swatches`: fixed 24-color palette (`SWATCHES`).
    - Swatch `.preset-swatch`: 24×24px disc, border 2px transparent.
    - Selected `.preset-swatch.active`: **the ring is drawn outside the disc** — `--swatch-ring: var(--swatch-hover, var(--text-primary))` + `box-shadow: 0 0 0 2px var(--swatch-ring)`. **Do not** make the ring by lighting up `border-color`: the swatch is a `<button>`, which the UA defaults to `box-sizing: border-box`, so that 2px border draws **inside** the 24px box and, once opaque, eats a ring of the background beneath it — the disc shrinks from 24 to 20px, reading as "the circle gets smaller when selected" (hit this). With the outer ring the disc keeps its size (measured: 22px in both states). The old `0 0 0 2px var(--card)` spacer ring was removed with it: it matched the dialog background and never separated anything.
    - **In dark mode this ring goes one step deeper** (override in `dark.css`, swapping only `--swatch-ring`'s color; the geometry stays in `cards.css`): `--swatch-hover` is an L 0.45 darkened variant — clear against light backgrounds, but in dark mode it is **not different enough** from the fill (L 0.6) and the ring does not read. There it is mixed with 32% pure black (≈L 0.35), so selection reads from the **light/dark difference** of "a bright disc with a bite taken out of it", leaving the fill untouched and the light-mode rule untouched. The reverse approach (a light ring in dark mode) was tried and discarded: it inverts the "deepen on light, lighten on dark" instinct and makes the whole palette look scattered.
    - Each swatch's `--swatch-hover` is computed in TS by `accentHoverColor(c)` and injected.
  - Per-band descriptions `.preset-band-list` (`.os-scroll` + OverlayScrollbar): the section title uses the generic `t("filters")` ("滤波器" / "Filters"); each row `.preset-band-row` shows the band tag (`fc · gain`) + an input whose placeholder is `preset.bandDesc` ("语义描述" / "Semantic description").
    **Defaults are pre-filled with the semantic view's default mapping** (`perceptualName(b)` = the perceptual label for the band's start frequency, rendered through `displayBandName` in the current language) — i.e. "what the semantic view shows for this card while unnamed", visible and editable the moment the dialog opens. Rows left empty store no `name`.
    - **Empty rows still resolve**: an empty row has no `name` in the config; the semantic view falls back via `semanticName` to the perceptual label derived from `fc` (the same mapping as `perceptualName`), then `displayBandName` renders it in the current language — so empty rows follow language switches too and never retain a stale language. `applyPreset` **must not** fall back to the preset name for a band name (it once did, via `b.name ?? nameBase`): the band card would then show the preset name — neither the default mapping nor translatable — and the empty row would be stuck in the language of the moment it was applied.
    - On save, decided per row (`customBandNameFields`): **untouched defaults are written pairwise as `name` + `name_en`** (so both `zh`/`en` resolve, and a language switch shows the right one); **user-typed text is kept verbatim with no added translation** — a language switch never tampers with user input.
    - The "default mapping" list derives only from `PERCEPTUAL_LABELS` (derived from `blocks.ts`'s `PERCEPTUAL_RANGES`), and the translations derive only from the i18n entries (the keys are the Chinese literals) — **never a second hand-copied label table**.
    - `padding: 10px 0`: **no horizontal allowance** — focus is conveyed by border color alone, with no outer halo, so there is nothing to clip. A caution: this is a scroll container, and `overflow-y:auto` also clips **horizontally at the padding box**, while the input is `flex:1` with its right edge exactly on that boundary — **any outer focus ring added back would lose its right segment** (the left edge has the band tag in front of it, so only the right side shows it). Adding an outer ring requires an equal inset as well.
    - Band tag `.preset-band-tag`: `width:122px`, `font-size:12px` (same level as `.vx-field-label` and the input; the old 11px/110px was both small and left the column airy). The column width leaves margin for the longest tag, `16.00 kHz · +10.5 dB`, at that size; `tabular-nums` aligns the digits.
- Input focus `.vx-text-input:focus-visible`: **only changes the border color**, `border-color: var(--brand)`, **no outer halo** — the same set as `.vx-select:focus-visible`. Two halo variants were tried (a `0 0 0 3px var(--brand-soft)` equal-width hard ring, and a 1px thin ring + 6px blurred halo); both read worse than the border itself, and any outer ring gets clipped by that scroll container anyway.
- Save action: `onSave(name, desc, color, descriptions)`; the intro field has been removed.

## 6. Generic confirm dialog

- `.vx-confirm-text` shows the confirmation copy.
- Used for irreversible operations: deleting a custom preset, closing the channel selector, etc.

## 7. Selection toolbar `.sel-toolbar`

- Appears when marquee-selected cards exist.
- Structure: outer `.fx-toolbar` (`position:absolute; left/top:0; translate:-50% 0; z-index:30`) › glass panel `.sel-toolbar` › content (`.sel-toolbar-label` + `.sel-toolbar-actions`). The float's position is owned exclusively by `useMarqueeSelection`'s follow loop writing `transform`; `translate` and the entrance lift belong to CSS.
- **The toolbar's size participates in the 8px inset clamp** (the same `TOOLBAR_EDGE = 8` as the cards): horizontally `clamp(cx, 8 + w/2, bodyW − 8 − w/2)`, vertically likewise. So the size must be the **true value**: beyond the `ResizeObserver`, measure once synchronously with `getBoundingClientRect()` on attach — an RO callback waits for the next layout; relying on it alone places the first toolbar from the fallback size (`280×64`), and the channel selector ("copy to channel") widens it far past that, so the right edge computes too far right, the float overshoots the inset line, and only the next frame pulls it back. The RO must also **watch the element itself**, not key off something like `selectedIds.length > 0`: the toolbar mounts only once `selGeom` exists (`selGeom` is the product of a passive effect, one commit later than that layout effect), so a boolean-dep effect runs while the element is still absent, returns early on a null `el`, and never reruns — the size stays at the fallback forever. Symptom: the **first** marquee after a refresh overflows, while after canceling once the next one is fine (during the exit animation the old element is still mounted, which is when the RO finally attaches). Hit this.
- Glass (light): the base color belongs to `.sel-toolbar::before` (`--glass-bg` = `color-mix(in srgb, color-mix(in srgb, var(--card) 80%, var(--border)) 50%, transparent)`), `backdrop-filter: blur(calc(var(--glass-blur) * var(--glass-t))) saturate(1.1) contrast(0.55) brightness(1.38)` (`--glass-blur` is `12px` light / `8px` dark; dark also overrides `--glass-bg` / `--glass-shadow`); the `--glass-shadow` drop fades with the base; the border is transparent — the outline is carried by the outer glass ring.
- **The fade in/out runs through `--glass-t` (0→1), not opacity**: a panel's `backdrop-filter` is downgraded wholesale by the browser when an ancestor's `opacity < 1` — the blur would only appear once the fade ends. `--glass-t` is registered as a `<number>` so transition can interpolate it: `.is-in` is added the frame after mount, easing over `180ms ease-out`; blur radius, the `::before` base, and content opacity are all driven by it. Content must be `position:relative; z-index:1` to sit above the base (the pseudo-element is positioned and would otherwise cover the text).
- The highlight band follows too: `.fx-toolbar::after` (top/bottom/side conic highlights) `opacity = var(--ring-op) * var(--glass-t)` (`--ring-op` light `0.4` / dark `0.8`); `.fx-toolbar::before` (the inner blur-bleed layer) `opacity: var(--glass-t)`.
- Exit: `usePresence()` waits `180ms` before `safeToRemove()` unmounts — the tint canvas is an independent layer and would vanish hard the moment the DOM disappears; the painting side scales opacity by `--glass-t`, and `driveFor()` explicitly drives per-frame repaints (DOM class changes produce no MutationObserver callbacks; without a driver it lags the DOM).
- **The filter must be zeroed as a whole**: beyond `blur`, `saturate`/`contrast`/`brightness` must also multiply `--glass-t` back to neutral — scaling only the blur leaves light mode's `contrast(0.55) + brightness(1.38)` active, and mid-fade a whitish base leaks through.

### 7.1 Action buttons

- `.sel-action.save`: save as custom preset. Background `var(--brand-soft)`, text `var(--brand-deep)` (light) / `var(--brand)` (dark), border `1px solid rgba(71,195,209,0.45)` (dark: `var(--brand)`).
- `.sel-action.delete`: delete. Background `var(--danger-soft)`, text `var(--danger)`, border `color-mix(in srgb, var(--danger) 45%, transparent)`.
- `.sel-copy-btn`: copy to channel. **Low-contrast background**: light border/text `#3b3c3f`, background `var(--surface-inset)`; dark border `var(--border-strong)`, text `var(--text-primary)`; hover background `var(--hover)`. 32px tall, padding `0 14px`, radius 9999px.
- `.sel-copy-menu`: copy-to-channel menu. **Must portal to `document.body`** — `position: fixed`, `left`/`top`/`min-width` written by JS from the trigger button's viewport rect, `z-index: 70` (same tier as `.vx-select-content`). Why: it used to be absolutely positioned inside `.fx-toolbar`, but `.fx-toolbar`'s `z-index: 30` is **below** the tint canvases (`Z_TOOL 35` / `Z_TOOL_SHADE 36`) and the `.fx-toolbar::after` highlight ring; `.sel-toolbar` also carries `backdrop-filter` and is its own stacking context, so a menu inside the panel **cannot rise above those layers no matter how large its z-index**, and the `mix-blend-mode: screen` tint covered it whole — it read as "a menu with a transparent background" (hit this). Two more points: the position must track the button via rAF (the toolbar is moved by the follow loop writing `transform`); and the outside-pointerdown close check must count `.sel-copy-menu` as "inside", otherwise clicking a menu item unmounts the menu before its `click` can dispatch (copying silently fails). Background `var(--card)`, border `1px solid var(--border)`, radius 12px.

## 8. Toast `.vx-toast`

- Fixed bottom-center: `position:fixed; left:50%; bottom:32px; transform:translateX(-50%); z-index:70`.
- Background `var(--accent-bg)`, text `var(--on-accent)`, radius 9999px, padding `8px 16px`, font-size 12px.
- Animation `vx-toast-in 0.2s ease`.

## 9. Custom overlay scrollbar (OverlayScrollbar)

Scroll containers uniformly get `.os-scroll` to hide the native scrollbar; `OverlayScrollbar` portals a `position:fixed` track that takes no layout width (see the *App Reference Specification* and `overlay-scroll.css`):

- Activation: responds only to **genuine user scrolling** (wheel / touch / thumb drag); programmatic scrolling (view-switch scroll restoration, height collapse) updates geometry only, without activating.
- Fade-out: auto-fades 1.2s after scrolling stops (`opacity 0.25s ease`); when content cannot scroll, the thumb geometry is kept while the active state smoothly fades; a scrollbar mid-fade is not "re-entered" by a view switch.
- Device switch: on `deviceKey` change it fades out first; the component itself survives across devices and re-appears per scrolling once the new device is mounted.
- Z-index: outside dialogs `z-index:50` (below the overlay's 60, correctly dimmed under it); inside dialogs `z-index:65` (above content's 61, stays visible).
- Integration points:
  - Content page `.tuning-scroll`: `right:-2`, bottom avoidance 26 (only the bottom corner).
  - Sidebar `.sidebar-scroll-inner`: `rounded` (avoiding 24/26 top/bottom).
  - Install list `.install-list`, save preset `.preset-band-list`: rounded thumb, `right:-6`, `z-index:65`.
  - Import preview `.import-preview-text`: bottom avoidance 14, `z-index:65`.
  - Uninstall progress `.op-progress`: `z-index:65`.
