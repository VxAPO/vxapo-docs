# 03 Views & Cards

## 1. View switching (semantic / parameter)

- The semantic view (`PresetView`) and the parameter view (`AdvancedView`) use `motion.div` with the class `.view-stage`.
- Switch animation: `x: ±100%`, duration `0.32s`, ease `easeInOut`.
- **Both views stay in the DOM** (`components/ViewStage.tsx`): the non-current view is `display:none`; on a switch the exiting view gets `.is-exiting` (absolute positioning, stepping out of the document flow), plays out the translate, then collapses to `display:none`. `AnimatePresence` mount/unmount is no longer used — rebuilding the 31 parameter-card DOM subtrees on every switch caused a first-frame spike.
- `.view-stage`: `display:flex; flex-direction:column; gap:22px`.
- **Both views share one visibility predicate**: `lib/filters.visibleBlockFor` (channel mode on = current channel, blocks without a channel marker belong to the first channel; off = fall back to the first channel). The semantic view is **no longer** "show everything when channel mode is off" — that painted the other channel's cards when the selector closed (they then vanished with the merge; see 04 §7).
- The switch orchestration lives in `useViewAnimation`: record the old scroll position → lock `.view-stack`'s height → the moment the translate ends (`VIEW_SLIDE_MS = 320ms`, aligned by timer, not waiting for the animation callback) immediately start the height-collapse transition (`cubic-bezier(0.22,1,0.36,1)`, fast then slow; duration scaled by the height delta at `260ms + 2ms/px`, capped at `VIEW_COLLAPSE_MS = 800ms`; a delta < 4px aligns directly with no animation). The collapse endpoint is the **new view's natural height** (`min-height` only affects the rendered height when larger than the content; an endpoint of 0 makes small height deltas look like teleporting); during the transition the scrollbar length uses max(old content, new content), so it never shortens before lengthening.

## 2. Card grid

- `.cards.device-cards`: grid layout.
- Columns: 2 by default; ≥900px 3; ≥1200px 4; ≥1700px 5.
- Gap 12px, `align-items:start`; group cards span all columns (`grid-column: 1 / -1`).

## 3. Filter cards (parameter view)

Class: `.band-card` (parameter view); band rows inside group cards share the `.band-params` structure.

- Background `var(--card)`, border `1px solid var(--border)`, radius 16px, padding `12px 14px`, card shadow.
- Enabled `.enabled`: border `rgba(71,195,209,0.55)`.
- Disabled `.disabled`: title/label/input text `var(--text-weak)`; `input` non-interactive.

### 3.1 Header `.b-head`

- `.enable-dot`: numeric toggle, 26×20px, radius 9999px, border 2px solid `var(--text-weak)`, text `var(--text-secondary)`, tabular-nums.
- `.enable-dot.on`: background/border `var(--brand)`, text `var(--on-accent)`.
- `.enable-dot:hover`: `border-color: var(--brand-deep)` (light) / `var(--brand)` (dark).
- `.b-type`: PEAK type text, 12px 600.

### 3.2 Parameter area `.band-params`

Each row `.band-param-row`: `display:flex; align-items:center; gap:8px`.

- Label `.band-param-label`: `var(--text-weak)`, 12px.
- Slider `.gs-root` (Radix Slider wrapper, thumb `--thumb` + border `--gs-thumb-border`; disabled thumb `--gs-thumb-disabled`) with `margin-right:4px`: at full value the 16px thumb's center sits on the track's right edge and protrudes 8px outward, which would eat the row gap and press against the input — the extra 4px only widens the "track end → input" spacing; the label side is untouched (at 2px the thumb's hover border still expanded onto the input, hence 4px).
- Input `.gain-input`: 56px wide, 22px tall, radius 8px, border `1px solid var(--border)`, background `var(--card)`, text `var(--text-primary)`, right-aligned.
  **Value protection (`NumInput`, uncontrolled)**: while typing, nothing is written back to the DOM and nothing enters the store — a `type=number` intermediate state ("-", emptied) reads back as an empty string, and a controlled write-back would push `Number("")=0` into the box, making a minus sign impossible to type; on blur/Enter the value is clamped back into the driver range (fc 20..20000, gain −30..30, q 0.1..12) and echoed into the input; it is written to the store only if it differs; empty/invalid input reverts to the previous value; while focused, external values do not sync in.
  **Precision**: display and blur commit converge per `precision` — Hz frequencies (fc / tune_hz / crossover / low_cut) are always integers, everything else keeps at most 2 decimals (inputs never expose long-float defaults). A further clamp backs this up before writing to disk (see the *App Reference Specification* §4.3). The semantic view's header frequency input `.fc-num` follows the same rules.
- focus-visible: `border-color: var(--card-accent, var(--brand)); box-shadow: 0 1px 3px rgba(15,23,42,0.1)`. The border color **follows the card's group color** (the same tokens as `.band-card.enabled.sem-group`'s outline); ungrouped cards fall back to `--brand`; dark mode switches to `--card-accent-dark` (see `dark.css`). It used to be hard-coded to `--brand` (the default cyan): the moment a grouped card's input focused, it flashed a foreign color that read as a control from somewhere else.

