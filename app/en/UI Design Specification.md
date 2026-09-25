# VxAPO App UI Design Specification (Implemented)

> This directory describes the actual UI structure and interaction logic of `vxapo-app`. It is based on the code: class names, CSS variables, and animation durations match `src/new.css`, `src/styles/*`, and `src/components/*`.
> Revision date: 2026-08-23 (styles moved from single-file `new.css` into partitioned `src/styles/`, with `new.css` acting as the `@import` entry).

## File map (Chinese originals in `app/zh/UI 设计规范/`)

| File | Content |
|------|---------|
| `01-design-foundations.md` | theme variables, colors, typography, radius, spacing, shadows, z-index |
| `02-shell-and-navigation.md` | window skeleton, TopBar, view switching, sidebar, device tabs |
| `03-views-and-cards.md` | semantic/parameter views, group cards, filter cards, effect cards, channel pills |
| `04-curve-and-device-panel.md` | frequency response curve, floating panel, device properties card, bottom area |
| `05-dialogs-and-selection.md` | dialogs, install/uninstall/save preset, selection toolbar, Toast, custom scrollbar |
| `06-drag-and-motion.md` | drag engine, marquee selection, animation curves, subpixel snapping |

## Design principles

1. Light/dark share the same structure; only CSS variables change. Color variables are registered with `@property` and interpolated during a 0.35 s theme transition.
2. All positioning and animation snap to device pixels via `snapPx()`.
3. Interaction stability comes before motion; animations must not change interaction results and can be interrupted.
4. Hover, drag, and marquee are pointer-level interactions; desktop-first, mouse/trackpad friendly.
5. Scrollbars are custom-drawn: they take no layout width, appear only on genuine user scrolling, auto-fade after 1.2 s idle, and are not re-triggered by programmatic scrolling (view switch / height collapse).

## 01 Design foundations

- Theme tokens: background, surface, text, accent, border, shadow, hover states for light and dark, plus curve/scrollbar/slider tokens (`--curve-path`, `--curve-axis`, `--scrollbar`, `--thumb`, `--gs-thumb-*`).
- Dark axis color is `#7c7d7f`; dark scrollbar is `#3f4348` / hover `#4f5358`.
- Z-index hierarchy: content 1, bottom row/toolbar 20, selection toolbar 30, marquee 40, scrollbar (non-dialog) 50, dialog overlay 60, dialog content 61, scrollbar (in dialogs) 65, Toast 70, drag fly 75.

## 02 Shell and navigation

- Window skeleton with custom title bar (window controls); TopBar: logo, settings/import/export, semantic/parameter view switch, window controls.
- View switch `.view-seg`: centered segment with sliding thumb (`--brand-bg` + `--brand` border), 0.25 s slide; disabled rules per view.
- Sidebar: segmented switcher (Preset | Custom | Advanced); preset/custom entries are compact pills with an 8 px accent dot, bold group name, and 12 px weak subtitle (`flex + gap` layout); advanced section uses category + pill rows.
- Device tabs: `tab-btn` (13 px, 16 px radius), tuning dot switch (`.tab-dot`), close button, add button; per-device tuning state initialized from disk.
- Empty state: centered logo + install row when no devices; the plus and install button use 0.18 s hover transitions, and the tip color follows theme variables (no lag on theme switches).

## 03 Views and cards

- Semantic view (`PresetView`) and parameter view (`AdvancedView`) share `.view-stage`; switch animation is a 0.32 s x-slide. Both views stay in the DOM (no `AnimatePresence` mount/unmount): the outgoing view gets `.is-exiting` (absolute positioning) for the slide, then is hidden with `display:none` — this avoids rebuilding the 31 parameter-card subtrees. A height collapse with scroll restoration follows.
- Card grid: 2–5 columns by width (900/1200/1700 px breakpoints), 12 px gap; group cards span all columns.
- Group cards (`.group-card` / `.sem-group`): accent edge, enable dot, group name (editable via `.sem-select`), chips; filter cards (`.band-card`): enable dot, type, params (slider + gain input); effect cards (`.effect-card`): dot, name, description, param rows, semantic strength row.
- Built-in effects: preamp, wide (Stereo Field), aural, reverb (Plate), compressor, loudness (Loudness EQ); semantic strength maps to core params (see App Reference Specification §6).

## 04 Curve and device panel

- Frequency response curve with `CurveGrid` (dashed Y/X grid lines; axis in `--curve-axis`), `CurvePlot` (path in `--curve-path`); coordinates via `lib/curve.ts` (`logX` / `dbY`).
- Evaluation points: 20 Hz–20 kHz log sweep (481 points) + band centers + high-Q refinement + neighbor midpoints.
- Performance: `useThrottledCompute` recomputes at 42 ms (≈24 fps) while dragging; RBJ coefficients cached.
- Device properties card (GUID/version/mode/slots/EAPO), normalization button; glassmorphism on both bottom cards.
- Curve hover tip: exponential follow (8% per frame), 280 ms side-flip, horizontal/vertical avoidance.

