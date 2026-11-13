# 02 Window Skeleton & Navigation

## 1. Overall structure

`App.tsx` renders `.app-shell-new` with the following structure:

```
.app-shell-new
├── .topbar                      top bar
├── .main
│   ├── .sidebar                 sidebar (Preset / Custom / Advanced)
│   │   └── .sidebar-scroll > .sidebar-scroll-inner.os-scroll   + OverlayScrollbar
│   └── .content                 main content
│       ├── .tab-bar             device tabs
│       ├── .device-body
│       │   ├── .device-page (remounts by data ownership: key=pageGuid=displayGuid; the page swaps only once the config has arrived; single-commit hard cut, no AnimatePresence)
│       │   │   ├── .no-device   empty state (no devices)
│       │   │   └── .tuning-scroll.os-scroll
│       │   │       ├── .view-stack > .view-stage (semantic / parameter views)
│       │   │       ├── .sel-toolbar  selection toolbar
│       │   │       └── .marquee-box  marquee rectangle
│       │   ├── OverlayScrollbar (outside AnimatePresence, survives across devices)
│       │   └── .bottom-row      fixed bottom row
│       │       ├── .dev-props-card  device properties card
│       │       └── .curve-wrap      frequency-response curve card
└── dialogs
```

- `.app-shell-new`: `display:flex; flex-direction:column; height:100%; padding:0 10px 10px; gap:0; background: var(--frame)`.
- `.main`: `flex:1; min-height:0; display:flex; gap:10px`.
- `.content`: `flex:1; display:flex; flex-direction:column; background: var(--content); border:1px solid var(--border); border-radius:26px; overflow:hidden`.
- `.content.is-empty` (no devices): falls back to the `var(--frame)` background and hides `.tab-bar`.
- Scroll areas (`.tuning-scroll`, `.sidebar-scroll-inner`, dialog lists) uniformly get `.os-scroll` to hide the native scrollbar; `OverlayScrollbar` draws its own (see 05 §9).

## 2. TopBar

Class: `.topbar`. Height 48px, horizontal flex, `gap:6px`, left/right padding 12px.

Content order:

1. `.logo`: VxAPO icon (`VxAPO_icon_v4.svg`), 18×18px.
2. Settings / Import / Export buttons: `.pill`, 28px tall, padding `0 12px`, radius 9999px.
3. `.spacer`: draggable window region (`data-tauri-drag-region`).
4. View switch: `.seg.view-seg` (semantic view / parameter view), wrapped in the `.view-seg-zone` dead zone — see 2.1.
5. `.spacer`
6. Window controls: `.pill.winbtn`, minimize / maximize / close; the close button's hover uses a `--danger-soft` background with `--danger` text.

### 2.1 View switch (semantic / parameter)

Class: `.seg.view-seg` (styles in `styles/curve.css`), wrapped by `.view-seg-zone`.