## 4. Group cards (semantic view / preset view)

Class: `.group-card`. Group cards share the same card, with `sem-group` added when a `group` exists.

- `.sem-group`: 3px group-color left edge, `border-left: 3px solid var(--card-accent, var(--brand))`.
- `.group-card.enabled`: border `rgba(71,195,209,0.55)`.
- `.group-card.enabled.sem-group`: border `color-mix(in srgb, var(--card-accent, var(--brand)) 55%, transparent)`.
- `.sem-group .enable-dot.on`: background/border `var(--card-accent, var(--brand))`, text `var(--on-accent)`.
- `.sem-group .enable-dot:hover`: border `var(--card-accent, var(--brand))`.
- `.sem-group .enable-dot.on:hover`: border `var(--card-accent-hover, ...)` (light) / `var(--card-accent-hover-dark, ...)` (dark).
- `.sem-chip`: group tag, 20px tall, padding `0 8px`, radius 9999px, background `color-mix(in srgb, var(--card-accent, var(--brand)) 14%, transparent)`, text `var(--card-accent, var(--brand-deep))`; hover background 24% mix.
- Group head `.group-head`: order circle `.ord` + group name (14px 600; rename via the `.sem-select` dropdown) + toggle.

### 4.1 Numeric toggle rules

| Case | Light | Dark |
|---|---|---|
| No group, off hover | `var(--brand-deep)` | `var(--brand)` |
| No group, on hover | `var(--brand-deep)` | `var(--brand-deep)` |
| Grouped, off hover | group color | dark group color |
| Grouped, on hover | light group hover color | dark group hover color |

## 5. Effect cards

Class: `.effect-card`.

- Background `var(--card)`, border `1px solid var(--border)`, radius 16px, padding `12px 14px`.
- Enabled `.effect-card.enabled`: border `rgba(71,195,209,0.55)`.
- Header `.effect-head`: enable dot `.effect-dot` (same rules as `.enable-dot`) + `.effect-name` + delete `.close-x`.
- `.effect-desc`: description text, `var(--text-weak)`.
- Param rows `.effect-param-row`: label + control (slider / dropdown / input); the slider also carries `margin-right:4px` (same reason as `.band-param-row`: at full value the thumb protrudes 8px, keeping it off the input). The generic numeric input (`NumInput`) stores the raw typed value in the store and displays the committed value when unfocused (`display={clamped}`; intermediate states never enter the store); on blur the committed value is written back to the store (the same two layers as the band inputs — "commit on blur + clamp before write", see §3.2).
- Semantic strength row `.effect-strength-row`: strength percentage `.g-val`, text `var(--text-weak)`.

### 5.1 Built-in effects (semantic strength mapping; see *App Reference Specification* ch. 6)

`preamp` (baseline level), `wide` (stereo field), `aural` (harmonic exciter), `reverb` (plate reverb), `compressor`, `loudness` (loudness compensation). The semantic strength slider maps only the core parameter:

- `wide`: strength = center air (sense of distance); high-frequency compensation stays manual in the parameter view.
- `aural` / `reverb`: strength = wet (`wet = 0.9s`, `dry = 1 - wet`, sum ≤ 1); reverb additionally drives decay / damping / pre_delay / room_size.
- `compressor`: strength = the internal 0–1 slope value itself (gain-reduction slope `1 − 1/ratio`, the same value as the parameter view; converted to x:1 when written to disk).
- `loudness`: strength = how far the target loudness digs below the reference loudness.
- `preamp`: strength = gain (−24dB → +24dB).

## 6. Channel pills `.ch-pill`

- **Both the semantic and parameter views' filter-section headers show** the channel pill group (while channel mode is on): outer `.col-head` (`inline-flex` + `var(--card)` background + `padding: 0 12px`), pills `height:24px; padding:0 16px; border-radius:9999px`. Both views share the same markup; switching calls `setActiveChannel` (the same source as the curve card's channel selector).
- Unselected: `border:1px solid var(--border)`, text `var(--text-secondary)`; hover border/text `var(--brand-deep)` (light) / `var(--brand)` (dark).
- Selected `.ch-pill.active`: border/text `var(--brand-deep)` (light) / `var(--brand)` (dark), background `var(--brand-soft)`.

## 7. Card delete button `.close-x`

- 18px circle, hover background `var(--danger-soft)`, text `var(--danger)`.

## 8. Empty hint `.hint-row`

- Shown when filters / effects are empty: "add tuning from the sidebar" / "add effects from the sidebar".
- Can be shifted horizontally via `hintShift` to stay centered in the empty state.
- Error state `.hint-row.err`: `color: var(--danger)`.
