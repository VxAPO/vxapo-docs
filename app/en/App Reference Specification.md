# App Reference Specification

> **Purpose**: Define the specification boundary, architecture, module structure, data flow, and evolution path for the VxAPO App (`vxapo-app`).
> **Positioning**: The App is the end-user UI layer (see `overview/Project Overview.md` "three-layer separation"). It makes decisions and writes files only; it does not perform real-time audio processing. The only communication channel with the DLL is the file system (`config.toml` / preset TOML).
> **Basis**: Source read of `D:\APO_Project\VxAPO\vxapo-app` (Vite 7 + React 19 + TS 5.8 + framer-motion 13 + lucide-react + @radix-ui + @dnd-kit + @tauri-apps/api 2, 2026-08-23) + `CLI Reference Specification.md` / `overview/Project Overview.md` / driver `Configuration and DSP Design.md`.
>
> **Last revision**: 2026-09-12 — refreshed against the current app/driver/cli code: stale-GUID
> banner + migrate/cleanup, polling/caching strategy, component/hook inventory and backend command table.

---

## 1. App positioning and three-layer separation

| Layer | App can do | App does not do |
|-------|------------|-----------------|
| Decision/presentation | preset selection, semantic/parameter views, effect editing, EQ curve preview, device switching, import/export, marquee batch operations | does not interpret DSP internals |
| File system | write `C:\ProgramData\VxAPO\{GUID}\config.toml` (via backend command); read/write preset TOML; clamp values on write | does not write the registry (install/uninstall goes through CLI/driver) |
| driver/RT | depends on CLI/driver for device enumeration/install/snapshot | does not touch pipeline/RT or do real-time audio processing |

**Core principle**: The DLL only reads `config.toml`; the App is responsible for all decisions; configuration is never written back by the DLL.

---

## 2. Current state (source read, 2026-08-23)

### 2.1 Project skeleton

```text
vxapo-app/
├── package.json            # Vite 7 + React 19 + TS 5.8 + framer-motion 13 + lucide-react 1.x
│                           # + @dnd-kit + @radix-ui (dialog/select/slider/switch)
│                           # + @tauri-apps/api 2 + plugin-dialog + plugin-opener
│                           # + sharp/@resvg/resvg-js (icon pipeline)
│                           # scripts: dev / build / build:win / icon:fix
├── index.html / vite.config.ts / tsconfig*.json
├── src-tauri/              # Tauri 2 Rust backend
│   ├── Cargo.toml / tauri.conf.json / icons/ (multi-size ico + png)
│   └── src/
│       ├── main.rs
│       └── lib.rs          # 18 Tauri commands + elevated CLI wrapper (see section 8)
└── src/
    ├── main.tsx            # entry (App + I18nProvider)
    ├── App.tsx             # main app: device/config/effects/preset/semantic+param views
    ├── App.css / new.css   # new.css aggregates styles via @import; partitions in styles/
    ├── styles/             # theme/topbar/sidebar/cards/tabs/device/curve/dialogs/
    │                       # toast/drag/overlay-scroll/dark (dark last)
    ├── assets/             # VxAPO_icon_v4.svg etc.
    ├── components/         # 28 UI components (see 3.3)
    ├── data/library.ts     # preset library data
    ├── hooks/              # 15 hooks (see 3.4)
    └── lib/                # api/model/toml/effects/blocks/channels/curve/rbj/...
```

### 2.2 Current conclusions

- The Tauri Rust backend exposes 18 commands: config read/write (including fingerprint-checked
  reads), device list, install/uninstall (with rollback), stale-GUID list/migrate/cleanup/ACL
  repair, progress read, import/export, Explorer reveal, language read/write, and window show;
  `opener` / `dialog` plugins are enabled.
