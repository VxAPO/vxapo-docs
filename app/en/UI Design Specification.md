# VxAPO App UI Design Specification (Implemented)

> This directory describes the actual UI structure and interaction logic of `vxapo-app`. It is based on the code: class names, CSS variables, and animation durations match `src/new.css`, `src/App.css`, and `src/components/*`.

## File map (Chinese originals in `app/zh/UI 设计规范/`)

| File | Content |
|------|---------|
| `01-design-foundations.md` | theme variables, colors, typography, radius, spacing, shadows, z-index |
| `02-shell-and-navigation.md` | window skeleton, TopBar, view switching, sidebar, device tabs |
| `03-views-and-cards.md` | preset/advanced views, group cards, filter cards, effect cards, channel pills |
| `04-curve-and-device-panel.md` | frequency response curve, floating panel, device properties card, bottom area |
| `05-dialogs-and-selection.md` | dialogs, install/uninstall/save preset, selection toolbar, Toast |
| `06-drag-and-motion.md` | drag engine, marquee selection, animation curves, subpixel snapping |

## Design principles

1. Light/dark share the same structure; only CSS variables change.
2. All positioning and animation snap to device pixels via `snapPx()`.
3. Interaction stability comes before motion; animations must not change interaction results and can be interrupted.
4. Hover, drag, and marquee are pointer-level interactions; desktop-first, mouse/trackpad friendly.

## 01 Design foundations

- Theme tokens: background, surface, text, accent, border, shadow, hover states for light and dark.
- Typography scale and font stack.
- Radius, spacing, and elevation tokens.
- Z-index hierarchy for sidebar, topbar, dialogs, toasts, drag layers.

## 02 Shell and navigation

- Window skeleton with custom title bar (window controls).
- TopBar: app title, theme toggle, settings.
- Sidebar: segmented switcher for Preset | Custom | Advanced.
- Device tabs: device list/properties and install/uninstall entry.

## 03 Views and cards

- Preset view: preset library cards, group cards, one band per card.
- Advanced view: filter/effect cards, PEQ band editing, channel scope pills.
- Cards support drag sorting, delete, and selection.
- Empty states: centered hint when no cards.

## 04 Curve and device panel

- Frequency response curve with standard grid:
  - Y: +6/+3/0/-3/-6/-10/-16 dB dashed lines.
  - X: 20/50/100/200/500/1k/2k/5k/10k/20k logarithmic dashed lines.
- Device info line: `{channels}ch · {sample_rate}Hz · {bit_depth}bit`.
- Channel selector without a "channel" text label.
- Device properties card shows GUID, version, mode, slots, EAPO/lost status.

## 05 Dialogs and selection

- Confirm dialogs, install/uninstall dialogs with progress, save preset dialog, import dialog, settings dialog.
- Selection toolbar for multi-select operations.
- Toast notifications for success/error.

## 06 Drag and motion

- Drag engine based on @dnd-kit; drag cards can be reordered.
- Marquee selection for multiple cards.
- Animation curves and durations from `framer-motion`; subpixel snapping via `snapPx()`.
- Motion must not block or alter interaction results.
