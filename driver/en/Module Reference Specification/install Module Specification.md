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
│   ├── identity.rs        # endpoint stable identity (instance ID / hardware IDs / product / history)
│   ├── stale.rs           # stale-GUID detect/migrate/cleanup (layered identity match)
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

`ensure_can_load()` checks/fixes `DisableProtectedAudioDG`. Service helpers used by install/uninstall:
`stop_audio_service()`, `restart_audio_service()`, `ensure_audio_service_running()`,
`stop_audio_service_with_dependents(stop_timeout_secs)`,
`start_audio_service_with_dependents(start_timeout_secs)`,
`restart_audio_service_wait(stop_timeout_secs, start_timeout_secs)`, and
`restart_endpoint_device(device_guid, is_capture)` for a targeted endpoint restart.

## 5.7 `install/device/stale.rs`

Detects, migrates and cleans up stale GUID records left after Windows re-enumerates an endpoint
(`HKLM\SOFTWARE\VxAPO\Child APOs\{oldGuid}`, `C:\ProgramData\VxAPO\{oldGuid}`,
`snapshots\{oldGuid}.json`). Matches old records to active endpoints through layered stable
identity (endpoint history / device instance ID / hardware IDs).

Layered match (v9.24): a Windows feature update re-rolls endpoint GUIDs and may delete the old
endpoint key `...\MMDevices\Audio\{Render|Capture}\{oldGuid}` entirely, which used to break the
only matching path (records degraded to `unmatched`, so the App offered cleanup but no recovery).
Priority order, recorded in `matched_by`:

1. `endpoint_history` — the old GUID appears in an active endpoint's endpoint history property
   `{4b416b7d-8501-40c1-acfd-97aa9bdc17c8},1` (REG_MULTI_SZ, entries like
   `{0.0.0.00000000}.{guid}`), or an active GUID appears in the record's stored `EndpointHistory`;
2. `device_instance_id` — instance ID read from the old endpoint key when it still exists;
3. `stored_identity` — instance ID persisted in the record key;
4. `hardware_id` — intersecting hardware IDs plus matching product name, requiring a **unique**
   candidate (USB port-change fallback).

Multiple candidates (ambiguous) or no hit stays `unmatched`: cleanup only, never guessed.

Public API: `list_stale_installs() -> Vec<StaleInstall>` (read-only; `target_state` is
`matched_healthy` / `matched_partial` / `unmatched`; `matched_by` is one of `endpoint_history` /
`device_instance_id` / `stored_identity` / `hardware_id` / `null`), `migrate_install(from, to,
config_from, snapshot_from) -> MigrationReport`, `cleanup_orphan(guid)`, `fix_config_acl(guid)`.
Overwritten target files are backed up to `C:\ProgramData\VxAPO\_migration_backup\{guid}\`.
Exposed to CLI as `stale list|migrate|cleanup|fix-acl` and to the App through the corresponding
Tauri commands. `stale migrate` **never stops AudioSrv** (measured 2026-09-16): writing or deleting
endpoint `FxProperties` values only needs a handle with `KEY_SET_VALUE` (`RegKey::open_for_write`),
independent of whether audiodg holds the endpoint — deleting slot values on a live audio stream
succeeds. Historical 0x80070005 errors came from the old implementation opening the key with
`SAM_ALL` (includes CreateSubKey, not granted by the ACL) or with a read-only handle, never from the
audio stack. The healthy path touches only ProgramData files and `HKLM\SOFTWARE\VxAPO\Child APOs\*`
values; `config.toml` is never held open either (one-shot `fs::read` plus
`FindFirstChangeNotificationW` directory notifications, while the App itself rewrites it with
tmp+rename continuously). The repair branch (unhealthy target → `write_install_config` rewrites the
slots) calls `audiodg::restart_endpoint_device` afterwards so the change takes effect — the engine
caches the endpoint APO chain, so a registry-only change is not picked up on the next stream. The
CLI always finishes with `audiodg::ensure_audio_service_running()` (idempotent when already
running). `copy_file`'s tmp→target replace retries up to 5 × 50 ms on ACCESS_DENIED /
SHARING_VIOLATION / LOCK_VIOLATION to absorb the microsecond window where a rename collides with a
DLL read.

## 5.8 `install/device/identity.rs`

Stable endpoint identity across GUID re-rolls: reads instance ID / hardware IDs / product name /
interface name / endpoint history from an active endpoint's `Properties` subkey, plus the identity
values persisted in `HKLM\SOFTWARE\VxAPO\Child APOs\{guid}`. Pure helpers
`normalize_device_id` (loops over `\\?\`, `{1}.`, `{2}.` prefixes), `normalize_endpoint_guid`,
`normalize_endpoint_history`, `merge_endpoint_history` are unit-tested. Identity values are written
as record-key **values** (`DeviceInstanceId`, `DeviceHardwareIds`, `DeviceProductName`,
`EndpointHistory`) — never subkeys, because migration's `copy_values` only copies values. They do
not affect APO loading (which reads only `FxProperties` slots).

## Hard constraints

1. Never depend on pipeline/config.
2. Device enumeration must go through `install/device/info::enumerate_devices`.
3. Registry writes are transactional through `operation.rs`; no direct ad-hoc registry modification.