- The frontend is split into `components / hooks / lib / data / styles`; business state lives in `App.tsx` and hooks; components are controlled.
- Implemented: device enumeration, config read/write with 300 ms auto-save debounce, 2 s external
  hot-update polling (content-fingerprint short-circuit, paused while the window is hidden),
  install/uninstall (`--verify` loop + progress events), stale-GUID banner (migrate/cleanup plus
  ACL self-repair on denied writes), drag-and-drop import/export, marquee batch operations, i18n,
  custom overlay scrollbar, semantic/parameter dual views, and semantic strength mapping for effects.
- The data model is based on `Block` / `Band` / `EffectItem` (PEQ blocks + non-PEQ effects).

### 2.3 Actual source structure (2026-08-23)

```text
src/
├── App.tsx                    # main layout + device/view/config/marquee orchestration
├── App.css / new.css          # style entry (new.css @import styles/*)
├── main.tsx                   # entry
├── components/                # TopBar, Sidebar, PresetDeck, PresetView, AdvancedView,
│                              # CurvePanel, CurvePlot, CurveGrid, DeviceTabs, DevicePropsCard,
│                              # EffectCard, EffectSemanticCard, SemanticUnitCard, BandParamCard,
│                              # GainSlider, SelectionToolbar, DragCard, DragLayer,
│                              # InstallDialog, UninstallDialog, ImportDialog, SavePresetDialog,
│                              # SettingsDialog, ConfirmDialog, OverlayScrollbar,
│                              # StaleInstallBanner, Toast, VxSelect
├── data/library.ts            # preset library (with zh/en fields)
├── hooks/                     # useConfig, useDevices, useDragSort, useViewAnimation,
│                              # useMarqueeSelection, useChannelState, usePresetActions,
│                              # useCurveHover, useThrottledCompute, useTheme, useToast,
│                              # useInterval, useWindowControls, useEdgeTintLayer, useGlassRing
├── lib/
│   ├── api.ts                 # Tauri command wrappers
│   ├── model.ts               # shared types + type guards
│   ├── toml.ts                # buildToml / parseConfigWithTail
│   ├── effects.ts             # effect defs/params/semantic strength round-trip
│   ├── blocks.ts / accent.ts  # block helpers / accent derivation (OKLCH)
│   ├── channels.ts            # channel labels
│   ├── curve.ts / rbj.ts      # evaluation freqs / RBJ coefficients
│   ├── normalize.ts           # reference-level normalization planning
│   ├── dragSortTypes.ts       # drag-sort types
│   ├── storage.ts             # local storage (custom presets/theme/language)
│   ├── snap.ts                # subpixel snapping
│   └── i18n.tsx + i18n/{core,zh,en}.ts   # Chinese/English i18n
└── styles/                    # partitioned styles (dark.css last)
```

---

## 3. Actual module structure

### 3.1 Root / lib

| File | Responsibility | Constraint |
|------|----------------|------------|
| `main.tsx` | React entry; mounts `App` with `I18nProvider` | no business logic |
| `App.tsx` | main layout + state orchestration: device selection, view switching, config load/auto-save, install/uninstall flow, marquee | composition/state only |
| `App.css` / `new.css` | style entry; `new.css` imports `styles/*` in fixed order | dark.css last |
| `lib/model.ts` | shared types: `Device`, `Block`, `Band`, `EffectItem`, `PresetLibraryEntry`, `ViewMode`, `ThemeMode` + guards | components must not define business types locally |
| `lib/api.ts` | Tauri command wrappers (`readConfig` / `writeConfig` / `listDevices` / `installDevice` / `uninstallDevice` / `rollbackInstall` / `exportConfig`) | no direct file system access |
| `lib/toml.ts` | `buildToml` / `parseConfigWithTail` | matches driver `[[effects]]` model |
| `lib/effects.ts` | effect defs, param validation, defaults, semantic strength round-trip | keys match driver |
| `lib/blocks.ts` / `accent.ts` | block helpers, semantic units, accent derivation (OKLCH) | — |
| `lib/channels.ts` | channel labels/names | — |
| `lib/curve.ts` / `rbj.ts` | evaluation frequencies / RBJ coefficients for curve rendering | — |
| `lib/normalize.ts` | normalization planning (reference level + peak compensation) | — |
| `lib/storage.ts` | local storage (custom presets, theme, language, metadata) | type-guarded |
| `lib/snap.ts` | numeric snapping / subpixel rounding | — |
| `lib/i18n.tsx` + `lib/i18n/*` | Chinese/English i18n (`core.ts` provides `t()` / `setLang`) | keys in zh/en dicts |

