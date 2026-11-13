# sys Module Specification

**Boundary**: Does not know what VxAPO is. Does not know audio processing, installation, or configuration.

**Allowed dependencies**: `windows` crate, `windows-core` crate, `core`, `alloc`.

**Forbidden dependencies**: `pipeline/`, `install/`, `config/`, `object/`, `utils/`, `telemetry/`.

## Module tree

```
sys/
├── com.rs                   # COM subsystem module entry
├── consts.rs                # single source: VxAPO CLSIDs + APOERR_* error codes
├── registry.rs              # registry module (win32_ok + RegKey + read/write/delete)
├── audio_defs.rs            # Windows audio base definitions (channel mask flags + standard layout mapping)
└── com/
    ├── prelude.rs           # COM base type re-exports + HRESULT constants
    ├── apo_interfaces.rs    # APO interface re-exports + IID constants
    └── apo_types.rs         # POD structs + enums + windows-rs type re-exports + constants
```

## Reference dependency table

| Module | May depend on | Must not depend on |
|--------|---------------|--------------------|
| `sys/audio_defs.rs` | `windows` crate | all others |
| `sys/consts.rs` | `windows-core` (`GUID`, `HRESULT`) | all others |
| `sys/com/prelude.rs` | `windows` crate, `sys/consts.rs` (APOERR re-export) | all others |
| `sys/com/apo_interfaces.rs` | `prelude`, `apo_types`, `windows` crate (3 system interfaces for IIDs only) | others |
| `sys/com/apo_types.rs` | `windows` crate, `sys/com/prelude` (APOERR re-export) | others |
| `sys/registry.rs` | `windows` crate, `windows-core` crate, `sys/com/prelude` (`guid_to_string`), `utils/vx_error` | others |

## 3.1 `sys/com/prelude.rs`

**Responsibility**: Re-export the COM base types and HRESULT constants of `windows-rs`, plus the safe GUID-formatting helper.

**References**:
- `windows::core::{IUnknown, IUnknown_Vtbl, Interface, interface, GUID, HRESULT, implement}`
- `windows::Win32::System::Com::{IClassFactory, StringFromGUID2}`

**Exports to**: all `sys/com/` submodules, plus every module that needs GUID formatting — `sys/registry.rs`, `install/selector/operation.rs`, `install/device/slots.rs`, etc.

