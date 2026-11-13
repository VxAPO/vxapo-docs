# 06 Drag, Marquee & Motion

## 1. Dragging cards

Every draggable card is wrapped by `DragCard`, class `.drag-card`.

- Drag handle `.drag-bar`: the card's top strip; grabbing it starts the drag.
- During the drag the pointer is captured via `setPointerCapture`, so the pointer is never lost.

### 1.1 Drag states

| State | Class | Appearance |
|---|---|---|
| hover | `.drag-card:hover` | `box-shadow: var(--hover-shadow)`; dark adds `0 0 0 1px rgba(255,255,255,0.05)` |
| dragging | `.drag-card.is-dragging` | the original spot becomes a placeholder slot: `background: var(--surface-inset); border:1px dashed var(--border-strong); box-shadow:none; > * { visibility:hidden }` |
| selected | `.drag-card.is-selected` | `outline: 2px solid var(--card-accent, var(--brand)); outline-offset:-2px` |
| selected + dragging | `.drag-card.is-dragging.is-selected` | outline removed; the dashed border takes the group color |
| grouped placeholder | `.drag-card.is-dragging:not(.is-selected).sem-group` | `border-left-color: var(--card-accent)` |

### 1.2 Drag copies

- `.drag-fly`: the flight copy; base class `position:relative; width:fit-content`.
- **The two layers mount in different places, with the same stacking and clipping discipline**:
  - **Grabbed overlay** `.drag-fly.overlay-fixed`: portaled into `.device-body` (**not** the scroll container), `position:fixed; top/left:0; z-index:75`, shadow `0 10px 28px` (dark uses `--shadow-ink`). Clipping relies on `.device-body`'s `clip-path: inset(0)` — clip-path clips the whole subtree, fixed descendants included. It remains viewport-positioned and is not inside the scroll container, so it **does not shift with content scrolling**: when the pointer stops, the card stays put in the viewport (this is exactly what cursor locking needs).
    ⚠️ Counter-example (hit this): "give the container a `transform` so `fixed` uses it as its containing block" — if that container is a scroll container, the containing block travels with the content, and the symptom is "the card offsets by exactly however much scrolled"; pointer tracking is destroyed.
  - **Flight copy** `.drag-fly.fly-anim`: portaled into the scroll container `.tuning-scroll`, `position:absolute; z-index:70`, positioned in **content coordinates** (no pointer constraint after release); scroll following is done by the browser's compositor thread (zero latency, no more jitter), and the container's overflow clips it at the content-area bounds.
