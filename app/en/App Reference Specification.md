# App Reference Specification

> **Purpose**: Define the specification boundary, current architecture, module structure, data flow, and evolution path for the VxAPO App (`vxapo-app`).
> **Positioning**: The App is the end-user UI layer (see `overview/Project Overview.md` "three-layer separation"). It makes decisions and writes files only; it does not perform real-time audio processing. The only communication channel with the DLL is the file system (`config.toml` / preset TOML).
> **Basis**: Source read of `D:\APO_Project\VxAPO\vxapo-app` (Vite 7 + React 19 + TS + Tailwind 4 + framer-motion + lucide-react + @tauri-apps/api 2, 2026-08-13) + `CLI Reference Specification.md` / `overview/Project Overview.md` / driver `Configuration and DSP Design.md`.

---

## 1. App positioning and three-layer separation

| Layer | App can do | App does not do |
|-------|------------|-----------------|
| Decision/presentation | preset selection, effect editing, EQ curve preview, device switching, import/export | does not interpret DSP internals |
| File system | write `C:\ProgramData\VxAPO\{GUID}\config.toml` (via backend command); read/write preset TOML; auto-save | does not write the registry (install/uninstall goes through CLI/driver) |
| driver/RT | depends on driver as library (or reuses CLI logic) for device enumeration/install/snapshot | does not touch pipeline/RT or do real-time audio processing |
| Loudness compensation | only a switch (writes `Loudness: on|off`) | no visualization |

**Core principle**: The DLL only reads `config.toml`; the App is responsible for all decisions; configuration is never written back by the DLL.

---

## 2. Current state (source read, 2026-08-13)

### 2.1 Project skeleton

```text
vxapo-app/
├── package.json            # Vite 7 + React 19 + TS 5.8 + Tailwind 4 + framer-motion 13 + lucide-react
│                           # + @dnd-kit + @radix-ui + @tauri-apps/api 2 + @tauri-apps/cli 2
├── index.html / vite.config.ts / tsconfig*.json
├── src-tauri/              # Tauri 2 Rust backend (initialized)
│   ├── Cargo.toml
│   ├── tauri.conf.json
│   └── src/
│       ├── main.rs
│       └── lib.rs          # Tauri commands: config read/write, device list, install/uninstall, import/export
└── src/
    ├── main.tsx            # entry (App + I18nProvider)
    ├── App.tsx             # main app: device/config/effects/preset/advanced views
    ├── App.css / new.css   # styles
    ├── assets/             # icons
    ├── components/         # 25+ UI components
    ├── data/library.ts     # preset library data
    ├── hooks/              # useConfig/useDevices/useDragSort/useTheme/useToast/...
    └── lib/                # api/model/toml/effects/blocks/channels/storage/i18n/...
```

### 2.2 Current conclusions

- Tauri Rust backend is initialized and provides `write_config`, `read_config`, `list_devices`, `install_device`, `uninstall_device`, `read_progress`, `export_config`, etc.
- `App.tsx` is no longer a single-file draft; components/hooks/lib/data are in place.
- Device enumeration (CLI `list --json`), config.toml read/write, auto-save (300 ms debounce), install/uninstall (elevated CLI), drag-and-drop import/export, and i18n are implemented.
- The data model is based on `Block`/`Band`/`EffectItem` (PEQ blocks + non-PEQ effects), not the earlier `DimensionMapping/Dimension/Filter/Preset` static example model.

---

## 3. Actual module structure

### 3.1 Root / lib

| File | Responsibility | Constraint |
|------|----------------|------------|
| `main.tsx` | React entry; mounts `App` with `I18nProvider` | no business logic |
| `App.tsx` | main layout + state orchestration: device selection, view switching, config load/auto-save, install/uninstall flow | composition/state only |
| `App.css` / `new.css` | global styles | — |
| `lib/model.ts` | shared types: `Device`, `Block`, `Band`, `EffectItem`, `PresetLibraryEntry`, `ViewMode`, etc. | components must not define business types locally |
| `lib/api.ts` | Tauri command wrappers | no direct file system access |
| `lib/toml.ts` | `buildToml` / `parseConfigWithTail` | matches driver `[[effects]]` model |
| `lib/effects.ts` | effect types/defaults/validation | — |
| `lib/blocks.ts` | Block helpers, semantic units | — |
| `lib/channels.ts` | channel labels/names | — |
| `lib/storage.ts` | local storage (custom presets, theme, metadata) | — |
| `lib/snap.ts` | numeric snapping | — |
| `lib/i18n.tsx` | Chinese/English i18n | — |

### 3.2 data

| File | Responsibility |
|------|----------------|
| `data/library.ts` | preset library with `group_en/name_en/desc_en` fields |

### 3.3 components (actual)

`TopBar`, `Sidebar`, `PresetView`, `AdvancedView`, `CurvePanel`, `CurvePlot`, `DeviceTabs`, `DevicePropsCard`, `EffectCard`, `EffectSemanticCard`, `SemanticUnitCard`, `BandParamCard`, `GainSlider`, `SelectionToolbar`, `DragCard`, `DragLayer`, `InstallDialog`, `UninstallDialog`, `ImportDialog`, `SavePresetDialog`, `SettingsDialog`, `ConfirmDialog`, `Toast`, `VxSelect`.