> **Necessary cross-layer dependency**: `guid_to_string` is essentially the safe narrowing of
> `StringFromGUID2` (an FFI `unsafe` call) and is a re-export-style normative function. Any
> module that genuinely needs standard GUID string formatting may depend on it (the consumer
> module's dependency table must list it explicitly).

**Public API**:

```rust
// ── Type re-exports ────────────────────────────────────────────────────────
pub use windows::core::{
    IUnknown, IUnknown_Vtbl, Interface, interface, GUID, HRESULT, implement,
};
pub use windows::Win32::System::Com::{IClassFactory, IClassFactory_Impl, StringFromGUID2};
pub use windows::Win32::System::Com::{
    CLSCTX_ALL, CLSCTX_INPROC_SERVER, COINIT_MULTITHREADED, CoCreateInstance, CoInitializeEx,
    CoTaskMemAlloc, CoTaskMemFree,
};

// ── GUID formatting helper (the only allowed "custom function" exception) ──
/// Formats a GUID as the standard `{XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX}` string
/// (braces included; unlike `GUID::Debug` which has no braces).
///
/// Rationale: `windows::core::GUID` does not implement `Display`/`ToString`; relying on
/// `Debug` output depends on an unstable format. The official FFI binding `StringFromGUID2`
/// requires an `unsafe` call; this function narrows that `unsafe` into a single safe
/// boundary, which is a system-layer responsibility. The stack buffer is 40 WCHAR
/// (the API requires >= 39 = 38 chars + NUL).
pub fn guid_to_string(g: &GUID) -> String;

// ── HRESULT constants ──────────────────────────────────────────────────────
pub const S_OK: HRESULT = HRESULT(0);
pub const S_FALSE: HRESULT = HRESULT(1);
pub const E_NOINTERFACE: HRESULT = HRESULT(0x8000_4002u32 as i32);
pub const E_POINTER: HRESULT = HRESULT(0x8000_4003u32 as i32);
pub const E_FAIL: HRESULT = HRESULT(0x8000_4005u32 as i32);
pub const E_UNEXPECTED: HRESULT = HRESULT(0x8000_FFFF_u32 as i32);
pub const E_INVALIDARG: HRESULT = HRESULT(0x8007_0057u32 as i32);
pub const CLASS_E_CLASSNOTAVAILABLE: HRESULT = HRESULT(0x8004_0111u32 as i32);
pub const E_OUTOFMEMORY: HRESULT = HRESULT(0x8007_000Eu32 as i32);
pub const CLASS_E_NOAGGREGATION: HRESULT = HRESULT(0x8004_0110u32 as i32);
pub const SELFREG_E_CLASS: HRESULT = HRESULT(0x8004_0201u32 as i32); // not exported by the windows crate; defined locally

// ── APO-specific HRESULT error codes (defined once in `sys/consts.rs`; re-exported here) ──
pub use crate::sys::consts::{
    APOERR_ALREADY_INITIALIZED, APOERR_ALREADY_UNLOCKED, APOERR_APO_LOCKED, APOERR_BUFFERS_OVERLAP,
    APOERR_FORMAT_NOT_SUPPORTED, APOERR_INVALID_APO_CLSID, APOERR_INVALID_COEFFCOUNT,
    APOERR_INVALID_COEFFICIENT, APOERR_INVALID_CONNECTION_FORMAT, APOERR_INVALID_CURVE_PARAM,
    APOERR_INVALID_INPUTID, APOERR_INVALID_OUTPUT_MAXFRAMECOUNT, APOERR_NOT_INITIALIZED,
    APOERR_NUM_CONNECTIONS_INVALID,
};
```

APOERR values (single source `sys/consts.rs`):

| Constant | Value |
|----------|-------|
| `APOERR_ALREADY_INITIALIZED` | `0x887D_0001` |
| `APOERR_NOT_INITIALIZED` | `0x887D_0002` |
| `APOERR_FORMAT_NOT_SUPPORTED` | `0x887D_0003` |
| `APOERR_INVALID_APO_CLSID` | `0x887D_0004` |
| `APOERR_BUFFERS_OVERLAP` | `0x887D_0005` |
| `APOERR_ALREADY_UNLOCKED` | `0x887D_0006` |
| `APOERR_NUM_CONNECTIONS_INVALID` | `0x887D_0007` |
| `APOERR_INVALID_OUTPUT_MAXFRAMECOUNT` | `0x887D_0008` |
| `APOERR_INVALID_CONNECTION_FORMAT` | `0x887D_0009` |
| `APOERR_APO_LOCKED` | `0x887D_000A` |
| `APOERR_INVALID_COEFFCOUNT` | `0x887D_000B` |
| `APOERR_INVALID_COEFFICIENT` | `0x887D_000C` |
| `APOERR_INVALID_CURVE_PARAM` | `0x887D_000D` |
| `APOERR_INVALID_INPUTID` | `0x887D_000E` |

`sys/consts.rs` also defines the two VxAPO COM class IDs, fixed at generation time:

```rust
pub const CLSID_VXAPO_PRE_MIX: GUID;  // 41C34613-D391-459D-A039-72B2B15A1A1D
pub const CLSID_VXAPO_POST_MIX: GUID; // B4A97313-ABC0-45ED-9C33-428B20D39428
```

**Forbidden**: no custom types or business logic. **Single exception**: the `guid_to_string` function — a necessary safe operation because `GUID` lacks `Display`; a system-layer responsibility. No other functions may be added.

> **Consumer boundary**: `guid_to_string` may be depended on by any module as needed (it is the safe narrowing re-export of `StringFromGUID2`); the consumer's dependency table must list this dependency explicitly.

## 3.2 `sys/com/apo_interfaces.rs`

**Responsibility**: re-export the 4 APO interface structs already provided by windows-rs 0.62.2 plus the corresponding `*_Impl` traits (required by `#[implement]` implementers); export the interface IID constants.

**References**:
- `crate::sys::com::prelude::*`
- `crate::sys::com::apo_types::*` (`REFERENCE_TIME`, `APO_REG_PROPERTIES`, `APO_CONNECTION_DESCRIPTOR`, `APO_CONNECTION_PROPERTY`, `UNCOMPRESSED_AUDIO_FORMAT`)
- `windows::Win32::Media::Audio::Apo` (4 APO interface structs re-exported directly: `IAudioMediaType`, `IAudioProcessingObject`, `IAudioProcessingObjectRT`, `IAudioProcessingObjectConfiguration`)
- `windows::Win32::Media::Audio::Apo::{IAudioProcessingObjectNotifications, IAudioSystemEffects, IAudioSystemEffects2}` (3 system interface structs re-exported, IIDs only)
- `windows::core::{Interface, IUnknown}` (`::IID`, interface root)

**Exports to**: `object/apo.rs`, `object/apo/{aggregate,child,negotiate,process}.rs`, `object/factory.rs`, `object/vx_reg_props.rs`, `pipeline/format.rs`.

### `*_Impl` trait re-exports (required by `#[implement]` COM objects)

```rust
pub use windows::Win32::Media::Audio::Apo::{
    IAudioProcessingObject_Impl,
    IAudioProcessingObjectRT_Impl,
    IAudioProcessingObjectConfiguration_Impl,
};
```

> These `*_Impl` traits are generated by the windows-rs `define_interface!` macro and are
> required when deriving `#[implement(...)]`; the custom APO object (`object/apo.rs`)
> implements them to provide the COM methods.
>
> **Wording clarification**: re-exporting `*_Impl` traits (method signatures only, for
> implementers) is allowed; what is forbidden is implementing business function bodies here
> (this file must not contain any COM method implementation logic).

```rust
// IAudioSystemEffects (marker interface, EAPO alignment): its `_Impl` has an empty body.
// With an independent parent slot (SFX/MFX/EFX) the engine queries this interface to
// confirm "system effects"; without it the engine refuses to load.
pub use windows::Win32::Media::Audio::Apo::{IAudioSystemEffects, IAudioSystemEffects_Impl};
```

### Interface re-exports (4 windows-rs structs)

**`IAudioMediaType`** (IID: `4e997f73-b71f-4798-873b-ed7dfcf15b4d`):

| Method | Signature | Description |
|--------|-----------|-------------|
| `IsCompressedFormat` | `fn IsCompressedFormat(&self, pf_compressed: *mut u32) -> HRESULT` | Whether the format is compressed |
| `IsEqual` | `fn IsEqual(&self, p_type: *mut IAudioMediaType, pdw_flags: *mut u32) -> HRESULT` | Compare two media types |
| `GetAudioFormat` | `fn GetAudioFormat(&self) -> *const c_void` | Read-only `WAVEFORMATEX*` |
| `GetUncompressedAudioFormat` | `fn GetUncompressedAudioFormat(&self, p_format: *mut UNCOMPRESSED_AUDIO_FORMAT) -> HRESULT` | Uncompressed format description |

**`IAudioProcessingObject`** (IID: `fd7f2b29-24d0-4b5c-b177-592c39f9ca10`):

| Method | Signature | Description |
|--------|-----------|-------------|
| `Reset` | `fn Reset(&self) -> HRESULT` | Reset internal APO state |
| `GetLatency` | `fn GetLatency(&self, p_latency: *mut REFERENCE_TIME) -> HRESULT` | Latency in 100 ns units |
| `GetRegistrationProperties` | `fn GetRegistrationProperties(&self, pp_props: *mut *mut APO_REG_PROPERTIES) -> HRESULT` | Registration properties; caller frees with `CoTaskMemFree` |
| `Initialize` | `fn Initialize(&self, cb_data_size: u32, pby_data: *mut u8) -> HRESULT` | Initialization |
| `IsInputFormatSupported` | `fn IsInputFormatSupported(&self, p_opposite: *mut IAudioMediaType, p_requested: *mut IAudioMediaType, pp_supported: *mut *mut IAudioMediaType) -> HRESULT` | Input format query; `S_FALSE` means an alternative format was returned |
| `IsOutputFormatSupported` | `fn IsOutputFormatSupported(&self, p_opposite: *mut IAudioMediaType, p_requested: *mut IAudioMediaType, pp_supported: *mut *mut IAudioMediaType) -> HRESULT` | Output format query |
| `GetInputChannelCount` | `fn GetInputChannelCount(&self, p_count: *mut u32) -> HRESULT` | Input channel count |

**`IAudioProcessingObjectRT`** (IID: `9e1d6a6d-ddbc-4e95-a4c7-ad64ba37846c`):

| Method | Signature | Description |
|--------|-----------|-------------|
| `APOProcess` | `fn APOProcess(&self, num_input: u32, pp_inputs: *mut *mut APO_CONNECTION_PROPERTY, num_output: u32, pp_outputs: *mut *mut APO_CONNECTION_PROPERTY)` | Returns void (the RT path must not propagate errors) |
| `CalcInputFrames` | `fn CalcInputFrames(&self, output_frames: u32) -> u32` | Input frames needed for a given output frame count |
| `CalcOutputFrames` | `fn CalcOutputFrames(&self, input_frames: u32) -> u32` | Output frames producible from a given input frame count |

**`IAudioProcessingObjectConfiguration`** (IID: `0e5ed805-aba6-49c3-8f9a-2b8c889c4fa8`):

| Method | Signature | Description |
|--------|-----------|-------------|
| `LockForProcess` | `fn LockForProcess(&self, num_input: u32, pp_inputs: *mut *mut APO_CONNECTION_DESCRIPTOR, num_output: u32, pp_outputs: *mut *mut APO_CONNECTION_DESCRIPTOR) -> HRESULT` | Lock the processing configuration |
| `UnlockForProcess` | `fn UnlockForProcess(&self) -> HRESULT` | Unlock the processing configuration |

### IID export constants

```rust
// ── 4 APO interface IIDs (via the Interface trait's ::IID) ─────────────────
pub const IID_IAPO: GUID = IAudioProcessingObject::IID;
pub const IID_IAPO_RT: GUID = IAudioProcessingObjectRT::IID;
pub const IID_IAPO_CONFIG: GUID = IAudioProcessingObjectConfiguration::IID;
#[cfg(test)]
pub const IID_IAUDIO_MEDIA_TYPE: GUID = IAudioMediaType::IID;

// ── 3 system-interface IIDs (referenced directly from windows-rs) ──────────
pub const IID_IAUDIO_SYSTEM_EFFECTS: GUID = IAudioSystemEffects::IID;
#[cfg(test)]
pub const IID_IAUDIO_SYSTEM_EFFECTS2: GUID = IAudioSystemEffects2::IID;
#[cfg(test)]
pub const IID_IAUDIO_PROCESSING_OBJECT_NOTIFICATIONS: GUID =
    IAudioProcessingObjectNotifications::IID;
```

> The 3 system interfaces are implemented by Windows (not by the APO); the APO side only
> needs their IIDs for `QueryInterface` queries. In the current code `IID_IAUDIO_MEDIA_TYPE`,
> `IID_IAUDIO_SYSTEM_EFFECTS2` and `IID_IAUDIO_PROCESSING_OBJECT_NOTIFICATIONS` (and the
> interface-type imports used only by them) compile in test builds only (`#[cfg(test)]`).
> `IID_IAUDIO_SYSTEM_EFFECTS` is always available because `IAudioSystemEffects` itself must
> be implemented by the APO object (marker interface).

### AEC interface reservation (O4, v6.6; feature-gated)

If Windows 11 AEC (acoustic echo cancellation) APO support is added in the future, the following interface re-exports and IID constants are reserved here (currently **not implemented**; declaration and gating position only, to avoid a large future rework):

```rust
// feature-gated: `#[cfg(feature = "aec")]` (not a default feature)
pub use windows::Win32::Media::Audio::Apo::{
    IApoAcousticEchoCancellation,
    IApoAuxiliaryInputConfiguration,
    IApoAuxiliaryInputRT,
};

pub const IID_IAPO_ACOUSTIC_ECHO_CANCELLATION: GUID = IApoAcousticEchoCancellation::IID;
pub const IID_IAPO_AUXILIARY_INPUT_CONFIGURATION: GUID = IApoAuxiliaryInputConfiguration::IID;
pub const IID_IAPO_AUXILIARY_INPUT_RT: GUID = IApoAuxiliaryInputRT::IID;
```

> **Gating design**: `feature = "aec"` is not a default feature — non-AEC builds must not
> pull in the Windows 11 SDK interface surface. When implemented, `object/apo.rs` would
> carry 3 interfaces + 3 AEC interfaces = **six interfaces** (mirroring tympan-apo's
> `AecApoInstanceCom` nine interfaces: six SISO + three AEC). Currently declaration only.

**Forbidden**: contains no implementation logic.

## 3.3 `sys/com/apo_types.rs`

**Responsibility**: **re-export the APO types already provided by windows-rs 0.62.2 (POD structs, `APO_FLAG`, `APO_BUFFER_FLAGS` and their constants)**; only define what windows-rs lacks (`UNCOMPRESSED_AUDIO_FORMAT`, `AUDIO_FLOW_TYPE`, `REFERENCE_TIME`, signature/comparison constants, APOERR error codes, compile-time assertions).

**References**:
- `windows::core::{GUID, HRESULT}`
- `windows::Win32::Media::Audio::Apo::{APO_FLAG, APO_BUFFER_FLAGS, APO_REG_PROPERTIES, APO_CONNECTION_DESCRIPTOR, APO_CONNECTION_PROPERTY, APOInitSystemEffects, APOInitBaseStruct}` (and all associated constants, re-exported directly)
- `windows::Win32::Media::Audio::{WAVEFORMATEX, WAVEFORMATEXTENSIBLE, PKEY_AudioEndpoint_GUID}`
- `windows::Win32::UI::Shell::PropertiesSystem::IPropertyStore`, `windows::Win32::System::Com::StructuredStorage::PROPVARIANT`, `windows::Win32::System::Variant::{VARENUM, VT_BSTR, VT_CLSID, VT_LPWSTR}`

**Exports to**: `sys/com/apo_interfaces.rs`, `pipeline/`, `install/`, `object/`, `config/`.

### 3.3.1 Types re-exported from `windows-rs`

**`APO_FLAG`** (SDK type used directly, not custom-defined):

```rust
pub use windows::Win32::Media::Audio::Apo::{
    APO_FLAG,
    APO_FLAG_NONE,
    APO_FLAG_INPLACE,
    APO_FLAG_SAMPLESPERFRAME_MUST_MATCH,
    APO_FLAG_FRAMESPERSECOND_MUST_MATCH,
    APO_FLAG_BITSPERSAMPLE_MUST_MATCH,
    APO_FLAG_MIXER,
    APO_FLAG_DEFAULT,
};
```

> windows-rs' `APO_FLAG` is a `repr(transparent)` wrapper around `i32`; its associated
> constants fully cover the SDK enumeration and support bitwise operations. No custom type
> is needed.
>
> **Flow direction**: one-way output. The APO sets the `Flags` field when building
> `APO_REG_PROPERTIES` → the Windows engine reads it. Windows never writes `APO_FLAG`
> back to the APO.

**`APO_BUFFER_FLAGS` and associated constants** (windows-rs types, used directly, no custom enum):

```rust
pub use windows::Win32::Media::Audio::Apo::{
    APO_BUFFER_FLAGS,
    BUFFER_INVALID,   // 0
    BUFFER_VALID,     // 1
    BUFFER_SILENT,    // 2
};
// For internal semantic references: BUFFER_INVALID / BUFFER_VALID / BUFFER_SILENT
```

**`WinAPO_BUFFER_FLAGS`** (windows-rs type alias, for external interaction):

```rust
pub use windows::Win32::Media::Audio::Apo::APO_BUFFER_FLAGS as WinAPO_BUFFER_FLAGS;
```

**POD structs** (provided by windows-rs 0.62.2, re-exported directly, not custom-defined):

```rust
pub use windows::Win32::Media::Audio::Apo::{
    APO_REG_PROPERTIES,          // registration properties (1092-byte conformant array)
    APO_CONNECTION_DESCRIPTOR,   // connection descriptor (x64 = 40 / x86 = 20 bytes)
    APO_CONNECTION_PROPERTY,     // connection property (x64 = 24 / x86 = 16 bytes)
};
pub use windows::Win32::Media::Audio::{WAVEFORMATEX, WAVEFORMATEXTENSIBLE};
```

> Struct size/layout is guaranteed by the windows-rs bindings, so the compile-time
> assertions target only the custom items (see 3.3.8).

### 3.3.1b `APOInitSystemEffects` (Initialize data)

```rust
pub use windows::Win32::Media::Audio::Apo::{APOInitSystemEffects, APOInitBaseStruct};
```

> Provided by windows-rs 0.62.2; re-exported, not custom-defined. `Initialize`
> (object 7.1.8) receives `pby_data` pointing to `APOInitSystemEffects`.
> Measured fields: `{ APOInit: APOInitBaseStruct, pAPOEndpointProperties,
> pAPOSystemEffectsProperties: ManuallyDrop<Option<IPropertyStore>>, pReserved,
> pDeviceCollection }` — the endpoint GUID is extracted via
> **`pAPOSystemEffectsProperties`** → `IPropertyStore::GetValue(PKEY_AudioEndpoint_GUID)`
> (measured on Windows 11 as `VT_LPWSTR`; `VT_CLSID` kept compatible)
> (v7.2 definition, v7.6 revised to the measured path, v8.10 corrected the property-store field).

### 3.3.2 Custom enums (missing from windows-rs)

**`AUDIO_FLOW_TYPE`**:

```rust
#[repr(u32)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum AUDIO_FLOW_TYPE {
    PULL = 0,
    PUSH = 1,
}
```

**`APO_CONNECTION_BUFFER_TYPE` (removed)**:

> **Status**: `sys/com/apo_types.rs` **no longer defines this enum** (0 hits repo-wide).
> The module header notes that windows-rs 0.62.2 already provides `APO_REG_PROPERTIES` /
> `APO_CONNECTION_DESCRIPTOR` / `APO_CONNECTION_PROPERTY` / `APO_FLAG` / `APO_BUFFER_FLAGS`;
> this module re-exports all of them and keeps only the custom items windows-rs lacks:
> `UNCOMPRESSED_AUDIO_FORMAT`, `AUDIO_FLOW_TYPE`, `REFERENCE_TIME`, the three connection
> signature constants (`ACDS`/`ACPS`/`ACP2`), the three comparison flags and the APOERR
> error codes.
>
> The pin-flags field in the connection property is named **`u32BufferFlags`** (type
> `APO_BUFFER_FLAGS`) with values `BUFFER_VALID` / `BUFFER_SILENT` / `BUFFER_INVALID` —
> imported and used by `pipeline/buffer.rs`. Older documents calling it `buffer_flags`
> are outdated.

### 3.3.3 Type alias

```rust
pub type REFERENCE_TIME = i64;
```

### 3.3.4 Custom POD structs (missing from windows-rs)

> `APO_REG_PROPERTIES` (1092 bytes), `APO_CONNECTION_DESCRIPTOR` (x64 = 40 / x86 = 20 bytes)
> and `APO_CONNECTION_PROPERTY` (x64 = 24 / x86 = 16 bytes) are provided by windows-rs 0.62.2
> and re-exported in 3.3.1; only the custom item is kept here.

**`UNCOMPRESSED_AUDIO_FORMAT`** (36 bytes, no pointers, identical across platforms):

```rust
#[repr(C)]
#[derive(Clone, Copy)]
pub struct UNCOMPRESSED_AUDIO_FORMAT {
    pub guid_format_type: GUID,
    pub dw_samples_per_frame: u32,
    pub dw_bytes_per_sample_container: u32,
    pub dw_valid_bits_per_sample: u32,
    pub f_frames_per_second: f32,
    pub dw_channel_mask: u32,
}
```

### 3.3.5 Signature constants

```rust
pub const APO_CONNECTION_DESCRIPTOR_SIGNATURE: u32 = u32::from_le_bytes(*b"ACDS");
pub const APO_CONNECTION_PROPERTY_SIGNATURE: u32 = u32::from_le_bytes(*b"ACPS");
pub const APO_CONNECTION_PROPERTY_V2_SIGNATURE: u32 = u32::from_le_bytes(*b"ACP2");
```

### 3.3.6 Comparison flag constants

```rust
pub const AUDIOMEDIATYPE_EQUAL_FORMAT_TYPES: u32 = 0x0000_0002;
pub const AUDIOMEDIATYPE_EQUAL_FORMAT_DATA: u32 = 0x0000_0004;
pub const AUDIOMEDIATYPE_EQUAL_FORMAT_USER_DATA: u32 = 0x0000_0008;
```

### 3.3.7 APO-specific HRESULT error codes

> APOERR constants are **defined once in `sys/consts.rs`** (single source, together with the
> VxAPO CLSIDs) and re-exported by `sys/com/prelude.rs` (3.1) and here, preventing the
> HRESULT constants from scattering across system submodules:

```rust
pub use crate::sys::com::prelude::{
    APOERR_ALREADY_INITIALIZED,
    APOERR_ALREADY_UNLOCKED,
    APOERR_APO_LOCKED,
    APOERR_BUFFERS_OVERLAP,
    APOERR_FORMAT_NOT_SUPPORTED,
    APOERR_INVALID_APO_CLSID,
    APOERR_INVALID_COEFFCOUNT,
    APOERR_INVALID_COEFFICIENT,
    APOERR_INVALID_CONNECTION_FORMAT,
    APOERR_INVALID_CURVE_PARAM,
    APOERR_INVALID_INPUTID,
    APOERR_INVALID_OUTPUT_MAXFRAMECOUNT,
    APOERR_NOT_INITIALIZED,
    APOERR_NUM_CONNECTIONS_INVALID,
};
```

Values: see the table in 3.1.

### 3.3.8 Compile-time assertions

A `const _: () = { ... };` block verifies the re-exported SDK types and the custom items:

| Assertion | Expected value |
|-----------|----------------|
| `size_of::<APO_FLAG>()` | 4 |
| `size_of::<AUDIO_FLOW_TYPE>()` | 4 |
| `size_of::<UNCOMPRESSED_AUDIO_FORMAT>()` | 36 |
| `size_of::<APO_REG_PROPERTIES>()` | 1092 |

> **Removed**: the `APO_CONNECTION_BUFFER_TYPE` size/repr assertions were deleted together
> with that enum (the type no longer exists). There are no field-offset assertions —
> windows-rs bindings guarantee struct layout.

**Forbidden**: contains no business logic.

## 3.4 `sys/registry.rs`

**Boundary**: pure registry utility layer. Knows nothing about APO, audio processing, installation business, or config files.

**Allowed dependencies**: `windows` crate, `windows-core` crate, `sys/com/prelude` (`guid_to_string`), `utils/vx_error` (error type).

**Forbidden dependencies**: `pipeline/`, `install/`, `config/`, `object/`, `telemetry/`.

> Merged from the former `read.rs`, `write.rs` and `delete.rs` into a single module. GUID
> formatting goes through `sys/com/prelude::guid_to_string` (the `unsafe` of
> `StringFromGUID2` is narrowed to that single safe boundary; depending on the prelude's
> `guid_to_string` is an allowed exception). Errors are unified into
> `VxApoError::Registry` (text from `windows::core::Error`). The privilege-elevation
> functions (`make_writable`, `take_ownership`, `PrivilegeGuard`,
> `enable_take_ownership_privilege`, `create_administrators_sid`) **have been removed from
> the code entirely** (the old note "migrated to `install/permission.rs`" is stale — that
> file does not exist and the symbols have 0 hits repo-wide). Current ACL handling lives in
> `install/device/stale/acl.rs`: `fix_config_acl(guid)` calls `grant_interactive_modify`
> on the config directory and snapshot file, which runs
> `icacls /grant *S-1-5-4:(OI)(CI)M` (recursive for directories, no `/T` for files);
> failures are logged as warnings and do not fail the migration (the App may suggest
> running one elevated repair). `cleanup_orphan(guid)` archives the old GUID's
> config/snapshot and deletes child-APO records.

### 3.4.1 Private helper functions

| Function | Description |
|----------|-------------|
| `win32_err(err: WIN32_ERROR) -> VxApoError` | Win32 error code → `VxApoError::Registry` (HRESULT form `0x8007_0000 \| (code & 0xFFFF)`) |
| `err_from_hr(hr: HRESULT) -> VxApoError` | HRESULT → `VxApoError::Registry` (keeps the system error text) |
| `win32_ok(err: WIN32_ERROR) -> Result<()>` | Win32 error code → `Result` |
| `utf16_bytes_to_string(buf: &[u8]) -> String` | LE UTF-16 byte stream → `String`, stops at NUL |
| `parse_multi_sz(buf: &[u8]) -> Vec<String>` | LE UTF-16 byte stream → `Vec<String>`, double-NUL terminated |
| `is_not_found(err: WIN32_ERROR) -> bool` | Whether the code is 2 (FILE_NOT_FOUND) or 3 (PATH_NOT_FOUND) |
| `dump_key_recursive(...)` | Recursively exports a key and its subkeys (inside `save_to_file`) |
| `root_display_name(root: HKEY) -> &'static str` | Root key → `.reg` file header name (no longer hardcoded to HKLM) |