### 3.2 data

| File | Responsibility |
|------|----------------|
| `data/library.ts` | preset library with `group_en/name_en/desc_en` fields |

### 3.3 components (actual)

| Component | Responsibility |
|-----------|----------------|
| `TopBar` | Logo, settings/import/export, view switch (semantic/param), window controls |
| `Sidebar` | Preset / Custom / Advanced segmented navigation + draggable divider |
| `PresetDeck` | preset list (compact pill: dot + group + subtitle + added state) |
| `PresetView` | semantic view: group/filter/effect cards + strength sliders |
| `AdvancedView` | parameter view: PEQ blocks/bands, effect params, channel pills |
| `CurvePanel` / `CurvePlot` / `CurveGrid` | frequency response computation, rendering, grid |
| `DeviceTabs` / `DevicePropsCard` | device tabs (with tuning switch) / device properties |
| `EffectCard` / `EffectSemanticCard` / `SemanticUnitCard` | effect param card / semantic strength card / semantic unit card |
| `BandParamCard` / `GainSlider` | band param card / gain slider |
| `SelectionToolbar` | marquee batch toolbar (save/delete/copy to channel) |
| `DragCard` / `DragLayer` | drag-sort card / flying copy layer |
| `OverlayScrollbar` | custom overlay scrollbar (no layout width, fade in/out, survives device switch) |
| `StaleInstallBanner` | stale-GUID banner on the selected device (migrate/cleanup + migration confirm dialog) |
| `InstallDialog` / `UninstallDialog` | install (`--verify` progress loop) / uninstall progress |
| `ImportDialog` / `SavePresetDialog` / `SettingsDialog` / `ConfirmDialog` | import / save custom preset / settings / confirm |
| `Toast` | toast notifications |
| `VxSelect` | unified dropdown (Radix Select wrapper) |

### 3.4 hooks

| Hook | Responsibility |
|------|----------------|
| `useConfig` | read/parse/auto-save config.toml (300 ms debounce); 2 s external poll (fingerprint short-circuit, paused while hidden); ACL repair + retry on denied write; 31-band limit; per-device tuning-state init |
| `useDevices` | device list + stale-GUID list (5 s poll, paused during install), selected device, install/uninstall, stale migrate/cleanup state |
| `useEdgeTintLayer` | external Canvas edge-tint layer (light-source sampling and dirty-rect repaint for cards/toolbar) |
| `useGlassRing` | glass ring geometry: injects measured corner angles and top-highlight falloff angles |
| `useDragSort` | custom pointer-level slot drag engine (avoidance/layout animation/fly) |
| `useViewAnimation` | view-switch animation orchestration (0.32 s slide + 800 ms height collapse + scroll restore) |
| `useMarqueeSelection` | marquee rectangle and card hit-testing (view-switch residue handling) |
| `useChannelState` | channel-selector state (per-device switch and active channel) |
| `usePresetActions` | preset apply / custom preset persistence (block-based) |
| `useCurveHover` | curve hover tip follow and avoidance |
| `useThrottledCompute` | throttled recompute (42 ms ≈ 24 fps, fixed interval while dragging + final pass) |
| `useTheme` | light/dark/system |
| `useToast` | notifications |
| `useInterval` / `useWindowControls` | pausable interval polling / window controls |

### 3.5 src-tauri backend commands

