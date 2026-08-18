# sys Module Specification

**Boundary**: Does not know what VxAPO is. Does not know audio processing, installation, or configuration.

**Allowed dependencies**: `windows` crate, `windows-core` crate, `core`, `alloc`.

**Forbidden dependencies**: `pipeline/`, `install/`, `config/`, `object/`, `utils/`, `telemetry/`.

## Module tree

```
sys/
├── com.rs                   # COM subsystem module entry
├── registry.rs              # registry module (win32_ok + RegKey + read/write/delete)
├── audio_defs.rs            # Windows audio base definitions (channel mask flags + standard layout mapping)
└── com/
    ├── prelude.rs           # COM base type re-exports + HRESULT constants
    ├── apo_interfaces.rs    # APO interface definitions + IID constants
    └── apo_types.rs         # POD structs + enums + windows-rs type re-exports + constants
```

## Reference dependency table

| Module | May depend on | Must not depend on |
|--------|---------------|--------------------|
| `sys/audio_defs.rs` | `windows` crate | all others |
| `sys/com/prelude.rs` | `windows` crate | all others |
| `sys/com/apo_interfaces.rs` | `prelude`, `apo_types`, `windows` crate | others |
| `sys/com/apo_types.rs` | `windows` crate | others |
| `sys/registry.rs` | `windows` crate, `windows-core` crate, `sys/com/prelude` (`guid_to_string`) | others |

## 3.1 `sys/com/prelude.rs`

Re-exports COM base types, HRESULT constants, and the safe `guid_to_string` helper.

```rust
pub use windows::core::{IUnknown, IUnknown_Vtbl, Interface, interface, GUID, HRESULT, implement};
pub use windows::Win32::System::Com::{IClassFactory, IClassFactory_Impl};
pub use windows::Win32::System::Com::{
    CLSCTX_ALL, CLSCTX_INPROC_SERVER, COINIT_MULTITHREADED,
    CoCreateInstance, CoInitializeEx, CoTaskMemAlloc, CoTaskMemFree,
};
pub use windows::Win32::System::Com::StringFromGUID2;
pub fn guid_to_string(g: &GUID) -> String;
pub const S_OK: HRESULT;
pub const S_FALSE: HRESULT;
pub const E_NOINTERFACE: HRESULT;
pub const E_POINTER: HRESULT;
pub const E_FAIL: HRESULT;
pub const E_UNEXPECTED: HRESULT;
pub const E_INVALIDARG: HRESULT;
pub const CLASS_E_CLASSNOTAVAILABLE: HRESULT;
pub const E_OUTOFMEMORY: HRESULT;
pub const CLASS_E_NOAGGREGATION: HRESULT;
pub const SELFREG_E_CLASS: HRESULT;
pub const APOERR_*: HRESULT;
```

## 3.2 `sys/com/apo_interfaces.rs`

Re-exports the four APO interface structs from `windows::Win32::Media::Audio::Apo`:
`IAudioMediaType`, `IAudioProcessingObject`, `IAudioProcessingObjectRT`, `IAudioProcessingObjectConfiguration`, plus `IAudioSystemEffects`/`IAudioSystemEffects2`/`IAudioProcessingObjectNotifications` as needed.

Exports IID constants:
`IID_IAPO`, `IID_IAPO_RT`, `IID_IAPO_CONFIG`, `IID_IAUDIO_MEDIA_TYPE`, `IID_IAUDIO_SYSTEM_EFFECTS`, `IID_IAUDIO_SYSTEM_EFFECTS2`, `IID_IAUDIO_PROCESSING_OBJECT_NOTIFICATIONS`.

## 3.3 `sys/com/apo_types.rs`

POD structs/enums and windows-rs type re-exports used by APO interfaces.

## 3.4 `sys/registry.rs`

RAII `RegKey` with `open`, `create`, `open_for_write`, `open_sub_key`, typed `read_value`, `write_sz`, `delete_value`, etc. Uses minimal SAM flags to handle MMDevices ACL restrictions.

## 3.5 `sys/audio_defs.rs`

Channel mask constants (`SPEAKER_*`), standard layouts (`KSAUDIO_SPEAKER_MONO/STEREO/QUAD/5POINT1/7POINT1`), `default_channel_mask(channels)`, and `get_channel_names(mask)`.

> Note: `sys/known_folder.rs` was removed from the actual codebase. Per-device config paths are handled by object/CLI using the fixed `C:\ProgramData\VxAPO\{GUID}` root, not a Documents-folder helper.