> **Removed helpers (do not reference)**: `close_key`, `to_registry_bytes`, `hkey_to_name`
> have 0 hits repo-wide — handle closing is done by `RegKey`'s RAII `Drop`, and value
> writing goes through the `RegKey::write_*` family (which performs `&str` → UTF-16
> internally; there is no separate conversion function).

### 3.4.2 Type definitions

**`RegValue`** (typed registry value):

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum RegValue {
    Sz(String),           // REG_SZ or REG_EXPAND_SZ
    Dword(u32),           // REG_DWORD
    Qword(u64),           // REG_QWORD
    Binary(Vec<u8>),      // REG_BINARY
    MultiSz(Vec<String>), // REG_MULTI_SZ
}
```

**`RegKey`** (registry key handle, RAII):

```rust
#[derive(Debug)]
pub struct RegKey {
    handle: HKEY,
}
// Drop calls RegCloseKey.
// unsafe impl Send / Sync: HKEY is a value-semantics handle; the RAII Drop releases it
// exactly once, and &self methods only pass the handle to Reg* APIs.
```

### 3.4.3 `RegKey` methods

**Open / create**:

| Method | Signature | Description |
|--------|-----------|-------------|
| `open` | `fn open(root: HKEY, sub_key: &str) -> Result<Self>` | Opens a subkey with `KEY_READ` |
| `create` | `fn create(root: HKEY, sub_key: &str) -> Result<Self>` | Creates or opens (`KEY_ALL_ACCESS`); entry point for write/delete on keys we own |
| `open_for_write` | `fn open_for_write(root: HKEY, sub_key: &str) -> Result<Self>` | Opens an existing subkey with only `KEY_SET_VALUE \| KEY_QUERY_VALUE` — the minimal-permission path for MMDevices keys whose ACL grants Administrators only `SetValue, ReadKey` (requesting `KEY_ALL_ACCESS` would fail with 0x80070005 because of the `KEY_CREATE_SUB_KEY` bit) |
| `open_sub_key` | `fn open_sub_key(&self, sub_key: &str) -> Result<Self>` | Delegates to `Self::open(self.handle, sub_key)` |
| `handle` | `fn handle(&self) -> HKEY` | Returns the underlying handle. Callers must not `RegCloseKey` it |

**Read operations**:

| Method | Signature | Description |
|--------|-----------|-------------|
| `read_value` | `fn read_value(&self, name: &str) -> Result<RegValue>` | Type-auto-detected read. Empty string `""` reads the default value. Truncated DWORD/QWORD data is an error, not silently zero |
| `read_sz_value` | `fn read_sz_value(&self, name: &str) -> Result<String>` | Reads REG_SZ |
| `read_sz` | `fn read_sz(&self, name: &str) -> Option<String>` | `Option` wrapper of `read_sz_value` |
| `read_dword_value` | `fn read_dword_value(&self, name: &str) -> Result<u32>` | Reads REG_DWORD |
| `read_binary_value` | `fn read_binary_value(&self, name: &str) -> Result<Vec<u8>>` | Reads REG_BINARY |
| `read_multi_value` | `fn read_multi_value(&self, name: &str) -> Result<Vec<String>>` | Reads REG_MULTI_SZ |
| `value_exists` | `fn value_exists(&self, name: &str) -> Result<bool>` | Whether a value exists under this key. `Ok(false)` only for FILE_NOT_FOUND/PATH_NOT_FOUND; other errors (e.g. ACCESS_DENIED) are `Err` |
| `key_exists_child` | `fn key_exists_child(&self, sub_key: &str) -> Result<bool>` | Whether a subkey exists under this key |
| `enum_sub_keys` | `fn enum_sub_keys(&self) -> Result<Vec<String>>` | Enumerates subkey names. **The underlying capability source for device enumeration** (used by `install/device/info::enumerate_devices` to walk MMDevices subkeys; this module knows nothing about devices). Long names double the buffer up to 32767 |
| `enum_values` | `fn enum_values(&self) -> Result<Vec<String>>` | Enumerates all value names (including the default value `""`) |
| `get_guid_string` | `fn get_guid_string(&self, name: &str) -> Result<String>` | Reads a GUID from REG_BINARY (16-byte LE) or REG_SZ; formatted via `sys/com/prelude::guid_to_string` |

**Write operations** (handle from `create` or `open_for_write`):

| Method | Signature | Description |
|--------|-----------|-------------|
| `write_sz` | `fn write_sz(&self, name: &str, value: &str) -> Result<()>` | Writes REG_SZ (UTF-16LE + NUL) |
| `write_dword` | `fn write_dword(&self, name: &str, value: u32) -> Result<()>` | Writes REG_DWORD |
| `write_qword` | `fn write_qword(&self, name: &str, value: u64) -> Result<()>` | Writes REG_QWORD (round-trips with the read side; needed by `.reg` backup) |
| `write_binary` | `fn write_binary(&self, name: &str, data: &[u8]) -> Result<()>` | Writes REG_BINARY |
| `write_multi_value` | `fn write_multi_value(&self, name: &str, values: &[String]) -> Result<()>` | Writes REG_MULTI_SZ (per-item NUL + double-NUL terminator). Required for the APO processing-mode registration: the `{d3993a3f-...},{PID}` slot's `ProcessingModes` value is REG_MULTI_SZ of processing-mode GUIDs (e.g. `AUDIO_SIGNALPROCESSINGMODE_DEFAULT`); without it the engine does not load the parent-slot APO |

**Delete operations** (handle from `create` or `open_for_write`):

| Method | Signature | Description |
|--------|-----------|-------------|
| `delete_value` | `fn delete_value(&self, name: &str) -> Result<()>` | Deletes a value (idempotent) |
| `delete_sub_key` | `fn delete_sub_key(&self, relative_child: &str) -> Result<()>` | Recursively deletes a child subtree (idempotent). `relative_child` is **relative to the current handle** (e.g. `"Child"`) — `RegDeleteTreeW` resolves the name relative to the handle; passing a full path silently does nothing. Use the free [`delete_tree`](#344-top-level-functions) for absolute-path subtrees |

### 3.4.4 Top-level functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `split_key` | `fn split_key(path: &str) -> Result<(HKEY, &str)>` | Splits a path into root key + subkey. Root names are case-insensitive; supports the 5 standard root keys and abbreviations (HKLM/HKCU/HKCR/HKU/HKCC) |
| `delete_tree` | `fn delete_tree(root: HKEY, sub_key: &str) -> Result<()>` | Recursively deletes a subtree (idempotent, no open handle needed) |
| `is_windows_version_at_least` | `fn is_windows_version_at_least(major: u32, minor: u32, build: u32) -> Result<bool>` | Reads `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion` (`CurrentMajorVersionNumber` / `CurrentMinorVersionNumber` / `CurrentBuildNumber`) |
| `save_to_file` | `fn save_to_file(root: HKEY, sub_key: &str, path: &str) -> Result<()>` | Recursively exports a registry key as a `.reg` file (UTF-16LE with BOM); QWORD as `hex(b)`, MULTI_SZ as `hex(7)`; used for pre-install backup. Per-value read failures are logged and skipped (backup becomes incomplete — callers should treat as a warning) |

> **Removed free functions (do not reference)**: the free `key_exists(root, sub_key)` and
> `value_exists(root, sub_key, name)` have 0 hits repo-wide. Only the
> `RegKey::value_exists` / `RegKey::key_exists_child` methods remain.

**Forbidden**: contains no APO-specific paths or installation business logic.

## 3.5 `sys/audio_defs.rs`

**Responsibility**: Windows audio base definitions — channel-mask bit-flag constants, standard layout constants, and channel-mask fallback / channel-name mapping functions.

**References**:
- Windows SDK `ksmedia.h` `SPEAKER_*` values. windows-rs 0.62.2's `Win32_Media_KernelStreaming`
  feature does **not** export the `SPEAKER_*` constants, so all of them are defined locally
  here with the exact SDK values (no re-export).

**Exports to**: `install/device/format.rs` (`default_channel_mask` fallback), `config/parser.rs`, `object/apo/{process,inner}.rs` (`get_channel_names`, building `DspContext.channel_names` injected into config).

**Public API**:

```rust
// ── 1. Channel mask bit flags (locally defined, SDK values) ────────────────
pub const SPEAKER_FRONT_LEFT: u32 = 0x1;
pub const SPEAKER_FRONT_RIGHT: u32 = 0x2;
pub const SPEAKER_FRONT_CENTER: u32 = 0x4;
pub const SPEAKER_LOW_FREQUENCY: u32 = 0x8;
pub const SPEAKER_BACK_LEFT: u32 = 0x10;
pub const SPEAKER_BACK_RIGHT: u32 = 0x20;
// cfg(test)-only (no production consumers): 0x40, 0x80, 0x100, 0x800..=0x20000
pub const SPEAKER_SIDE_LEFT: u32 = 0x200;
pub const SPEAKER_SIDE_RIGHT: u32 = 0x400;