| Command | Responsibility |
|---------|----------------|
| `write_config` / `read_config` | atomic write / read `C:\ProgramData\VxAPO\{guid}\config.toml` |
| `read_config_checked` | fingerprint-checked read: returns the revision and `text = null` when unchanged (poll short-circuit) |
| `read_lang` / `write_lang` | read/write UI language `lang.txt` (shared with the installer) |
| `list_devices` | invoke `vxapo-cli list --json` and deserialize to `Device[]` |
| `install_device` | background-thread `vxapo-cli install --verify --progress-file`, emits `install-progress`, returns `InstallResult` |
| `uninstall_device` / `rollback_install` | elevated CLI uninstall / install-failure rollback |
| `list_stale_installs` | read-only `vxapo-cli stale list --json` → `StaleInstall[]` |
| `migrate_stale_install` / `cleanup_stale_install` / `repair_stale_acl` | elevated CLI `stale migrate` / `cleanup` / `fix-acl`: migrate stale GUIDs, clean up orphans, repair config ACL |
| `read_progress` | read progress file from elevated CLI |
| `read_import_file` / `export_config` / `open_in_explorer` | import/export and Explorer reveal |
| `show_main_window` | set background color per system dark mode, then show main window (no white flash) |

---

## 4. State management and data flow

### 4.1 State sources

- Device state: `useDevices` maintains `devices`, `staleInstalls`, `selectedGuid`, `installedDevices`,
  install/uninstall targets and progress, and stale migrate/cleanup busy state; devices and stale
  records are fetched in one `Promise.allSettled` and shallow-compared so unchanged data keeps its
  previous array reference (avoids whole-tree re-renders).
- Config state: `useConfig` maintains `blocks`, `effects`, `tuningMap` (per-device top-level enabled), `loaded`, `dirtyRef`, and `tailRef` (preserved unknown-effect tail).
- Channel state: `useChannelState` keeps per-device channel-selector state.
- UI state: `App.tsx` / `useViewAnimation` / `useMarqueeSelection` hold view mode, marquee, drag sorting, dialogs, etc.

### 4.2 Data flow

```text
UI operation (add/edit band, toggle enabled, apply preset, semantic strength)
  -> markDirty()
  -> after 300 ms debounce, buildToml(blocks, enabled, effects, channelCtx) + tail
  -> Tauri write_config(guid, content)   # Rust atomic write
  -> driver watcher detects change -> hot reload
  -> useConfig polls read_config every 2 s (backend short-circuits on the content fingerprint;
     paused while the window is hidden, refreshed once on return); external changes refresh the UI
     (skipped while editing)

Curve preview (independent path):
  -> useThrottledCompute(42 ms) recomputes buildEvalFreqs + curveRange (RBJ coefficient cache)
  -> CurvePlot renders SVG; slider move only re-renders the dragged card
```

### 4.3 Derivation rules (actual implementation)

- `buildToml` emits `version = 1` / `enabled` / `[meta]` / `[[effects]]` (`type="peq"` or non-PEQ effects); PEQ blocks also emit `crossover_hz = 200` and `channels` in channel mode.
- `parseConfigWithTail` parses PEQ blocks and non-PEQ effects; the first unknown effect onward is kept as `tail` and appended unchanged on save.
- `applyPreset` checks the 31-band limit and inserts preset bands according to current language/channel.
- Channel mode: `ChannelCtx { mode, first, active }` controls `channels` emission and block filtering; outside channel mode only first-channel blocks are written.
- Effect defaults are merged into params when writing TOML.

### 4.4 Stale-GUID flow (added 2026-09-10)

