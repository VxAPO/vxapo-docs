# CLI Reference Specification

> **Purpose**: Define the specification boundary, current capabilities, modification path, and execution constraints for the VxAPO CLI (`vxapo-cli`).
> **Positioning**: The CLI is a developer tool (see `overview/Project Overview.md` "three-layer separation"). It depends on `vxapo-driver` as a library and uses the install-layer API to operate devices. It does not touch pipeline/RT.
> **Basis**: Source read of `D:\APO_Project\VxAPO\vxapo-cli` (v0.3.2) + `D:\APO_Project\VxAPO\vxapo-driver` (2026-08-13).

---

## 1. Current CLI state

### 1.1 File structure (9 files)

```
vxapo-cli/
├── Cargo.toml          # winreg = "0.52" + vxapo-driver (path = "../vxapo-driver") + windows
├── src/
│   ├── main.rs         # no args -> interactive menu; subcommand -> automation
│   ├── commands.rs     # command layer: list/status, install, uninstall, config, snapshot, help
│   ├── app.rs          # viewer-mode App state
│   ├── display.rs      # endpoint list/detail printing
│   ├── endpoint.rs     # standalone winreg enumeration (viewer mode)
│   ├── knowledge.rs    # known GUID/CLSID knowledge base
│   ├── probe.rs        # probe helpers
│   ├── reg.rs          # registry read helpers
│   └── regdump.rs      # endpoint registry dump ([x] command)
```

### 1.2 Current commands

| Command | Behavior | Status |
|---------|----------|--------|
| no-argument launch | interactive menu: 1 viewer mode / 2 driver mode / q quit | implemented |
| `list` / `status` | enumerate devices, show GUID/version/mode/5 slots/EAPO/lost status; supports `--json` | implemented |
| `install -d <device> [--mode ...] [--no-child]` | install/reinstall (auto mode detection, snapshot baseline, auto-register driver, auto-import config) | implemented |
| `uninstall -d <device>` | uninstall (stop audio service, clear slots, verify, snapshot diff) | implemented |
| `config set/show` | write/read `C:\ProgramData\VxAPO\{GUID}\config.toml` | implemented |
| `snapshot diff/restore/create` | registry baseline snapshot, diff, restore | implemented |
| `[0-N]` select endpoint | viewer-mode endpoint detail submenu | implemented |
| `[x]` registry dump | dump endpoint registry keys | implemented |
| `[r]` refresh / `[q]` exit | refresh / exit | implemented |

The CLI is no longer a pure interactive diagnostic tool. It depends on `vxapo-driver` APIs such as `install::device::info::enumerate_devices` and `install::selector::operation`.

---

## 2. Reusable driver APIs

- `InstallConfig` / `install_endpoint` / `uninstall_endpoint` in `install/selector/operation.rs`.
- `DeviceInfo` / `enumerate_devices` / slot helpers in `install/device/info.rs` and `install/device/slots.rs`.
- `ensure_can_load` in `install/audiodg.rs`.
- `ConfigParser` in `config/parser.rs` for read-back verification.

---

## 3. CLI boundary

| Layer | CLI can do | CLI does not do |
|-------|------------|-----------------|
| install-layer API | call `install_endpoint` / `uninstall_endpoint` / `enumerate_devices` | write registry by itself (driver transaction) |
| slot-loss detection | detect via `enumerate_devices` + slot helpers | directly write childApoPath |
| config management | `config set` / `config show` / `config convert` | parse DSP semantics |
| pipeline/RT | — | never touches real-time audio |
| rollback | snapshot before driver changes | — |

---

## 4. Command parameter definitions

### `<device>`

Accepted forms are resolved by `resolve_device` into `(device_guid, device_name, connection_name)`:

| Form | Syntax | Notes |
|------|--------|-------|
| GUID (recommended) | `{XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX}` | stable unique; passed directly to `install_endpoint` |
| enumeration index | non-negative integer `0..N-1` | mapped through `enumerate_devices()[n]` |

### `<file>` (only `config set -f`)

- Absolute or relative path; must exist and be readable.
- Content is copied verbatim to `C:\ProgramData\VxAPO\{GUID}\config.toml`.
- The CLI does not modify file content; syntax validation is performed by driver hot reload.

---

## 5. Command flows

### 5.1 Install

```text
vxapo-cli install -d <device> [--mode ...] [--no-child]
  -> resolve_device
  -> InstallConfig::default_config()
  -> snapshot_device(guid, replace=true)      # baseline before install (registry only)
  -> auto_register_driver()                    # refresh CLSID -> DLL binding
  -> install_endpoint(&guid, &name, &conn, &config, true)
  -> if no config exists, auto-import exe-dir config.toml
```

### 5.2 Config set/show

```text
config set -d <device> -f <file>
  -> path = C:\ProgramData\VxAPO\{guid}\config.toml
  -> write file
  -> config show reads back for file-level verification

config show -d <device>
  -> read and print config.toml
```

### 5.3 Uninstall

```text
uninstall -d <device>
  -> require snapshot baseline
  -> stop audio service / kill audiodg
  -> uninstall_endpoint(&guid)
  -> verify no VxAPO CLSID remains in 5 slots
  -> print snapshot diff
```

### 5.4 Device state machine

| State | Meaning |
|-------|---------|
| U (Unmanaged) | not installed, no snapshot |
| B (Baselined) | not installed, snapshot exists |
| I (Installed) | installed and snapshot exists |
| L (Lost) | installed but installation-mode slots are not VxAPO CLSIDs |

Transitions:
- U --install--> I
- B --install--> I
- I --uninstall--> B
- L --install--> I
- L --uninstall--> B
- Reinstall replaces the baseline.

---

## 6. Hard constraints

1. Do not touch pipeline/RT.
2. Do not write the registry directly; always use driver transaction APIs.
3. Do not duplicate enumeration; use `enumerate_devices` as the single entry point.
4. Config writes belong to the CLI (driver only provides default config fallback).
5. Install/uninstall require administrator privileges; the CLI detects and rejects non-admin operations.