- Stacking: both layers land inside `.device-body`'s stacking context (`position:relative; z-index:0`; the whole block sits beneath the device tab bar's 40, so the tab bar covers both). Within that context they only need to beat the cards (1), the marquee box (40), and the tint layers (25–36).
- **The overlay follows the pointer via `transform`**: `left`/`top` stay 0 as the static basis; JS writes `transform: translate()` each frame. Writing `left`/`top` per frame is a layout property — every frame relayouts; `transform` changes only the visual position. `translate3d`/`will-change` are deliberately unused — once promoted to a composited layer, text rasterization while the drag rests differs from a normal layer (the same reason as the flight copy's "landing handover").
- **The overlay is clamped inside the content-area rectangle** (`.content`, the big rounded rectangle), with a `DRAG_BOUNDS_INSET_PX = 8` inset — **never hugging the edge** (shadow and corners need room): positioning clamps the card's top-left to `[bounds.left + 8, bounds.right − cardW − 8] × [bounds.top + 8, bounds.bottom − cardH − 8]` (`DragSession.bounds` / `cardW` / `cardH`, measured once at grab time). The pointer may keep moving outward — **a web page cannot lock the system cursor**; what gets limited is the card: it stops at the boundary and goes no further. **Placeholder hit-testing uses the card's center** (`DragSession.curMidX/curMidY`), not the pointer — once the card is clamped, the pointer may already be outside the content area, and testing the pointer would misread "outside every slot", flinging the placeholder to the end, out of sync with what the eye sees (hit this). "Outside every slot = append to the end" still works: drag the card into the blank area below the cards (more than 48px from every slot). The flight's start rect is still read from the overlay's real box.
- **Edge-hugging drives the page scroll (edge auto-scroll)**: when the overlay hugs the content area's top/bottom edge and the container can still scroll that way, each frame advances `EDGE_SCROLL_SPEED_PX_S = 600` (px/s, converted by frame duration so 60/120Hz feel alike; long frames are clamped to 64ms, avoiding a violent jump when returning from the background). Scrolling continues while the pointer is still: `flushFrame` keeps scheduling the next frame after scrolling; it stops naturally when the direction leaves the edge or that end of the scroll range is reached. The scrolling itself reuses the existing "translate slot rects + re-hit-test"; the overlay does not shift with the scroll (it locks to the pointer).
- On grab, the shadow smoothly expands from the hover shadow to the drag shadow: `drag-shadow-lift 0.18s ease-out`.
- Landing grey bar `drag-bar-land 0.4s ease-out`: duration aligned with the arc flight (`FLY_ANIM_MS`) so it finishes before the copy unmounts (the old 0.45s exceeded the total landing time and its tail got hard-cut).

## 2. Slot detection & avoidance

`useDragSort.tsx` implements the custom pointer-level slot drag engine (constants in `lib/dragSortTypes.ts`):

- Enter-slot debounce `ENTER_DEBOUNCE_MS = 500` (must be longer than every drag animation: lower bound = `LAYOUT_ANIM_MS + ANIM_SETTLE_BUFFER_MS` = 480, so a new layout is never triggered while one is still playing).
- Layout animation `LAYOUT_ANIM_MS = 400` (outside slots `LAYOUT_ANIM_OUTSIDE_MS = 320`), curve `LAYOUT_EASE`.
- An avoiding card never "snaps back to its old spot before jumping to the new one" (clearing offsets and reordering commit in the same frame), avoiding a flash / twitch.
- On release, if a layout animation is still running, wait an extra `ANIM_SETTLE_BUFFER_MS = 80` before landing so the animation is not hard-cut. The account is kept against **the duration actually used this time** (an outside-slot backfill is booked at 320ms, not 400ms), otherwise the animation has long stopped while the landing still waits.
- Only beyond `OUTSIDE_DIST = 48` from every slot counts as truly leaving the card area (while the card has not left its original spot, "outside = end" does not apply).
- Total landing animation `FLY_TOTAL_MS = 530 = FLY_ANIM_MS(400) + FLY_HOLD_MS(100) + FLY_TAIL_MS(30)`: 0.4s quadratic-bezier arc flight + 0.1s shadowless hold + a timing buffer before the copy unmounts.
- The position segment is only `FLY_MOVE_RATIO = 0.72` of the total (`FLY_MOVE_MS = 288`): framer's `x`/`y` `times` and the shadow tail's `times` derive from the same ratio, so the shadow starts closing the moment the position stops — no separately kept ratios.
- **Scrolling during a drag**: slot rects translate wholesale by the scroll delta (no re-measuring — the cards carry layout-animation `transform`s, so a measurement would read a mid-animation state); hit-testing reruns from the last pointer position. The dragged card stays glued to the pointer while the card area moves.
- **The landing point is the real post-commit position**: on release, first synchronously commit "clear offsets + reorder" (`flushSync`), then measure the dragged card's real rect within the same task as the flight's endpoint — scrolling during the drag or micro-shifts from reordering never put the landing off by pixels.
- **The flight is transform-composited only**: `left`/`top` are the static basis and the path runs through `x`/`y` (compositor-friendly; no reflow, no repaint of the card content); size is set statically (writing `width`/`height` per frame would re-wrap the text every frame). Both the basis and the per-frame offsets snap to the device-pixel grid, and the final frame lands on the snapped landing point, so the composited offsets are whole device pixels and the copy's text raster origin matches a resting card's.
- **The basis is the landing point**: the copy's `left`/`top` are set directly to the (snapped) landing point; path offsets are measured from it and the final frame is `0`. The other way around (basis at the start, final offset = landing delta), once framer rewrites `transform` from its cache after the animation ends, it stacks the offset onto the landing point again: the landing drifts, and the copy flashes back to its origin on unmount.
- **Landing handover**: after the position animation finishes (`FLY_MOVE_MS = 288`), `FLY_HANDOVER_MS` (+40ms) removes `transform` and `will-change`, returning the copy from a composited layer to normal painting — a composited layer's text antialiasing and raster resolution differ from a normal layer's; staying composited after the flight stops reads as "the resolution dropped". The handover is idempotent: the basis is already the landing point, no coordinates change.
- **Release performs no layout change**: the landing settles on the **already-confirmed slot** (`d.entered`), not recomputed from the pointer at the release instant — otherwise the placeholder jumps to an unconfirmed slot at the moment of release, then waits for that layout animation before taking off, chaining into a release stutter. Wherever the placeholder rests is where the card lands (WYSIWYG). `finalizeDrop` is also deferred into a timer, not done synchronously in the `pointerup` callback ("commit reorder + measure landing + two `flushSync` renders" would stall the input pipeline).
- **The flight copy lives in the scroll content layer (zero-latency scroll following)**: the copy portals into `.tuning-scroll` and is positioned in **content coordinates** (`content = viewport − container rect + scrollTop`, converted once when the flight starts), so it scrolls with the content — the follow runs on the browser's compositor thread with no JS involved, hence no lag. Side effect (intentional): the parts of the copy outside the content area are clipped and hidden behind the tab bar.
  (Two historical detours: ① it was first a fixed top-level layer with JS compensation per `scroll` event — the compensation is inherently a frame late and visibly jitters during fast scrolling; ② that compensation's sign was once inverted — the landing is in viewport coordinates and must translate **with** the content; inverting it doubles the scroll error.)
- **Pointer and scroll coalesce into frames**: `pointermove` / `scroll` only record the latest state; positioning and hit-testing settle once per frame in `requestAnimationFrame` (translate the slots first, then position and hit-test). High-rate pointers (high-refresh trackpads, gaming mice) deliver hundreds of `move`s per second; handling each means rewriting styles and re-measuring rects repeatedly within one frame. Before release, the pending frame settles synchronously — the flight's start measured from the overlay must be where the pointer actually stopped.
- **Single curve source**: the avoidance curve `LAYOUT_EASE` and the view-collapse `COLLAPSE_EASE` are the same constant (`lib/motionEase.ts`'s `EASE_OUT_SOFT = cubic-bezier(0.22,1,0.36,1)`); changing one place updates all three (including the scrollbar-length morph).
- The timing relations above are pinned by `lib/dragSortTypes.test.ts` (the debounce lower bound, the sum of the three landing segments, the position ratio, the shared curve, and the CSS grey-bar duration ≤ flight duration).

## 3. Marquee selection (useMarqueeSelection)

- Starts on `pointerdown` on blank space within `.device-body`.
- `.marquee-box`: `position:absolute; z-index:40; border:1px dashed var(--brand-deep); background:rgba(71,195,209,0.1); pointer-events:none`.
- Coordinates snap via `snapPx()`.
- Results drive `.drag-card.is-selected`.
- **View/channel-switch residue handling**: during a view switch `AnimatePresence` may hold both an exiting and an entering `view-stage`; marquee hit-testing measures only within the current `view-stage`; the marquee float fades in only after the view-switch animation ends, so the marquee state is not cleared by residue.

## 4. Motion durations

| Animation | Duration / curve |
|---|---|
| View switch translate | `0.32s easeInOut` (`VIEW_SLIDE_MS`); x translate and overall opacity tween together |
| View height collapse (locked-height rebound) | starts the moment the translate ends, `cubic-bezier(0.22,1,0.36,1)` (fast then slow); duration scaled by the height delta **260–800ms** (`2ms/px`, cap `VIEW_COLLAPSE_MS`); a delta < 4px aligns directly with no animation |
| Sidebar indicator | `0.22s cubic-bezier(0.4,0,0.2,1)` |
| View switch thumb | `0.25s cubic-bezier(0.4,0,0.2,1)` |
| Theme three-state thumb | `0.2s cubic-bezier(0.4,0,0.2,1)` |
| Tab hover/active | `0.15s ease` |
| Card hover shadow | `0.18s ease` |
| Plus (no-device page / tab bar) / install button hover | `0.18s ease` |
| Dialog enter | `0.18s cubic-bezier(0.2,0.8,0.3,1)` (`vx-pop`: lifts up from 2% below) |
| Dialog exit | `0.15s cubic-bezier(0.4,0,0.2,1)` (`vx-pop-out`: mirrors, sinking back 2% below). Shorter than the entrance for a crisp finish |
| Dialog dim layer enter/exit | `0.15s ease` (`vx-overlay-in` / `vx-overlay-out`: `opacity` 0↔1 and `backdrop-filter` `blur(0px)`↔`blur(2px)` tween together — tweening only `opacity` reads as "blur first, then dim"). **Must be a CSS animation**: Radix's Presence holds the unmount on `animationend`; with a `transition`, or no animation name in the closed state, the element unmounts immediately and the exit is never seen |
| Toast enter | `0.2s ease` |
| Drag shadow lift | `0.18s ease-out` |
| Drag landing layout animation | `320–400ms`, same curve source as the view collapse (`cubic-bezier(0.22,1,0.36,1)`, `lib/motionEase.ts`) |
| Drag flight | `530ms = 400ms arc (position segment 288ms, ratio 0.72) + 100ms hold + 30ms buffer` |
| Drag landing grey bar | `0.4s ease-out` (same length as the arc flight; finishes before the copy unmounts) |
| Frequency-response curve shape transition | `320ms cubic-bezier(0.4,0,0.2,1)` (same length as the stroke transition); tweens between any two paths, sample counts aligned by `lib/pathMorph.ts` |
| Curve hover follow | critically damped spring: `FOLLOW_SETTLE_MS = 240` treated as the basically-settled time, `omega = 6.6 / (settle/1000)`; snap-to-stop at position `0.5px` and velocity `40px/s` |
| Curve hover side flip | `280ms ease-out` |
| Selection toolbar glass fade | `180ms ease-out` (`--glass-t` 0→1; exit same length, unmount waits for it) |
| Device page switch | **hard cut**: no opacity tween in or out (`duration: 0`), no fade on the tuning cards. Why: once anything tweens, the whole subtree is promoted to a **composited layer** — text on the layer loses subpixel (LCD) antialiasing and `1px` strokes raster on a different grid; thin elements (section-title glyphs, card group-color borders) flash while solidly filled controls (sliders / toggles / numbers) show no difference (confirmed on device: the entrance flashed, and so did the exit). `key={pageGuid}` makes it a **keyed single-commit hard cut** — the old page unmounts and the new one mounts in the same commit, so the browser paints only after both are done and no empty frame can exist between them; **`AnimatePresence mode="wait"` is deliberately not used**: it splits exit/enter into two commits and waits on the exit callback, guaranteeing an empty frame in between — the bigger the before/after difference, the more visible, reading as the device/curve cards blinking away for a few frames (hit this). `pageGuid` takes `configStore.displayGuid` (**data ownership**), not `selectedGuid` (the click) — `load()` is async, and between the tab click and `readConfig` returning, the store still holds the previous device's blocks; swapping on the click would have the new container (scrollTop=0) render that stale content first, and with the old page scrolled mid-way it reads as "the old page snapped hard to the top" (hit this); `displayGuid` is written in the same step as `blocks`, so the data swap and the page swap share a frame, and the old page — scroll position included — stays until the new data is ready. After the swap, the tint canvas gets a `DEVICE_SETTLE_MS` settle window. The element exiting is the **previous round's old instance**, still carrying the old device's props — so the device-page data (`blocks` / `effects` / `channelOn` / `activeChannel` / `channelNames`) must flow from `App` through `ViewStage` into the views as props; **the views must not subscribe to the store directly**: the store is globally live, and once the old element subscribes to it, it renders as the new device's content (page height included). `bottom-row` (device props + curve card) deliberately sits **outside** the keyed subtree — `CurvePlot`'s morph state (`prevDRef` / animation handle) lives on the component instance and would reset on a keyed remount, degrading device switches into "the old curve flashes a frame, then morphs from zero" (hit this); moved out, the curve instance survives across devices, and device and channel switches share one `alignPaths` 320ms morph. `.bottom-row` is absolutely positioned with its containing block in `.device-body` (`.device-page` is not positioned), so moving it out changes no layout |
| Tint canvas follow fade | per frame, opacity scaled by the "target visibility": the toolbar reads `--glass-t`, in-page targets read their `.device-page`'s live opacity; after a device-page swap, `DEVICE_SETTLE_MS` opens a settle window of per-frame repaints (`lib/edgetint/renderLoop.ts`) |
| Curve recompute throttle | `42ms` (≈24fps, `useThrottledCompute`) |
| Scrollbar fade | `opacity 0.25s ease`; auto fade-out `1.2s` after scrolling stops |
| Theme transition | `0.35s cubic-bezier(0.4,0,0.2,1)` (`.theme-transition`, removed afterwards) |

### 4.1 Card stagger on switches

The staggered card fade-in (`lib/staggerIn.ts` + the `STAGGER_*` constants + the two patches it required) **has been removed entirely**: view switches are back to "translate + overall fade", and the device page is a **hard cut** (no tween in either direction).

Conclusions kept for reference (all hit in practice):

- **Never animate an ancestor's `opacity` for a fade**: it promotes the whole subtree to a **composited layer** — text on the layer loses subpixel (LCD) antialiasing and `1px` strokes raster on a different grid; thin elements (section-title glyphs, card group-color borders) flash while solidly filled controls (sliders / toggles / numbers) show no difference. Hit twice: the entrance fade flashed → switched to an exit-only fade → the exit flashed too. Conclusion: the device page **fades on neither side**.
- The other face of the same cause: an ancestor's `opacity < 1` also **downgrades the panel's own `backdrop-filter` wholesale** (the blur only appears once the fade ends) — most visible on glass panels like the bottom pair (rule in `drag.css`'s `--glass-t` comment).
- The stagger's "drop + overshoot" read as a twitch on the large cards; a pure-fade stagger instead made other elements on the page appear dropped. The switch's feel is left entirely to the view's own `VIEW_SLIDE_MS` (the device page is a hard cut and takes no part).
- Independent layers like the tint canvas **must open their repaint window in the same frame as the page** (`driveFor` runs in a `useLayoutEffect`): a frame late means "the new page is already fading in while the canvas still shows the previous round's tint" — on thin elements, another flash.
- The stagger's original trigger (`viewAnimating` / rAF-waiting for the new `.device-page`) is therefore obsolete and removed.

### 4.2 Frequency-response curve shape transition

The curve "morphs by itself": whenever the path changes it tweens smoothly (`CurvePlot`'s `useLayoutEffect`), **regardless of scenario** — channel switch, filter toggle / add / remove, dragging band parameters, device switch all take the same road; callers never special-case "should this animate".

- **The start is "the previous animation's current position" (`commitStyles()`), falling back to "the previous target"**: `commitStyles()` commits the animation's current value into inline styles before cancelling, so when the target changes mid-morph it continues from where it is instead of jumping back — continuous input (slider drags) reads as **exponential smooth following**, settling on the exact value after release. **Do not** read `getComputedStyle(el).d`: by layout time React has already written the new `d`, so what you get is the new value itself and the tween degenerates to "new → new" = no visible animation (hit this). Only one `d` animation may run at a time; the previous one is cancelled when a new one starts — **only our own** animations are cancelled; the stroke CSS transition in `curve.css` is left alone; on finish/cancel the inline `d` is cleared, returning control to React's attribute.
- **The tween happens purely in pixel space — no range conversion, no clamping**: the curve and the axis are separate concerns — the Y range (`yTop`/`yBottom`, peak-adaptive, stepping every 2dB) changes instantly when it should (the grid is redrawn directly by React), while the curve only travels from where the old shape sat on screen to where the new shape sits. Start and end both already lie inside the plot area, so the interpolation does too. An earlier version "converted to dB by range, then clamped to the plot": when the range narrowed, the old shape necessarily fell outside after conversion, so part of it got squashed flat against the top edge before curving back — terrible to watch (reverted).
- **Sample points align to the "union of key points" (`lib/pathMorph.ts`'s `alignPaths`)**: beyond the 481-point log grid, `buildEvalFreqs` appends each enabled band's center frequency, high-Q neighborhood refinement points, and midpoints of adjacent centers — switch a channel or toggle a filter and the count changes; `d` interpolation requires both sides' command sequences to correspond segment by segment, and with mismatched counts the browser either cannot interpolate or jumps straight to the end value. **Do not** uniformly resample by x to the larger count: refinement points pack dozens of samples around a peak, and uniformizing flattens the peak — it reads as "a low-sampled curve turning into a high-sampled curve". The right way is the **union** of both sides' x as the common grid (the base grids are identical; only each side's extra refinement points differ): the peaks survive intact, both sides' x correspond one-to-one, only y interpolates, and it reads as "each key point rises or falls in place" rather than "the whole line being kneaded". When the two sides' x spans differ (resize), it returns `null` and skips the animation.
- **Use `useLayoutEffect``: it must take over before paint, otherwise the new `d` paints one frame before the animation pulls it back (a flash).
- Duration matches the `stroke` transition: `320ms` / `cubic-bezier(0.4,0,0.2,1)`; the animation sets no `fill`, so on finish it returns to React's written `d` — neither the alignment nor the conversion ever pollutes the final rendered curve.

## 5. Subpixel snapping

`src/lib/snap.ts`:

```ts
export function snapPx(v: number): number {
  return Math.round(v * DPR) / DPR;
}
```

All positioning, animation start/end coordinates, marquee rectangles, and hover-tip transforms go through `snapPx()`, avoiding fractional pixels that blur text or shift by 1px.

## 6. Interaction interruption principles

- Animation affects visuals only; it never blocks interaction.
- Window blur / Escape during a drag cancels the drag.
- The curve hover tip disappears the moment the pointer leaves the SVG; the flip animation can be interrupted mid-flight by a new target.
- View-switch animations can be interrupted mid-flight (rapid re-clicks trigger the enter/exit only once; re-clicking the current view triggers nothing).
