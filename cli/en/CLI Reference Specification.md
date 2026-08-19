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
| `install -d <device> [--mode ...] [--no-child] [--verify] [--timeout=<sec>] [--progress-file=<path>]` | install/reinstall; `--verify` runs the verification loop (write → full service restart → named-pipe APO load verification → score/retry), see 5.1b | implemented |
| `uninstall -d <device>` | uninstall (stop audio service, clear slots, verify, snapshot diff) | implemented |
| `config set/show` | write/read `C:\ProgramData\VxAPO\{GUID}\config.toml` | implemented |
| `snapshot diff/restore/create` | registry baseline snapshot, diff, restore | implemented |
| `[0-N]` select endpoint | viewer-mode endpoint detail submenu | implemented |
| `[x]` registry dump | dump endpoint registry keys | implemented |
| `[r]` refresh / `[q]` exit | refresh / exit | implemented |

The CLI is no longer a pure interactive diagnostic tool. It depends on `vxapo-driver` APIs such as `install::device::info::enumerate_devices` and `install::selector::operation`.

---

## 2. Reusable driver APIs

- `InstallConfig` / `write_install_config` / `install_endpoint` / `uninstall_endpoint` in `install/selector/operation.rs` (write-only registry install + full install with tail restart).
- `DeviceInfo` / `enumerate_devices` / slot helpers in `install/device/info.rs` and `install/device/slots.rs`.
- `ensure_can_load` / `stop_audio_service_with_dependents` / `start_audio_service_with_dependents` / `restart_audio_service_wait` in `install/audiodg.rs`.
- `ConfigParser` in `config/parser.rs` for read-back verification.

---

## 3. CLI boundary

| Layer | CLI can do | CLI does not do |
|-------|------------|-----------------|
| install-layer API | call `write_install_config` / `install_endpoint` / `uninstall_endpoint` / `enumerate_devices` / `audiodg::*_with_dependents` | write registry by itself (driver transaction) |
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
vxapo-cli install -d <device> [--mode ...] [--no-child] [--verify] [--timeout=<sec>]
  -> resolve_device
  -> InstallConfig::default_config()
  -> snapshot_device(guid, replace=true)      # baseline before install (registry only)
  -> auto_register_driver()                    # refresh CLSID -> DLL binding
  -> if --verify: install_verify()            # see 5.1b
  -> else: install_endpoint(&guid, &name, &conn, &config, true)
  -> if no config exists, auto-import exe-dir config.toml
```

### 5.1b Verified install (`install --verify`, added 2026-08-19)

- Mode order = `[preferred] + [sfx_efx, sfx_mfx, lfx_gfx]` minus preferred (EAPO fallback order).
- Per mode: `write_install_config` → `stop_audio_service_with_dependents(3)` →
  `start_audio_service_with_dependents(5)` → create named pipe `VxAPODeviceTest`
  (DACL SYSTEM + Administrators + Everyone) + write
  `HKLM\SOFTWARE\VxAPO\DeviceTestPipeName` → `trigger_apo_load`
  (IMMDevice → IAudioClient → GetMixFormat → Initialize only, aligned with EAPO;
  E_PENDING/DEVICE_INVALIDATED retry 5×500ms; no trigger-level timeout) →
  collect pipe messages ≤2s (fixed 200ms×10 iterations) → score.
- Scoring: premix_init 20 / postmix_init 10 / child_premix 2 / child_postmix 1;
  full score render=33, capture=22; child judged against registry expectation
  (absent expected child counts as passed, so clean installs reach full score).
- Full score → `complete success:true`, exit 0; otherwise retry next mode; after all
  modes fail, roll back with `uninstall_endpoint`, ensure the audio service is
  running, emit `complete success:false`, exit 1.
- Events are one JSON object per line on stdout and (when given) appended to
  `--progress-file`; the `test` event carries only `mode` (no scoring fields for
  the user). Global watchdog: 20s hard abort; `--timeout` default 180s (overall
  install budget).

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
