# install Module Specification

**Boundary**: Does not know `pipeline/` or `config/`. Responsible only for device APO install/uninstall and device query.

**Allowed dependencies**: `sys/`, `utils/`, `object/vx_reg_props.rs`.

**Forbidden dependencies**: `pipeline/`, `config/`.

## Module tree

```
install/
├── device.rs              # module entry
├── device/
│   ├── endpoint.rs        # endpoint state/name query (read-only)
│   ├── format.rs          # WAVEFORMATEX parsing + channel mask fallback (read-only)
│   ├── slots.rs           # 5-slot read + 3 modes + GUID fallback (read-only)
│   ├── sysfx.rs           # CAPX MSFX template locate/claim/restore
│   └── info.rs            # combined query layer + single device-enumeration entry
├── selector.rs            # module entry (pub mod operation;)
├── audiodg.rs             # DisableProtectedAudioDG check/fix
└── selector/
    └── operation.rs       # install + uninstall + rollback (install_endpoint / uninstall_endpoint / Transaction)
```

> Note: `install/selector/select.rs` was removed. Device selection interaction now lives in CLI/App.

## 5.1 `install/device/endpoint.rs`

Queries Windows audio endpoint device ID, friendly name, and connection state. Read-only.

## 5.2 `install/device/format.rs`

Parses WAVEFORMATEX and provides channel-mask fallback. Read-only.

## 5.3 `install/device/slots.rs`

Reads the 5 APO slots (LFX/GFX/SFX/MFX/EFX), defines `InstallMode`, `SlotValue`, and helpers:
- `read_all_slots(endpoint_key) -> [SlotValue; 5]`
- `detect_install_mode(...)`
- `child_apo_key_exists(device_guid) -> bool`
- `CHILD_APO_PATH_ROOT = r"HKLM\SOFTWARE\VxAPO\Child APOs"`

## 5.4 `install/device/info.rs`

Combines endpoint/slots/format and provides `enumerate_devices()` as the single device-enumeration entry. `DeviceInfo` includes `endpoint`, `install_mode`, `slots`, `format`, `installed_version`, and predicates `is_installed()`, `can_be_upgraded()`, `is_disabled()`, `is_unplugged()`.

Mode detection logic:
- LFX=VxAPO_PRE && GFX=VxAPO_POST -> LfxGfx
- SFX=VxAPO_PRE && MFX=VxAPO_POST -> SfxMfx
- SFX=VxAPO_PRE && EFX=VxAPO_POST -> SfxEfx
- otherwise -> default `SfxEfx`

## 5.5 `install/selector.rs` (module entry)

Current entry only declares `pub mod operation;`. Interactive selection has moved to CLI/App.

## 5.5.1 ~~`install/selector/select.rs`~~ (deleted)

Removed since v9.19.

## 5.5.2 `install/selector/operation.rs`

Core install/uninstall execution with transaction rollback.

```rust
pub struct InstallConfig {
    pub install_premix: bool,
    pub install_postmix: bool,
    pub install_mode: InstallMode,
    pub use_original_apo_premix: bool,
    pub use_original_apo_postmix: bool,
    pub allow_silent_buffer: bool,
    pub auto_adjust: bool,
}

pub fn install_endpoint(
    device_guid: &str,
    device_name: &str,
    connection_name: &str,
    config: &InstallConfig,
    verify: bool,
) -> Result<()>;

pub fn uninstall_endpoint(device_guid: &str) -> Result<()>;
```

- `install_endpoint` performs the 7-step install with transaction rollback and v8.5 full/non-full path detection.
- `uninstall_endpoint` removes only VxAPO-owned parts, deletes childApoPath, and restores original slots when appropriate.

## 5.6 `install/audiodg.rs`

`ensure_can_load()` checks/fixes `DisableProtectedAudioDG`. Also provides `stop_audio_service()` helpers used by CLI before uninstall.

## Hard constraints

1. Never depend on pipeline/config.
2. Device enumeration must go through `install/device/info::enumerate_devices`.
3. Registry writes are transactional through `operation.rs`; no direct ad-hoc registry modification.