### 3.4 hooks

| Hook | Responsibility |
|------|----------------|
| `useConfig` | read/parse/auto-save config.toml (300 ms debounce), poll external updates |
| `useDevices` | device list, selected device, install/uninstall state |
| `useDragSort` | drag sorting |
| `useTheme` | light/dark/system |
| `useToast` | notifications |
| `useInterval` / `useWindowControls` | timer / window controls |

### 3.5 src-tauri backend commands

| Command | Responsibility |
|---------|----------------|
| `write_config` / `read_config` | atomic write / read `C:\ProgramData\VxAPO\{guid}\config.toml` |
| `list_devices` | invoke `vxapo-cli list --json` and deserialize to `Device[]` |
| `install_device` / `uninstall_device` | elevated CLI install/uninstall, return progress |
| `read_progress` | read progress file from elevated CLI |
| `read_import_file` / `export_config` / `open_in_explorer` | import/export and Explorer reveal |
| `show_main_window` | show main window after render |

---

## 4. State management and data flow

### 4.1 State sources

- Device state: `useDevices` maintains `devices`, `selectedGuid`, `installedDevices`, install/uninstall targets and progress.
- Config state: `useConfig` maintains `blocks`, `effects`, `tuningMap` (top-level enabled), `loaded`, `dirtyRef`, and `tailRef` (preserved unknown-effect tail).
- UI state: `App.tsx` holds view mode, channel mode, drag sorting, dialogs, etc.

### 4.2 Data flow

```text
UI operation (add/edit band, toggle enabled, apply preset)
  -> markDirty()
  -> after 300 ms debounce, buildToml(blocks, enabled, effects, channelCtx) + tail
  -> Tauri write_config(guid, content)   # Rust atomic write to C:\ProgramData\VxAPO\{guid}\config.toml
  -> driver watcher detects change -> hot reload
  -> useConfig polls read_config every 2 s; external changes refresh UI (skipped while editing)
```

### 4.3 Derivation rules (actual implementation)

- `buildToml` emits `version = 1` / `enabled` / `[meta]` / `[[effects]]` (`type="peq"` or non-PEQ effects).
- `parseConfigWithTail` parses PEQ blocks and non-PEQ effects; the first unknown effect onward is kept as `tail` and appended unchanged on save.
- `applyPreset` checks the 31-band limit and inserts preset bands according to current language/channel.
- Channel mode: `ChannelCtx { mode, first, active }` controls `channels` emission and block filtering.

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
  kind?: PeqBandKind;
}

export interface Block {
  id?: string;
  group?: string;
  name?: string;
  channel?: string;
  enabled: boolean;
  bands: Band[];
}

export interface Device {
  index: number;
  name: string;
  guid: string;
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

export interface EffectItem {
  id?: string;
  type: string;
  enabled: boolean;
  params?: Record<string, number | string>;
  channels?: string[];
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
```

> All business types are centralized in `lib/model.ts`.

---

## 6. Hard decisions

1. The App does not touch real-time audio; DSP is handled by the driver pipeline.
2. Config path is fixed: `C:\ProgramData\VxAPO\{GUID}\config.toml`.
3. Config writes are atomic in Rust (temp file + rename); unknown third-party effects are preserved via `tail`.
4. Auto-save uses a 300 ms debounce; external hot updates are synchronized by a 2 s poll.
5. Install/uninstall go through elevated CLI; the App never writes the registry directly.
6. The top-level `enabled` switch is implemented (whole-chain passthrough, driver v9.17+).
7. EQ curve preview is implemented via `CurvePlot`.
8. i18n Chinese/English UI is implemented; the preset library contains Chinese/English fields.

---

## 7. Tauri backend commands (implemented)

See section 3.5 for the implemented command table.

---

## 8. Current state and future work

### Implemented

- Tauri 2 backend, device list, config read/write, auto-save, elevated install/uninstall, import/export, i18n.
- Preset library, PEQ block editing, curve preview, channel mode, drag sorting, custom preset storage.

### Possible future work

- Richer visual editing for non-PEQ effects (currently unknown effects are preserved as tail).
- Preset intensity / multi-preset mixing (can be reintroduced on top of `Block` if needed).
- GUI for snapshot diff/restore (CLI already supports it).
- Packaging, signing, auto-update, and other release engineering.

---

## 9. Hard constraints

1. No real-time audio processing in the App or its backend.
2. No direct registry writes; install/uninstall/snapshot go through CLI/driver logic.
3. Config syntax must match the driver (`[[effects]]` TOML model); file path is fixed.
4. No loudness compensation visualization.
5. State has a single source of truth; components are controlled.
6. Clamp/validation rules follow driver behavior where applicable.

---

## 10. Related documents

- `overview/Project Overview.md`: three-layer separation, product positioning, system components.
- `CLI Reference Specification.md`: backend-reused commands/device resolution/snapshot logic.
- `driver/Configuration and DSP Design.md`: config.toml model, spec fingerprint, hot reload semantics.
- `driver/Architecture and Module Specification.md`: DSP capabilities and numerical boundaries.
- `项目定位/VxAPO 项目完整方案.txt`: App phase roadmap.