// ── 2. Standard layout constants ───────────────────────────────────────────
pub const KSAUDIO_SPEAKER_MONO:    u32 = SPEAKER_FRONT_CENTER;
pub const KSAUDIO_SPEAKER_STEREO:  u32 = SPEAKER_FRONT_LEFT | SPEAKER_FRONT_RIGHT;
pub const KSAUDIO_SPEAKER_QUAD:    u32 = SPEAKER_FRONT_LEFT | SPEAKER_FRONT_RIGHT
                                       | SPEAKER_BACK_LEFT | SPEAKER_BACK_RIGHT;
pub const KSAUDIO_SPEAKER_5POINT1: u32 = SPEAKER_FRONT_LEFT | SPEAKER_FRONT_RIGHT
                                       | SPEAKER_FRONT_CENTER | SPEAKER_LOW_FREQUENCY
                                       | SPEAKER_BACK_LEFT | SPEAKER_BACK_RIGHT;
pub const KSAUDIO_SPEAKER_7POINT1: u32 = SPEAKER_FRONT_LEFT | SPEAKER_FRONT_RIGHT
                                       | SPEAKER_FRONT_CENTER | SPEAKER_LOW_FREQUENCY
                                       | SPEAKER_BACK_LEFT | SPEAKER_BACK_RIGHT
                                       | SPEAKER_SIDE_LEFT | SPEAKER_SIDE_RIGHT;