```text
useDevices (mount + 5 s poll; paused during install)
  -> listDevices() + listStaleInstalls() (Tauri -> CLI `list --json` / `stale list --json`)
  -> shallow-compare and store devices / staleInstalls

StaleInstallBanner (rendered only when a stale record's target_guid == the selected device)
  -> multiple matches: prefer matched_partial, then the newest config_mtime_ms
  -> [migrate] ("migrate + repair" for matched_partial, otherwise "migrate config")
       -> confirm dialog (target device / config / snapshot / inferred mode)
       -> migrateStale(from, to) -> Tauri migrate_stale_install -> CLI `stale migrate`
       -> refresh() reloads devices + stale records
  -> [cleanup] -> cleanupStale(guid) per match -> CLI `stale cleanup` -> refresh()

config write denied (migrated file inherits an administrator ACL)
  -> useConfig.writeConfigSafe catches access denied -> repairStaleAcl(guid)
       -> CLI `stale fix-acl` (grants the interactive user Modify) -> retry the write (once per device)
```

---

## 5. Type specification (`lib/model.ts`)

Actual types are defined in `src/lib/model.ts`:

```ts
export type ViewMode = "preset" | "advanced";
export type SideSection = "preset" | "custom" | "advanced";
export type PeqBandKind = "peaking" | "low_shelf" | "high_shelf" | "low_pass" | "high_pass";

export interface Band {
  fc: number;
  gain_db: number;
  q: number;
  kind?: PeqBandKind;          // default peaking; written as [[effects.bands]].type
}

export interface Block {
  id?: string;                 // client-stable id, not written to TOML
  group?: string;
  name?: string;
  channel?: string;            // first channel short name in channel mode
  enabled: boolean;
  bands: Band[];
}

export interface Device {
  index: number;
  name: string;
  guid: string;
  device_id?: string | null;   // device instance ID (CLI `list --json`)
  connection?: string | null;  // connection name (currently empty; reserved)
  installed_version: string;
  install_mode: string;
  slots: Record<string, string | null>;
  sample_rate?: number | null;
  channels?: number | null;
  bit_depth?: number | null;
  kind?: "playback" | "capture" | null;
  volume?: number | null;
  eapo?: string;
  lost_slot?: string;
}

export interface PresetLibraryEntry {
  id: string;
  group: string;
  name: string;
  desc: string;
  group_en?: string;
  name_en?: string;
  desc_en?: string;
  color?: string;
  bands: (Band & { name?: string; name_en?: string })[];
}

export interface PresetMetaEntry { presetId: string; accent: string; }
export type PresetMeta = Record<string, PresetMetaEntry>;

/** Non-PEQ effect (written to config.toml [[effects]], natively supported by driver) */
export interface EffectItem {
  id?: string;                 // client-stable id; distinguishes per-channel effects (e.g. preamp)
  type: string;
  enabled: boolean;
  params?: Record<string, number | string>;
  channels?: string[];         // default = all channels
}

export type ThemeMode = "light" | "dark" | "system";

/** Stale GUID record (deserialized from CLI `stale list --json`) */
export interface StaleInstall {
  guid: string;
  device_instance_id: string;
  display_name: string;
  config_path?: string | null;
  config_mtime_ms?: number | null;
  snapshot_path?: string | null;
  snapshot_mtime_ms?: number | null;
  premix_slot?: string | null;
  postmix_slot?: string | null;
  inferred_mode: string;
  has_child_backup: boolean;
  has_sysfx_backup: boolean;
  target_guid?: string | null;
  target_name?: string | null;
  /** matched_healthy | matched_partial | unmatched */
  target_state: string;
}

/** Migration report (CLI `stale migrate --json`) */
export interface MigrationReport {
  success: boolean;
  target_guid: string;
  config_from?: string | null;
  snapshot_from?: string | null;
  config_migrated: boolean;
  snapshot_migrated: boolean;
  install_repaired: boolean;
  removed_guids: string[];
  warnings: string[];
}
```

> All business types are centralized in `lib/model.ts` with `isPresetLibraryEntry` / `isPresetMeta` guards.

## 6. Effect model (`lib/effects.ts`, keys match driver)

