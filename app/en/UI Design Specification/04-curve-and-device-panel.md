# 04 Frequency-Response Curve & Device Properties Card

## 1. Bottom fixed row

`.bottom-row`: absolutely positioned at the bottom of `.content`, `left:0; right:0; bottom:0; z-index:20; display:flex; gap:12px; padding:10px 12px 12px; pointer-events:none`.

Interactive children:

- `.dev-props-card`: device properties card
- `.curve-wrap`: frequency-response curve card

Both use glassmorphism:

- Light: `background: color-mix(in srgb, var(--card) 56%, transparent); backdrop-filter: blur(40px) saturate(1.4)`.
- Dark: `background: color-mix(in srgb, var(--card) 82%, transparent); backdrop-filter: blur(28px) saturate(1.15)`.

## 2. Device properties card `.dev-props-card`

- `flex:none; min-width:240px; border-radius:16px; padding:10px 18px 12px; border:1px solid var(--border); box-shadow: var(--card-shadow)`.
- Content: device name, sample rate / channels / bit depth, current gain, band-count stats, normalize button.
- dB values in the peak-gain readout and the normalize toasts all go through `lib/format.fmtDb1`: **round to one decimal first, then decide the sign**. Writing `(v >= 0 ? "+" : "") + v.toFixed(1)` leaves a minus sign when |v| < 0.05 — `(-0.04).toFixed(1)` is `"-0.0"`, and normalization pushes the peak to ~0, so the UI displayed "-0.0 dB" (hit this). The numeric `-0` is folded away by `planNormalize`'s `+ 0`, otherwise the number input shows "-0".
- Button `.dev-prop-btn`: 20px tall, padding `0 8px`, radius 9999px, border `1px solid var(--border)`, text `var(--text-secondary)`; hover border/text `var(--brand-deep)` (light) / `var(--brand)` (dark), background `var(--brand-soft)`.

## 3. Frequency-response curve card `.curve-wrap`

- `flex:1; min-width:0; border-radius:16px; padding:10px 18px 14px; border:0; box-shadow: 0 6px 14px -4px rgba(15,23,42,.18); overflow:hidden` (border / edge highlighting is delegated to the outer glass ring; see 01 §9).
- Title row `.curve-head`: 32px tall, title "Frequency Response" 16px 700 + channel selector.
- The SVG shifts left with `margin-left:-10px`, CSS width `calc(100% + 28px)`; the `viewBox` width `curveW` is computed by `CurvePanel` from the container's **content box** (`widthFor`, minimum 660px). **The layout effect and the ResizeObserver must measure the same box**: the former once used `getBoundingClientRect().width` (**includes padding**, and `.curve-wrap` has 18px on each side) while the latter used `contentRect.width` — a 36px difference. Every device-page remount then painted one frame with a viewBox 36px too large → the RO reported 36px smaller → all x coordinates rescaled wholesale, visible as a **subtle horizontal jitter of the axis** (the x=40 vertical axis and its ticks) (hit this). The 8px discrepancy between `widthFor`'s `+20` and the `<svg>`'s `+28` only affects overall scale by ~1% and is irrelevant to stability.
- The curve's `path` / `circle` / `line` stroke colors transition with `stroke 0.32s cubic-bezier(0.4,0,0.2,1)` (theme-switch interpolation).

## 4. Curve drawing (CurvePlot + CurveGrid)

`lib/curve.ts` provides the coordinate mapping and evaluation points:

- Frequency axis `logX`: `x = 40 + (log10(f/20)/3) * (w - 80)` (20Hz→40, 20kHz→w-40).
- Gain axis `dbY`: `y = 24 + ((top - db) / (top - bottom)) * 180` (viewport 220 tall).
- Evaluation points `buildEvalFreqs`: global 20Hz–20kHz log sweep (481 points) + band centers + high-Q (>3) neighborhood refinement (±2 × half-width, ×24 points) + geometric midpoints of adjacent centers; returned sorted by frequency ascending.
- Curve path color: `var(--curve-path)` (light `#009aa2` / dark `#47c3d1`).

The grid is drawn by `CurveGrid`:

- Y grid: the step comes from the span (`>26dB` uses 4dB, else 2dB, `yStepFor`). **The range itself is aligned to a multiple of the step by `axisRange(peakGain, troughGain)`** (first round on the 2dB scale and soft-clamp to 6–30, then align to the step; converges in at most two iterations) — so the grid runs exactly from `yTop` to `yBottom`: the first and last lines sit right on the plot's top/bottom edges, and 0dB necessarily lands on a tick line. Row positions are then rounded via `snapPx()`: the spacing often carries half pixels (e.g. `180×4/32 = 22.5px`); unrounded, text lands on subpixels and the renderer rounds it — the "tick numbers randomly drift downwards" symptom. Labels use `dominantBaseline="middle"` so the vertical center sits exactly on the grid row, with no reliance on empirical offsets like `+3`. Dashed lines `stroke: var(--border)`, `strokeDasharray="4 4"`.
  (Historical detour: the drawing side once used `floor` / `ceil` to force step multiples — when the range was not a step multiple, the first grid line shrank in by half a cell, the dashes missed the top of the axis, and the labels shifted wholesale.)
- X grid: log positions of 20/50/100/200/500/1k/2k/5k/10k/20k (`logX`); positions are also rounded via `snapPx()` — any change to `curveW` rescales the whole mapping; unrounded, vertical lines land on subpixels and the renderer rounds them, and adjacent lines rounding differently jitter (same root cause as the Y axis). Dashes same as the Y axis.
- Axis solid lines: `stroke: var(--curve-axis)` (light `#b3bbc8` / dark `#7c7d7f`).
- Tick labels and crosshair styling: see the `CurveGrid` / `CurvePlot` implementations.

## 5. Performance: throttled recompute

- `useThrottledCompute` (`THROTTLED_COMPUTE_MS = 42`, ≈24fps): slider drags / curve recomputes run on a fixed interval, with one catch-up compute after stopping; a slider move re-renders only the dragged card, preserving drag frame rate.
- **The range and the curve path must come from the same snapshot**: in `App`, `curveSnap = deferredCurve ?? liveCurve` computes the eval frequencies and the peak/trough in one pass; `yTop`/`yBottom` (`axisRange`) and the `blocks` / `evalFreqs` handed to `CurvePlot` all come from it — the drawing side no longer computes eval points from the live blocks itself. The snapshot also carries the **baseline preamp** (`preampGainDb`): it is the curve path's vertical offset and feeds the range; omitting it draws the non-existent curve "old device's chain + new device's baseline" for one step when switching devices (especially "channels off → channels on", where the baseline switches from the global preamp to that channel's) (hit this).
  A vintage mismatch between the two betrays itself: it used to be "range via throttle (deliberately one step behind), path computed live from the current blocks", so on the frame the channel selector closed, the path was already the whole chain while the range was still the single-channel value — the curve tweened one step on the wrong range and recovered the next. The symptom reads as "the transition takes a wrong value once, then recovers" (the shape was always right; the axis was wrong — hit this). The curve **width** is still computed live every time (keeping it consistent with the grid / viewBox and preventing resize overruns), unaffected by the snapshot's lag.
- RBJ coefficients are cached per band (`rbj.ts bandDbCached`) to avoid recomputation.

## 6. Cursor hover tip

Class: `.curve-tip`. Absolutely positioned; `left`/`top` are computed by `useCurveHover` and applied via `transform: translate(x,y)`.

### 6.1 Horizontal avoidance

- The tip is anchored at its top-left corner.
- Rightward it stops at most at 12kHz; leftward at 23Hz.
- Base horizontal position: `cx + sign * gH`, where `gH` derives from the screen slope, range 0–10px.
- Final `left = clamp(base, x(23Hz), x(12kHz))`.
- The wall-hugging side is recorded in `hSide` for the flip animation.

### 6.2 Vertical avoidance

- Vertical gap `gV = 10px`.
- Switching above/below the curve point has a safety zone `SAFE_RADIUS = 10px`; inside it the current side is kept, avoiding jitter.

### 6.3 Follow animation

- **Critically damped spring** (`hooks/useCurveHover.ts`): `FOLLOW_SETTLE_MS = 240` is treated as the "basically settled" time; the angular frequency is `omega = 6.6 / (FOLLOW_SETTLE_MS / 1000)`; each frame advances by spring integration (not a fixed-ratio approach).
- **Snap-to-stop**: settling requires position `FOLLOW_SNAP_PX = 0.5` and velocity `FOLLOW_SNAP_V = 40` (px/s) simultaneously, avoiding micro-jitter from floating-point residue.
- A base-side switch plays a 280ms full-speed ease-out translate.

