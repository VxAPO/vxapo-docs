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