Built-in effects: `preamp`, `wide`, `aural`, `reverb`, `compressor`, `loudness`.

| Effect | Name | Params | Defaults |
|--------|------|--------|----------|
| `preamp` | Preamp | `gain_db` | `0` |
| `wide` | Stereo Field | `gain` (HF boost) / `air` (center air) / `air_side` (side air) / `mix` (dry/wet) / `crossover_hz` (200–1000) | `0.05 / 0.3543 / 0 / 0.6 / 200` |
| `aural` | Aural Exciter | `tune_hz` / `drive` / `odd` / `even` / `wet` / `dry` | `1760 / 1.7699 / 1.5 / 0.25 / 0.5 / 0.5` |
| `reverb` | Plate Reverb | `room_size` / `decay` / `damping` / `pre_delay_ms` / `low_cut_hz` / `wet` / `dry` | `1 / 0.41 / 0.4083 / 0 / 100 / 0.27 / 0.73` |
| `compressor` | Compressor | `threshold_db` / `ratio` / `knee_db` / `attack_ms` / `release_ms` / `makeup_gain_db` / `wet` / `dry` | `-18 / 4 / 3 / 10 / 100 / 6 / 1 / 0` |
| `loudness` | Loudness EQ | `phon` (target) / `reference_phon` (reference) | `80 / 80` |

Semantic strength round-trip (`semanticStrength` / `applySemanticStrength`):

| Effect | Strength = | Write-back |
|--------|-----------|------------|
| `wide` | `air` (center distance = air absorption depth) | `air = s` (HF boost stays manual) |
| `aural` | `wet / 0.9` | crossfade: `wet = 0.9s`, `dry = 1 - wet` (sum ≤ 1) |
| `reverb` | `wet / 0.9` | `wet = 0.9s`, `dry = 1 - wet`, `decay = 0.2 + 0.7s`, `damping = 0.15 + 0.63·decay`, `pre_delay_ms = clamp(s-0.3,0,0.7)·50`, `room_size = 0.85 + 0.5s` |
| `compressor` | `(ratio - 1) / 19` | `ratio = 1 + 19s` (1:1 → 20:1) |
| `loudness` | `(ref - phon) / 40` | `phon = ref - 40s` |
| `preamp` | `(gain_db + 24) / 48` | `gain_db = 48s - 24` |

> Crossfade rule: for effects with dry/wet (aural/reverb), semantic write-back caps `wet` at 0.9 with `dry = 1 - wet` to keep the sum ≤ 1 and avoid clipping.

---

## 7. Hard decisions

1. The App does not touch real-time audio; DSP is handled by the driver pipeline.
2. Config path is fixed: `C:\ProgramData\VxAPO\{GUID}\config.toml`.
3. Config writes are atomic in Rust (temp file + rename); unknown third-party effects are preserved via `tail`.
4. Auto-save uses a 300 ms debounce; external hot updates are synchronized by a 2 s poll
   (content-fingerprint short-circuit, paused while the window is hidden); device/stale lists poll
   every 5 s and pause during install.
5. Install/uninstall go through elevated CLI with an `--verify` loop and `rollback_install` fallback; the App never writes the registry directly.
6. The top-level `enabled` switch is implemented (whole-chain passthrough); per-device tuning switches are remembered and initialized from disk.
7. EQ curve preview is implemented via `CurvePlot` + `CurveGrid`, with 42 ms throttled recompute while dragging.
8. i18n Chinese/English UI is implemented; the preset library contains Chinese/English fields.
9. Scrollbars are custom-drawn (`os-scroll` hides native scrollbars); the overlay scrollbar takes no layout width, appears on genuine user scrolling (wheel/touch/thumb drag), auto-fades after 1.2 s idle (0.25 s transition), and fades out on device switch instead of hard-disappearing.
10. Stale GUID records are surfaced by `StaleInstallBanner` on the selected device; migration,
    cleanup and ACL repair always go through the elevated CLI `stale` subcommands. The App never
    moves files or changes permissions by itself.