- **Both the centering and the dead zone belong to `.view-seg-zone`**: `position:absolute; left:50%; top:50%; transform:translate(-50%,-50%); display:inline-flex; padding:8px`. It does **not** carry `data-tauri-drag-region`, so a click that lands on the 8px band around the control falls onto this container instead of the top bar / `.spacer` drag region — it cannot accidentally trigger the title bar's double-click maximize/restore (the dead zone also can no longer be used to drag the window; that trade-off is intentional).
- `.seg.view-seg` itself is just `position:relative` (the containing block for `.seg-thumb`) plus box model and colors.
- Container: `display:flex; align-items:center; gap:4px; height:26px; border:1px solid var(--border); border-radius:9999px; background:transparent`.
- The two buttons `.view-seg button`: `flex:1; height:22px; padding:0 14px; gap:6px`, 12px font, with icons.
- The large pill `.view-seg .seg-thumb`: absolutely positioned, `top:-2px; bottom:-2px; width:calc(50% - 2px)`, `background: var(--brand-bg); border:1px solid var(--brand); border-radius:9999px; box-shadow: var(--card-shadow)`.
- Right state `.seg-thumb.right`: `left: calc(50% + 2px)`.
- Transition: `left 0.25s cubic-bezier(0.4, 0, 0.2, 1)`, plus 0.32–0.35s background-color / border-color / box-shadow transitions (theme-switch interpolation).
- Selected button: `color: var(--brand)`; icon tilt animation `vx-icon-tilt-left/right 0.65s ease` (direction decided by `data-dir`).
- Disable rule: both buttons are disabled only while `noDevices`. **Opening the channel selector no longer disables the semantic view, nor does it force a switch to the parameter view** — the semantic view also filters its content by channel (`PresetView`'s `visible`), and the channel-switch entry is the selector on the curve card, usable from both views.

## 3. Sidebar

Class: `.sidebar`. Width `clamp(210px, 15vw, 260px)`, `flex:none`, background `var(--card)`, border `1px solid var(--border)`, radius 26px. Internal scroll structure:

```text
.sidebar > .sidebar-scroll (overflow:hidden, border-radius:inherit)
  └── .sidebar-scroll-inner.os-scroll (padding: 12px 14px 14px)
        └── OverlayScrollbar (rounded mode, avoiding 24/26px top/bottom)
```

### 3.1 Three-state switch `.side-seg`

- Three buttons: Preset / Custom / Advanced.
- Unselected: `color: var(--text-weak)`, height 30px.
- Selected: `color: var(--text-primary)`.
- Bottom track `.track`: 2px tall, `background: var(--border)`.
- Active indicator `.ind`: 2px tall, `background: var(--text-primary)`, `transition: left 0.22s cubic-bezier(0.4,0,0.2,1), width 0.22s`.

### 3.2 Preset side (PresetDeck)

- `.preset-list`: `display:flex; flex-direction:column; gap:12px` (flex + gap, no margin-squeeze issues).
- Entry `.adv-pill.preset-pill`: 30px tall, padding `0 12px`, radius 9999px, background `var(--card)`, border `1px solid color-mix(in srgb, var(--preset-accent, var(--brand)) 55%, var(--border))`.
- Structure: an 8px dot `.preset-dot` on the left (group-color background) + `.preset-name` (group name `.p-group` bold in the group color + 8px gap + group subtitle `.p-sub` 12px `--text-weak`) + "added" `.preset-added` on the right (12px weak text).
- **Dark mapping**: the dot, the border, and the group name all read `var(--preset-accent-dark, var(--preset-accent, …))` — the same discipline as the tuning cards' `--card-accent-dark`. `presetCardStyle()` already injects the dark base color into `--preset-accent-dark`; dark mode must consume it, otherwise the preset pills stay on their light color and disagree with the cards (hit this). The overrides live in `dark.css` and the selectors carry `:not(.disabled)`: the grey disabled states of `.preset-pill.disabled` are shared between light and dark; if specificity lets the override win, the disabled state gets re-tinted with color. The custom side's `.preset-pill.custom` shares the same set.
- Hover: `box-shadow: var(--hover-shadow); transform: translateY(-1px)`; dot `scale(1.25)`.
- Used `.disabled`: `border-color: var(--border)`, dot/group name turn `--text-weak`, hover lift disabled.

### 3.3 Custom side

- Same `.preset-pill.custom` as the preset side, plus a delete button `.preset-pill-del` on the right (20×20px circle, hover `--danger-soft` background + `--danger` text).
- Empty state `.preset-empty`: a single line of centered plain text (12px `--text-weak`), no background pattern.

### 3.4 Advanced side

- `.adv-list`: vertical stack, 6px spacing.
- `.adv-cat`: category heading, 13px 700, color `var(--text-secondary)`, margin `14px 2px 2px`.
- `.adv-pill`: row button, 30px tall, padding `0 12px`, radius 9999px, background `var(--card)`, border `1px solid var(--border)`, card shadow.
- `.adv-pill .adv-plus`: plus sign, color `var(--brand-deep)` (light) / `var(--brand)` (dark), flex:none, rotates 45° when active.
- Hover: `box-shadow: var(--hover-shadow); transform: translateY(-1px)`.
- Disabled: `color: var(--text-weak); border-style:dashed; box-shadow:none`.

### 3.5 Sidebar resize divider

- `.sidebar-resizer`: absolutely positioned just outside the sidebar's right edge at `calc(-1 * (--side-gap + 1px))`, 10px wide.
- On hover/drag the dot indicator switches from `--text-weak` to `--brand`.
- Sidebar width changes run through JS logic; CSS only handles the visuals.

## 4. DeviceTabs

Class: `.tab-bar`, at the top of `.content`, 38px tall, padding `0 10px`, margin-top 8px.

Each device tab `.tab-item`:

- Active `.tab-item.active`: z-index 3.
- Tab button `.tab-btn`: `padding: 7px 30px`, font-size 13px, font-weight 500, border-radius 16px, margin `4px 6px 4px 2px`, width 100%, text ellipsis.
- Unselected hover: `background: var(--frame)` (light) / `var(--tab-inactive)` (dark), text `var(--text-secondary)`.
- Active: `background: var(--tab-active-bg)`, text `var(--text-primary)`.

### 4.1 Tuning enable toggle `.tab-dot`

- Absolutely positioned at the tab's left (left 10px), 13×13px circle, border 2px solid `var(--text-weak)`, transparent background.
- On `.tab-dot.on`: background/border `var(--brand)`.
- Hover: `border-color: var(--brand-deep)` (light) / `var(--brand)` (dark, off) / `var(--brand-deep)` (dark, on).
- Clicking toggles that device's tuning switch (remembered per device, without switching tabs); the initial state comes straight from disk.

### 4.2 Close tab `.tab-close`

- 18×18px circle, absolutely positioned on the right, hover background `--danger-soft`, text `--danger`.

### 4.3 Add device `.tab-add`

- 22×22px circle, `color: var(--text-secondary)`.
- Hover: `color: var(--brand-deep)` (light) / `var(--brand)` (dark), background `var(--brand-soft)`.
- Transition: `background-color / color 0.18s ease`.

## 5. Empty state

- `.no-device`: shown inside `.device-body` when there are no devices, centered, background `var(--frame)`.
- `.no-device-logo`: 256×256, `opacity:0.16; filter:grayscale(1) invert(1)`.
- `.no-device-row`: install entry button.
- `.no-device-plus`: 26px circle, background `var(--brand)`, text `var(--on-accent)`; hover background `rgba(88,205,219,0.75)`; transition `background-color / color 0.18s ease`.
- `.no-device-tip`: the tip text color derives from `--text-primary` (`color-mix(in srgb, var(--text-primary) 50%, transparent)`; dark uses `--text-weak`); on theme switches it is added to the `.theme-transition` exemption list, so it follows the root variable interpolation without lagging a beat.