// ── 3. Functions ───────────────────────────────────────────────────────────
/// Standard layout mask for a channel count (fallback).
/// 1→MONO, 2→STEREO, 4→QUAD, 6→5POINT1, 8→7POINT1; anything else returns 0.
pub fn default_channel_mask(channels: u32) -> u32;

/// Mask → short channel-name list (L/R/C/LFE/BL/BR/SL/SR, in standard bit order).
/// Maps strictly by mask bits; unmatched bits are not padded.
/// A mask of 0 returns an empty list (no channel count is available to fall back to).
pub fn get_channel_names(mask: u32) -> Vec<String>;
```

**Channel name → mask bit mapping table**:

| Short name | Bit flag | Standard position |
|------------|----------|-------------------|
| `L` | `SPEAKER_FRONT_LEFT` | 0x1 |
| `R` | `SPEAKER_FRONT_RIGHT` | 0x2 |
| `C` | `SPEAKER_FRONT_CENTER` | 0x4 |
| `LFE` | `SPEAKER_LOW_FREQUENCY` | 0x8 |
| `BL` | `SPEAKER_BACK_LEFT` | 0x10 |
| `BR` | `SPEAKER_BACK_RIGHT` | 0x20 |
| `SL` | `SPEAKER_SIDE_LEFT` | 0x200 |
| `SR` | `SPEAKER_SIDE_RIGHT` | 0x400 |

**Boundary of responsibility**: only static definitions and pure functions with Windows SDK semantics. No VxAPO business logic, no references to `pipeline/`, `config/`, `install/`, `object/`, `utils/`.

**Forbidden**: no business logic, no runtime-state awareness.

> Note: `sys/known_folder.rs` was removed from the actual codebase. Per-device config paths are handled by object/CLI using the fixed `C:\ProgramData\VxAPO\{GUID}` root, not a Documents-folder helper.
