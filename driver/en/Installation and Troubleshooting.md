# VxAPO Driver Installation and Troubleshooting

## 1. Installation architecture

### 1.1 Devices and slots

Windows audio endpoints mount APOs through 5 slots in FxProperties:

```text
LFX / GFX / SFX / MFX / EFX
```

Supported install modes:

| Mode | PreMix slot | PostMix slot |
|------|-------------|--------------|
| `LfxGfx` | LFX | GFX |
| `SfxMfx` | SFX | MFX |
| `SfxEfx` | SFX | EFX |

### 1.2 Install flow

`install::selector::operation::install_endpoint` performs:

1. Check `DisableProtectedAudioDG`.
2. Read device slots and format.
3. Write VxAPO CLSIDs to the target slots.
4. Backup original APOs under `HKLM\SOFTWARE\VxAPO\Child APOs\{GUID}` (if child APO is enabled).
5. Write `AudioEngine\AudioProcessingObjects\{CLSID}` registration properties.
6. Verify by `CoCreateInstance`.
7. Restart/refresh the audio service.

### 1.3 Uninstall flow

`uninstall_endpoint` removes only VxAPO-owned parts:

- Remove VxAPO CLSIDs from slots.
- Delete the `Child APOs` info area.
- Restore original APOs when possible.
- Do not touch other third-party APOs.

## 2. Device query

`install::device::info::enumerate_devices` is the single device-enumeration entry. It returns:

- endpoint GUID / name / state.
- install mode.
- 5-slot occupancy.
- format info.
- installed version.

## 3. Snapshot and rollback

The CLI uses `snapshot` subcommands to save registry baselines:

- Create baseline before install.
- `snapshot diff` after install.
- Compare with the original baseline after uninstall.
- `snapshot restore` restores registry state.

Snapshot covers registry only; it does not include `config.toml`.

## 4. audiodg not loading: root cause and fix

### 4.1 Symptom

VxAPO is written to endpoint slots, but `audiodg` does not load `vxapo_driver.dll`.

### 4.2 Root cause

After reading the slot CLSID, the Windows audio engine queries:

```text
HKCR\AudioEngine\AudioProcessingObjects\{CLSID}
```

If this key is missing, the engine silently skips the APO. Early implementations only wrote `HKCR\CLSID\{...}` and missed the `AudioEngine` registration key.

### 4.3 Fix

`register_com_class` now writes:

```text
HKCR\AudioEngine\AudioProcessingObjects\{CLSID}
    FriendlyName
    Copyright
    MajorVersion
    MinorVersion
    Flags = 0xd
    MinInputConnections
    MaxInputConnections
    MinOutputConnections
    MaxOutputConnections
    MaxInstances
    NumAPOInterfaces = 1
    APOInterface0 = {FD7F2B29-24D0-4B5C-B177-592C39F9CA10}
```

`unregister_com_class` deletes the key.

## 5. Common checks

| Check | Description |
|-------|-------------|
| Slot value is VxAPO CLSID | confirms install |
| `AudioEngine\AudioProcessingObjects` key exists | missing key causes silent rejection |
| `DisableProtectedAudioDG` is 1 | protected mode may block loading |
| `ProcessingModes` contains the required mode | missing mode prevents invocation |
| Child APO path exists | affects child delegation |
| Administrator rights | required for install/uninstall |

## 6. Stale GUID records: detect, migrate, clean up

### 6.1 Why it happens

Windows may re-enumerate an endpoint with a new GUID (different USB port, driver reinstall,
Bluetooth re-pairing). The old endpoint slot keys disappear, but VxAPO's own records remain:

- `HKLM\SOFTWARE\VxAPO\Child APOs\{oldGUID}` child-APO backup and install info.
- `C:\ProgramData\VxAPO\{oldGUID}\config.toml`.
- `C:\ProgramData\VxAPO\snapshots\{oldGUID}.json`.

The new GUID then looks "not installed" while the old config and snapshot stay on disk.

### 6.2 Stable identity and matching

`install/device/stale.rs` uses the **device instance ID** (e.g. `BTHENUM\...`) as the stable identity
and matches old records against active endpoints:

| `target_state` | Meaning |
|----------------|---------|
| `matched_healthy` | Matched to an active endpoint that is already installed and healthy |
| `matched_partial` | Matched to an active endpoint with an incomplete install (migration repairs it) |
| `unmatched` | No active endpoint; the record can only be cleaned up |

### 6.3 API and commands

```rust
pub fn list_stale_installs() -> Result<Vec<StaleInstall>>;   // scan + match (read-only)
pub fn migrate_install(from, to, config_from, snapshot_from) -> Result<MigrationReport>;
pub fn cleanup_orphan(guid: &str) -> Result<()>;
pub fn fix_config_acl(guid: &str) -> Result<()>;
```

- `StaleInstall`: `guid` / `device_instance_id` / `display_name` / `config_path` /
  `config_mtime_ms` / `snapshot_path` / `snapshot_mtime_ms` / `premix_slot` / `postmix_slot` /
  `inferred_mode` / `has_child_backup` / `has_sysfx_backup` / `target_guid` / `target_name` / `target_state`.
- `MigrationReport`: `success` / `target_guid` / `config_from` / `snapshot_from` /
  `config_migrated` / `snapshot_migrated` / `install_repaired` / `removed_guids` / `warnings`.
- CLI: `vxapo-cli stale list|migrate|cleanup|fix-acl` (see CLI Reference Specification 5.6);
  the App surfaces the same actions through `StaleInstallBanner` on the selected device page.

### 6.4 Migration flow

```text
stale migrate --from <oldGuid> --to <newGuid>
  -> require_admin
  -> stop audio service + taskkill audiodg
  -> migrate_install:
       when config_from / snapshot_from are omitted, choose the source between the old records
       of the same device instance and the target's existing files (latest config, earliest snapshot)
       migrate config.toml + snapshot -> repair the new GUID install state -> delete old records
       existing target files are backed up to C:\ProgramData\VxAPO\_migration_backup\{guid}\
  -> emit MigrationReport (JSON); on failure restart audiosrv
```

Migration runs through the elevated CLI, so the migrated `config.toml` may inherit an administrator
ACL. When the App hits "access denied" on write, it calls `stale fix-acl` (grants the interactive
user Modify via `icacls` SID `*S-1-5-4`) and retries.

### 6.5 Uninstall fallback

If the endpoint key has already been removed by Windows (`find_endpoint_path` fails) but the GUID is
still listed as stale, `vxapo-cli uninstall` falls back to `stale cleanup` so no Child APOs key or
config directory is left behind.