---

## 8. Tauri backend commands (implemented)

| Command | Input | Output | Implementation |
|---------|-------|--------|----------------|
| `list_devices` | — | `Device[]` | CLI `list --json` + serde deserialization |
| `read_config` | guid | `String` | read `C:\ProgramData\VxAPO\{guid}\config.toml` |
| `read_config_checked` | guid, known_revision | `{revision, text?}` | fingerprint-checked read (`text = null` when unchanged, for the 2 s poll) |
| `write_config` | guid, content | `()` | atomic write (tmp + rename) |
| `install_device` | guid | `InstallResult` | background `install --verify --progress-file`, emits `install-progress` |
| `uninstall_device` | guid | `String` | elevated `uninstall -d <guid> --json` |
| `rollback_install` | guid | `String` | elevated rollback after install failure |
| `list_stale_installs` | — | `StaleInstall[]` | CLI `stale list --json` (read-only, no elevation) |
| `migrate_stale_install` | from, to, config_from?, snapshot_from? | `String` (MigrationReport JSON) | elevated `stale migrate --from --to --json` |
| `cleanup_stale_install` | guid | `String` | elevated `stale cleanup -d <guid> --json` |
| `repair_stale_acl` | guid | `String` | elevated `stale fix-acl -d <guid> --json` |
| `read_progress` | tag | `String` | read elevated CLI progress file |
| `read_import_file` | path | `String` | drag-and-drop import file content |
| `read_lang` / `write_lang` | — / lang | `String` / `()` | read/write UI language `lang.txt` (shared with the installer) |
| `export_config` / `open_in_explorer` | guid/path | `()` | export and reveal in Explorer |
| `show_main_window` | — | `()` | set background per system dark mode, show and maximize window |

Additional Tauri plugins: `tauri-plugin-opener`, `tauri-plugin-dialog`.

---

## 9. Current state and future work

### Implemented

- Tauri 2 backend, device list, config read/write, auto-save, elevated install/uninstall with verification loop and rollback, import/export, i18n.
- Preset library, PEQ block editing, curve preview (throttled), channel mode, drag sorting, marquee batch operations, custom preset storage, semantic/parameter dual views, effect semantic strength mapping, custom overlay scrollbar.
- Stale GUIDs: in-page banner with one-click migrate/cleanup and post-migration ACL self-repair
  (wired to the driver/cli `stale` commands).

### Possible future work

- Richer visual editing for non-PEQ effects (currently unknown effects are preserved as tail).
- Preset intensity / multi-preset mixing (can be reintroduced on top of `Block` if needed).
- GUI for snapshot diff/restore (CLI already supports it).
- Packaging, signing, auto-update, and other release engineering.

## 10. Hard constraints

1. No real-time audio processing in the App or its backend.
2. No direct registry writes; install/uninstall/snapshot go through CLI/driver logic.
3. Config syntax must match the driver (`[[effects]]` TOML model); file path is fixed.
4. State has a single source of truth; components are controlled.
5. Clamp/validation rules follow driver behavior (`[-120, +48]`, floor -60, NaN/inf rejected); clamping happens on write only.
6. 31-band limit enforced by `applyPreset` / `addBand`.
7. Stale-GUID migration never writes the registry or file permissions directly from the App; it
   calls the elevated CLI `stale migrate/cleanup/fix-acl` commands.

---

## 11. Related documents

- `overview/Project Overview.md`: three-layer separation, product positioning, system components.
- `CLI Reference Specification.md`: backend-reused commands/device resolution/snapshot logic.
- `driver/Configuration and DSP Design.md`: config.toml model, spec fingerprint, hot reload semantics.
- `driver/Architecture and Module Specification.md`: DSP capabilities and numerical boundaries.
- `项目定位/VxAPO 项目完整方案.txt`: App phase roadmap.
