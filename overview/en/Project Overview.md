# VxAPO Project Overview

## 1. Positioning

VxAPO is a Windows audio APO (Audio Processing Object) DSP engine that provides:

- Real-time audio DSP: EQ, gain, loudness, reverb, dynamic compression, stereo widening, etc.
- Per-device configuration: one `config.toml` per audio endpoint.
- User-facing tools for different levels: CLI and GUI App.
- Hot reload with dual-chain transition: no pops or interruptions when changing configuration.

## 2. Components

| Component | Description |
|-----------|-------------|
| `vxapo-driver` | Windows APO DLL; core DSP and install logic |
| `vxapo-cli` | Command-line tool for device enumeration, install/uninstall, config, snapshot |
| `vxapo-app` | Tauri 2 + React desktop application for end users |

## 3. Current architecture

```text
vxapo-app (React/Tauri)
    |
    | config.toml / CLI --json
    v
vxapo-cli --------------------> vxapo-driver (install layer)
    |
    | config.toml
    v
Windows audiodg.exe
    |
    v
vxapo_driver.dll (APO)
    -> config parser -> Filter chain -> real-time DSP
```

## 4. Documentation map

- `app/`: App reference specification and UI design.
- `cli/`: CLI reference specification and commands.
- `driver/`: Driver architecture, module specification, configuration/DSP design, installation/troubleshooting.
- `overview/`: this overview.

## 5. Current status

- Driver: TOML config model, hybrid PEQ, DSP numerical safety, install/uninstall, hot reload;
  effects are peq/preamp/aural/reverb/compressor/wide/loudness, plus stale-GUID detection,
  migration and cleanup (device instance ID as the stable identity).
- CLI: `list/status`, `install` (with `--verify`), `uninstall`, `config`, `snapshot`,
  `stale list/migrate/cleanup/fix-acl`.
- App: device list, config read/write, auto-save, install/uninstall, import/export, i18n, and a
  stale-GUID banner (migrate/cleanup + ACL self-repair).
- Production build: `panic = "abort"`, `codegen-units = 1`.

## 6. Key design principles

- Audio is real-time: RT path has zero allocation, zero locks, zero I/O.
- Configuration is per-device.
- Changing configuration does not interrupt sound: hot reload + raised-cosine transition.
- Clear layering: driver does not depend on UI; App/CLI interact through the file system and CLI.