## 05 Dialogs and selection

- Radix Dialog based overlays (`vx-dialog-overlay` z-60 / content z-61); settings (theme three-state + language), install (with `--verify` progress loop and `InstallResult`), uninstall, save preset (name + color swatches + per-band descriptions, empty by default; no intro field), import (TOML preview), confirm.
- Selection toolbar with save/delete/copy-to-channel actions; copy button uses a low-contrast background.
- Custom overlay scrollbar (`OverlayScrollbar`): no layout width; appears on wheel/touch/thumb drag only; fades out after 1.2 s idle (0.25 s opacity); survives device switches; z-index 50 normally, 65 inside dialogs.
- Toast: fixed bottom-center, `--accent-bg`, 0.2 s entrance.

## 06 Drag and motion

- Custom pointer-level drag engine (`useDragSort`): 500 ms enter debounce (must exceed `LAYOUT_ANIM_MS` + `ANIM_SETTLE_BUFFER_MS` = 480 ms), 400/320 ms layout animations on `LAYOUT_EASE`, 80 ms settle buffer booked against the duration actually used (an outside-slot move waits 320 ms, not 400 ms), 48 px outside distance, and a 530 ms fly landing (`FLY_ANIM_MS` 400 + `FLY_HOLD_MS` 100 + `FLY_TAIL_MS` 30; the position segment is `FLY_MOVE_RATIO` 0.72 = 288 ms, and the framer `x`/`y` `times` and the shadow-tail `times` are derived from that same ratio). Pointer moves and scroll events are coalesced into one `requestAnimationFrame` settlement per frame (shift slots, then position + hit-test), because high-rate pointers deliver hundreds of events per second and handling each one would rewrite styles and re-measure rects inside a single frame; the release flushes any pending frame first, so the flight starts where the pointer actually stopped. The grabbed overlay follows the pointer with `transform: translate()` (`left`/`top` stay 0 as the static basis — writing `left`/`top` every frame is a layout write — and no `translate3d`/`will-change` is used on purpose, so the card is not promoted to a composited layer that would change text rasterisation while the pointer rests). Slots shift with the page scroll during a drag (the hit test re-runs from the last pointer position), the landing point is re-measured after the synchronous reorder, and the flight animates `x`/`y` (composited, snapPx-aligned) with a static size. The flight's basis is the landing point itself, with path offsets measured from it and a final frame of 0, so a cached transform rewrite cannot shift the landing; shortly after the motion ends the flight drops `transform`/`will-change` and returns to normal painting (a layer that stays composited renders text at a different raster/antialiasing quality). The release settles on the already-confirmed slot instead of re-testing the pointer (so the placeholder never jumps at release and the landing matches what the user saw), and `finalizeDrop` is deferred out of the `pointerup` handler so the input pipeline is not blocked. While a flight is in the air the fly tracks content scrolling: it is a fixed layer, so the landing basis is shifted by the active view stage's displacement, keeping the arc anchored to the target slot. That compensation is written synchronously inside the scroll event — it cannot be frame-coalesced, because the scroll offset participates in the same frame and a deferred write would leave the copy displaced for that frame. The avoid curve `LAYOUT_EASE` and the view-collapse curve `COLLAPSE_EASE` are the same constant (`EASE_OUT_SOFT = cubic-bezier(0.22,1,0.36,1)` in `lib/motionEase.ts`, also used by the scrollbar-thumb morph), and `lib/dragSortTypes.test.ts` pins the timing relations (debounce lower bound, the three fly segments, the position ratio, the shared curve, and the CSS landing-bar duration vs the flight).
- Marquee selection (`useMarqueeSelection`): dashed box, hit-testing limited to the current view stage during view switches.
- Animation table: view switch 0.32 s + 800 ms collapse, device page fade 180 ms out + 180 ms in (`DEVICE_FADE_MS`, `AnimatePresence mode="wait"` with `key=<device>`; the exiting element keeps the previous device's props, which is why page data flows down as props rather than store subscriptions), selection-toolbar glass 180 ms via `--glass-t` (the exit waits the same 180 ms before unmount so the tint canvas can fade), tab 0.15 s, card hover 0.18 s, dialog 0.18 s, toast 0.2 s, drag 320–530 ms (landing bar 0.4 s, matching the flight), curve tip critically-damped spring (`FOLLOW_SETTLE_MS` 240 ms) + 280 ms flip, curve recompute 42 ms throttle, scrollbar 0.25 s fade + 1.2 s idle, theme transition 0.35 s.
- Subpixel snapping via `snapPx()` for all positioning and animation endpoints.