### 6.4 Content

- `.curve-tip-body`: background `var(--accent-bg)`, text `var(--on-accent)`, radius 10px, padding `6px 10px`.
- Light: no border.
- Dark: `border: 1px solid var(--border-strong)`.
- First line, frequency: `fmtFreq` (≥1000Hz shows kHz, else Hz).
- Second line, gain: `.tip-gain`, color `var(--brand)`.

## 7. Channel selector

- The curve card title row's channel dropdown: once the channel selector is **on**, it lists the channels; switching swaps the curve (`setActiveChannel`, linked to the global active channel). While **off it is locked to the first channel (left)**: the selector is greyed out and cannot be changed (the option list is always the channels; there is no "all channels" entry any more).
- The curve's target channel is **always a single channel**: in `App`, `curveChannel = channelOn ? effActiveChannel : firstChannel`; the curve, the range (`axisRange`), and the peak gain all take blocks by it. **Do not** sum the channels' blocks when the selector is off: the summed response does not exist in the real chain (each block carries its own `channel`; the driver applies them separately), and the instant the selector closes it would paint a "summed curve" that only corrects on the next throttled step — read as "the transition takes a wrong value, then recovers" (hit this). The baseline preamp carries over as before: with channel mode off, the config itself already holds the "merged global preamp" (`setChannelPreampMode` splits/merges on toggle), so it is still taken from the `target === "all"` entry.
- Closing the selector **merges the preamp together with the blocks into the first channel**: `setChannelPreampMode` keeps only blocks with no channel marker or the first channel and erases the channel markers — exactly matching `buildToml`'s `mode=false` on-disk behavior (only the first channel's blocks, no `channels`). Memory and file land in the same step; there is no intermediate state where "the UI is off but the file is still per-channel". Non-first-channel blocks and their bands are dropped here — that is the agreed "closing means a real merge" semantic. If only the preamp were merged in memory and the blocks left for the 2s poll to collect, the other channel's cards would paint first and then vanish (most visible in the semantic view; hit this).
- There is **exactly one** visibility predicate: `lib/filters.visibleBlockFor` (channel mode on = current channel, blocks without a channel marker belong to the first channel; off = blocks without a channel marker or the first channel). Both views' `visible` and `App`'s curve block selection use it; `CurvePanel` no longer filters on its own — it receives the blocks from App's **curve snapshot** (see §5), and re-filtering a throttled snapshot against the live channel would blank the curve for one step on every channel switch. Once each side writes its own predicate, divergences like "the parameter view closed but the semantic view still shows everything" appear (hit this).
- **Channel state is remembered per device** (`stores/channelStore`): switching devices saves the previous device's `channelOn` / `activeChannel` and restores the new device's. Restoration **must wait until the new device's config has arrived** (`useConfigStore.loadedGuid === selectedGuid`; cleared when `load()` starts, written back when parsing finishes), and it must take effect **before paint** (`useChannelState` and App's `configChannelMode` sync effect both use `useLayoutEffect`).
  `load()` is async, so right after a switch the store still holds the **previous** device's `blocks` for a moment: restoring the new device's channel state inside that window filters the **previous** device's chain with the **new** device's channel state — switching from a device with the selector on (preamp split into two per-channel baseline cards) to one with it off flashes a view where the previous device's left/right cards are **merged into one** before jumping to the new device (hit this). Waiting for the config keeps that window self-consistent (the channel state matches the previous device's data on screen), so the page swaps exactly once, the moment the new data lands. The page swap itself honors the same gate: `.device-page`'s key follows `configStore.displayGuid` (written in the same step as `blocks`; **not** cleared when `load()` starts) rather than `selectedGuid`, so the old page — scroll position included — stays until the new data is ready and then hard-cuts (otherwise the new container, scrollTop=0, first renders the previous device's stale content; with the old page scrolled mid-way that reads as "the old page snapped hard to the top"; see 06's device page switch).
  Additionally, a device with **no memory keeps the current state** instead of falling back to "off + left channel" (same reason: the fallback would also paint the previous chain filtered to the left channel during that window; the device page is a hard cut now, so nothing hides that frame).
- The curve card exists in both views, so **channels can be switched from the semantic view too** (the old doc's "semantic view shows all channels and is unusable" is obsolete: channel mode used to force the parameter view; that restriction is gone). The semantic view's filter-section header carries the same channel pills (see 03 §6); both entries share one source.
