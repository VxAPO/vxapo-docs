# VxAPO App UI Design (Implemented)

This directory describes the UI structure and interaction logic of the VxAPO App (`vxapo-app`) as actually implemented. The code is the baseline: class names, CSS variables, and animation durations all match `src/new.css`, `src/styles/*`, and `src/components/*`.

> Revision date: 2026-09-12 (glass-ring tokens and geometry backfilled from the current `theme.css` / `dark.css` / `useGlassRing.ts` / `useEdgeTintLayer.ts`; the previous revision, 2026-08-23, split the single `new.css` into the partitioned `src/styles/` files).

## File map

| File | Content |
|---|---|
| `01-design-foundations.md` | theme variables, colors, typography, radius, spacing, shadows, z-index, glass-ring (edge highlight) tokens |
| `02-shell-and-navigation.md` | window skeleton, TopBar, view switching, sidebar, device tabs |
| `03-views-and-cards.md` | semantic/parameter views, group cards, filter cards, effect cards, channel pills |
| `04-curve-and-device-panel.md` | frequency-response curve, hover tip, device properties card, bottom area |
| `05-dialogs-and-selection.md` | dialogs, install/uninstall/save-preset, selection toolbar, Toast, custom scrollbar |
| `06-drag-and-motion.md` | drag engine, marquee selection, animation curves, subpixel snapping |

## Design principles

1. **Light and dark share one structure**: the structure never changes; only CSS variables switch. Dark mode keeps its own base-color and hover-color mapping tables (see `01-design-foundations.md`).
2. **All positioning and animation snap to device pixels**: `snapPx()` handles this uniformly, avoiding 1px offsets and blurry text.
3. **Stability before motion**: animation must never change the outcome of an interaction, and an in-flight animation can be interrupted by input.
4. **Hover, drag, and marquee are pointer-level interactions**: desktop-first, mouse/trackpad friendly, with no dependency on touch gestures.
5. **Scrollbars are custom-drawn**: they occupy no layout width; they appear only during genuine user scrolling and fade out automatically when scrolling stops; programmatic scrolling (view switch / height collapse) never activates a scrollbar, and a scrollbar mid-fade is not "re-entered" by a view switch.
