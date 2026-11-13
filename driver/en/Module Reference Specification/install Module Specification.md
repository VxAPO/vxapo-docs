# install Module Specification

**Boundary**: Does not know `pipeline/` or `config/`. Responsible only for device APO install/uninstall and device query.

**Allowed dependencies**: `sys/`, `utils/`, `object/vx_reg_props.rs`.

**Forbidden dependencies**: `pipeline/`, `config/`.

## Module tree

```
install/
├── device.rs              # module entry
├── audiodg.rs             # DisableProtectedAudioDG check/fix + audio service control
├── device/
│   ├── endpoint.rs        # endpoint state/name query (read-only)
│   ├── format.rs          # WAVEFORMATEX parsing + channel mask fallback (read-only)
│   ├── slots.rs           # slots module entry
│   │   ├── types.rs       # ApoSlot / InstallMode / SlotValue + constants
│   │   ├── read.rs        # read_slot_value / read_all_slots / detect_install_mode
│   │   └── child.rs       # child-APO info area + original-GUID fallback
│   ├── sysfx.rs           # CAPX MSFX template locate/claim/restore/heal (v9.0)
│   ├── identity.rs        # endpoint stable identity (v9.24)
│   ├── stale.rs           # stale-GUID detect/migrate/cleanup entry (detect/migrate/matching/acl)
│   └── info.rs            # combined query layer + single device-enumeration entry (read-only)
└── selector.rs            # module entry (pub mod operation;)
    └── operation.rs       # operation entry (execute / helpers / capx submodules)
        └── execute.rs     # install_endpoint / uninstall_endpoint / write_install_config / Transaction
```

> The historical flat files `slots.rs`, `stale.rs` and `selector/operation.rs` are module
> entries over real submodules (`slots/{types,read,child}`,
> `stale/{detect,migrate,matching,acl}`, `operation/{execute,helpers,capx}`). Public APIs
> are re-exported from the entries.

## Reference dependency table

> v9.17 (single source of truth): this table is no longer maintained independently — the
> main specification `Module Reference Specification (Without Detailed Modules).md`,
> section 11 "Reference dependency table", is the only baseline; the per-file allowed and
> forbidden dependencies of install are listed line by line there.

## 5.1 `install/device/endpoint.rs`

**Responsibility**: query the device ID, friendly name, and connection state of a Windows audio endpoint. Read-only; never modifies system state.

**References**:
- `crate::sys::registry::RegKey`
- `crate::utils::vx_error::VxApoError`

**Exports to**: `install/device/info.rs`.

**Public API**:

```rust
/// Windows audio endpoint state (IMMDevice::GetState values).
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum EndpointState {
    Active,              // DEVICE_STATE_ACTIVE = 1
    Disabled,            // DEVICE_STATE_DISABLED = 2
    NotPresent,          // DEVICE_STATE_NOTPRESENT = 4
    Unknown(u32),
}

/// Audio stream direction.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Flow { Render, Capture }

/// Audio endpoint info (queried from the registry, read-only).
#[derive(Debug, Clone)]
pub struct EndpointInfo {
    pub device_id: String,         // IMMDevice's PKEY_DeviceInstanceId
    pub friendly_name: String,     // PKEY_DeviceInterface_FriendlyName
    pub state: EndpointState,
    pub flow: Flow,                // default Render; the enumerator back-fills per root
    pub endpoint_guid: String,
}

/// Queries endpoint info from an opened endpoint registry key.
///
/// Endpoint keys live at:
/// HKLM\...\MMDevices\Audio\Render\{guid} or ...\Capture\{guid}
///
/// Returns Ok(Some(info)) / Ok(None) (properties missing) / Err (I/O error).
pub fn query_endpoint(endpoint_key: &RegKey) -> Result<Option<EndpointInfo>>;
```

> **Removed**: the old `is_endpoint_active(endpoint_key)` free function no longer exists
> (0 hits). Active-state checks derive from `EndpointState` / `DeviceInfo` predicates.

## 5.2 `install/device/format.rs`

**Responsibility**: parse WAVEFORMATEX / WAVEFORMATEXTENSIBLE from registry binary values; extract channel count, sample rate, bit depth, and channel mask. Read-only.

**References**:
- `crate::sys::registry::RegKey`
- `crate::utils::vx_error::VxApoError`
- `crate::sys::audio_defs::default_channel_mask` (fallback)

**Exports to**: `install/device/info.rs`.

**Channel mask fallback chain** (Note 27):
1. `WAVEFORMATEXTENSIBLE`'s `dwChannelMask` (bytes[20..24])
2. Registry `PKEY_AudioEndpoint_PhysicalSpeakers` (DWORD,
   `{1da5d803-d492-4edd-8c23-e0c0ffee7f0e},3`, read in `install/device/info.rs`)
3. `sys::audio_defs::default_channel_mask` (standard layout mapping, 1/2/4/6/8 channels)

**Public API**:

```rust
/// Parsed audio format info.
#[derive(Debug, Clone)]
pub struct AudioFormat {
    pub format_tag: u16,          // WAVE_FORMAT_PCM = 1, EXTENSIBLE = 0xFFFE
    pub channels: u16,
    pub sample_rate: u32,
    pub bits_per_sample: u16,
    pub channel_mask: u32,        // guaranteed non-zero after the fallback chain
    pub is_extensible: bool,
}

/// Parses audio format from raw bytes (pure function, no I/O, unit-testable).
///
/// `channel_mask_override`: external fallback channel mask (from a registry DWORD).
pub fn parse_audio_format(bytes: &[u8], channel_mask_override: Option<u32>) -> Option<AudioFormat>;

/// Resolves the channel mask by priority (fallback chain).
pub fn resolve_channel_mask(extensible_mask: u32, registry_mask: Option<u32>, channels: u16) -> u32;

/// Reads and parses the audio format from a registry key.
///
/// Tries to read the `format_value_name` binary value and parse it.
/// Returns Ok(None) when the value does not exist.
pub fn read_audio_format(
    key: &RegKey,
    format_value_name: &str,
    channel_mask_value_name: Option<&str>,
) -> Result<Option<AudioFormat>>;
```

## 5.3 `install/device/slots.rs`

**Responsibility**: manage the 5 APO GUID slots under FxProperties; provide install-mode definitions, raw APO GUID fallback queries, and reading of the **VxAPO-independent install info area (child APO backup)**. Read-only.

**References**:
- `crate::sys::registry::RegKey`
- `crate::utils::guid::{guid_from_bytes, is_zero_guid, parse_guid_string}`
- `crate::sys::com::prelude::guid_to_string` (standard GUID string formatting, safe narrowing boundary)

**Exports to**: `install/device/info.rs`, `install/selector/operation.rs`, `object/apo.rs` (v8.4: runtime child-APO GUID read — endpoint GUID → the VxAPO child-APO registry path `{PreMixChild|PostMixChild}`).

**Child APO install info area (v8.4, VxAPO-independent path)**:

```
HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}
    PreMixChild                     = {former PreMix APO GUID} (written when useOriginalAPOPreMix)
    PostMixChild                    = {former PostMix APO GUID} (written when useOriginalAPOPostMix)
    AllowSilentBufferModification   = "true"/"false"
    DisableAutomaticAdjustment      = "true" (written when auto_adjust = false)
    Version                         = "2"
```

> **Path isolation (v8.4, directive)**: **never** read or write EAPO's
> `HKLM\SOFTWARE\EqualizerAPO\Child APOs` (EAPO `childApoPath`, RegistryHelper.h 33 +
> DeviceAPOInfo.cpp 43) — VxAPO uses its own `HKLM\SOFTWARE\VxAPO` root. Writing the EAPO
> path would pollute EAPO's install info area (EAPO would read the PreMixChild/PostMixChild
> written by VxAPO and get overwritten or semantically confused when EAPO is reinstalled or
> uninstalled). What is aligned is EAPO's **mechanism** (an independent registry path holding
> the child APO backup), not the path itself.

**The 5 slots** (Note 25):

| Index | Name | Role |
|-------|------|------|
| 0 | LFX | Legacy PreMix (Win8.1+) |
| 1 | GFX | Legacy PostMix (Win8.1+) |
| 2 | SFX | Side-effect PreMix (Win10+) |
| 3 | MFX | Mixed-effect PostMix (Win11 Bluetooth) |
| 4 | EFX | Endpoint-effect PostMix (default) |

**The 3 install modes** (Note 26):

| Mode | PreMix slot | PostMix slot | Scenario |
|------|-------------|--------------|----------|
| LfxGfx | LFX(0) | GFX(1) | Win8.1+ Legacy |
| SfxMfx | SFX(2) | MFX(3) | Win11 Bluetooth |
| SfxEfx | SFX(2) | EFX(4) | **Default** |

**Public API**:

```rust
pub const FX_PROPERTIES_KEY: &str = "FxProperties";
pub const INSTALL_VERSION: &str = "2";
pub const INSTALL_VERSION_LEGACY: &str = "1";

/// APO slot index.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
#[repr(u8)]
pub enum ApoSlot { Lfx = 0, Gfx = 1, Sfx = 2, Mfx = 3, Efx = 4 }

impl ApoSlot {
    pub const ALL: [ApoSlot; 5];
    pub fn index(self) -> u8;
    /// Windows' real registry slot property ID (PID) — **v8.9 measured
    /// (2026-08-04 reg query)**: the slot PIDs under `{d04e05a6-...}` are **0/3/5/6/7**
    /// (not contiguous 0-4): LFX=0 / GFX=3 / SFX=5 / MFX=6 / EFX=7.
    pub fn registry_pid(self) -> u8;
    /// Registry value name: `{d04e05a6-...},{registry_pid}`
    pub fn value_name(self) -> String;
    pub fn is_premix(self) -> bool;
    pub fn is_postmix(self) -> bool;
}

/// Install mode.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum InstallMode { LfxGfx, SfxMfx, SfxEfx }

impl InstallMode {
    pub fn premix_slot(self) -> ApoSlot;
    pub fn postmix_slot(self) -> ApoSlot;
    pub fn default_mode() -> InstallMode;  // SfxEfx
}

/// Slot value state (three-state).
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SlotValue {
    NoKey,                       // FxProperties key does not exist
    NoValue,                     // value empty or occupied by another APO
    Guid(windows::core::GUID),   // a concrete APO CLSID
}

impl SlotValue {
    pub fn is_guid(&self) -> bool;
    pub fn is_empty(&self) -> bool;
    pub fn as_guid(&self) -> Option<windows::core::GUID>;
}

/// Reads one slot from FxProperties (**v8.9 dual-format compat + all-zero normalization**).
///
/// Windows slot values exist both as REG_SZ (EAPO and other third parties write GUID
/// strings, measured) and REG_BINARY (16-byte LE, some implementations / old VxAPO) —
/// auto-detected by the real type. **The all-zero GUID (`{00000000-...}`) is Windows'
/// "no APO" placeholder and normalizes to NoValue** (otherwise auto-detection would
/// mistake zeroed slots for occupied).
pub fn read_slot_value(fx_key: &RegKey, slot: ApoSlot) -> SlotValue;

/// Reads all 5 slots from an endpoint root key.
/// When FxProperties does not exist, all slots return NoKey.
pub fn read_all_slots(endpoint_key: &RegKey) -> [SlotValue; 5];

/// Gets the original PreMix APO GUID (with fallback, Note 46).
///
/// Fallback: when the primary slot is NoValue, falls back to the other mode's PreMix slot.
/// Returns an empty string on NoKey or when no fallback exists.
pub fn get_original_pre_mix(slots: &[SlotValue; 5], mode: InstallMode) -> String;

/// Gets the original PostMix APO GUID (with fallback, Note 46).
///
/// Fallback order when the primary slot is NoValue:
/// - SfxEfx: EFX → MFX → GFX
/// - SfxMfx: MFX → EFX → GFX
/// - LfxGfx: GFX → EFX → MFX
pub fn get_original_post_mix(slots: &[SlotValue; 5], mode: InstallMode) -> String;

/// **EAPO three-tier install-mode auto-detection** (v8.9 — ported from EAPO load()
/// 396-413, C41-C44). Pure logic API — the caller (driver info layer / APP) prepares the
/// inputs; the driver only decides:
///
/// | Priority | Condition | Mode |
/// |----------|-----------|------|
/// | 0 | Win < 8.1 (no probing) | LfxGfx (Legacy initial default) |
/// | 1 | Win8.1+ and FxProperties has **only LFX/GFX values, SFX/MFX/EFX all empty** | LfxGfx (driver supports Legacy only) |
/// | 2 | endpoint instance ID starts with `BTHENUM` / `BTHLE` (Bluetooth audio) | SfxMfx (Win11 Bluetooth combo; EFX invalid) |
/// | 3 | otherwise (modern driver default) | SfxEfx |
///
/// # Parameters
/// - `is_windows_8_1_or_newer`: OS version check
///   (`registry::is_windows_version_at_least(6,3,9600)`).
/// - `slots`: 5 slot values (from the endpoint FxProperties, `read_all_slots`).
/// - `has_bluetooth`: endpoint instance ID starts with `BTHENUM`/`BTHLE`.
pub fn detect_install_mode(
    is_windows_8_1_or_newer: bool,
    slots: &[SlotValue; 5],
    has_bluetooth: bool,
) -> InstallMode;
```

**Install info area existence check (v8.5, the sole basis for full/non-full decisions)**:

```rust
/// VxAPO-independent install info area key path (v8.5).
pub const CHILD_APO_PATH_ROOT: &str = "HKLM\SOFTWARE\VxAPO\Child APOs";

/// Child APO slot kind (PreMixChild / PostMixChild).
pub enum ChildApoKind { /* ... */ }

/// Whether a device has a VxAPO install info area (the only basis for the
/// full/non-full decision, intent section 7 v8.5).
///
/// - Absent → initial install / install after full uninstall → `install_endpoint` takes
///   the **full backup path**;
/// - Present → reinstall / re-takeover → **non-full path** (slot overwrite or keep old
///   child APO records).
///
/// Path privacy guarantee: written only by install_endpoint, deleted only by
/// uninstall_endpoint (uninstall always deletes the whole key); third-party APO software
/// never writes it (each product uses its own private path) — existence is a sufficient test.
pub fn child_apo_key_exists(device_guid: &str) -> bool;

/// Reads a stored child APO GUID (v8.4 runtime path used by object/apo.rs).
pub fn read_child_apo_guid(device_guid: &str, kind: ChildApoKind) -> Option<GUID>;
```

## 5.4 `install/device/info.rs`

**Responsibility**: combines the `endpoint`, `slots`, and `format` submodules into high-level queries. **The single device-enumeration entry** (`enumerate_devices`). Read-only.

**References**:
- `install/device/endpoint`, `install/device/slots`, `install/device/format`
- `crate::sys::registry::RegKey`
- `crate::object::vx_reg_props::{CLSID_VXAPO_PRE_MIX, CLSID_VXAPO_POST_MIX}`
- `crate::utils::vx_error::VxApoError`

**Exports to**: `install/selector/operation.rs`, `object/apo.rs`, CLI (`commands.rs`).

**Public API**:

```rust
/// Combined device info (query result, read-only).
#[derive(Debug, Clone)]
pub struct DeviceInfo {
    pub endpoint: Option<EndpointInfo>,   // None when the endpoint key vanished mid-scan
    pub install_mode: InstallMode,
    pub slots: [SlotValue; 5],
    pub format: Option<AudioFormat>,
    pub installed_version: String,        // "2", "1", or "" (not installed)
}

impl DeviceInfo {
    /// Whether VxAPO is installed: version non-empty and the PreMix or PostMix slot
    /// holds a VxAPO CLSID.
    pub fn is_installed(&self) -> bool;
    /// Whether upgradable (installed and version < INSTALL_VERSION).
    pub fn can_be_upgraded(&self) -> bool;
    /// Whether this is an experimental install (LfxGfx mode).
    pub fn is_experimental(&self) -> bool;
    /// Whether this is a Legacy install (LfxGfx alias used by CLI/App).
    pub fn is_legacy(&self) -> bool;
    /// Whether audio enhancements are disabled.
    pub fn is_enhancements_disabled(&self, endpoint_key: &RegKey) -> bool;
    /// Whether the device is disabled (DEVICE_STATE_DISABLED, E3.2/EAPO-inspired).
    pub fn is_disabled(&self) -> bool;
    /// Whether the device is unplugged (DEVICE_STATE_NOTPRESENT, E3.2/EAPO-inspired).
    pub fn is_unplugged(&self) -> bool;
    /// Whether there are unapplied changes.
    pub fn has_changes(&self) -> bool;
}

/// Queries combined device info (endpoint + slots + format).
pub fn query_device_info(endpoint_key: &RegKey) -> Result<Option<DeviceInfo>>;

/// Device enumeration — **returns all legal install containers** (walks every endpoint
/// key under MMDevices\Audio\Render and Capture, combining endpoint/slots/format).
/// Only guarantees "the container is legal"; it does not decide the default device
/// (user-layer COM duty, see below).
pub fn enumerate_devices() -> Result<Vec<DeviceInfo>>;

/// Locates the registry path of an endpoint GUID (Render first, then Capture).
/// Used by runtime self-healing (object/apo/init.rs Initialize) and the install/migration
/// flows. Invalid GUID → Err(internal); not found → Err(device_not_found).
pub fn find_endpoint_path(device_guid: &str) -> Result<String>;
```

> **Device physical state predicates (E3.2)**: `is_disabled()`/`is_unplugged()` derive from
> `DeviceInfo.endpoint.state` (`EndpointState::{Disabled, NotPresent}`) with zero extra
> I/O; used to further filter legal install containers.

> **Default-device boundary (E3.1)**: the driver layer does **not** decide "is this the
> default device" — `enumerate_devices()` only returns legal install containers under
> MMDevices. Default/effective device display is a **user-layer (CLI/APP) duty via COM
> `IMMDeviceEnumerator::GetDefaultAudioEndpoint`**, correlated with the driver enumeration
> by `device_id`/`endpoint_guid`. The driver keeps a registry-only stack (no COM enumeration).

**Mode detection logic (v9.18 revision — only VxAPO CLSID pairs count, measured 2026-08-04)**:
1. LFX=VxAPO_PRE **and** GFX=VxAPO_POST → LfxGfx (**a non-VxAPO-occupied slot does not count** —
   EDIFIER measurement: the old implementation misjudged SfxMfx from "any GUID in a slot",
   while actually SFX/EFX were occupied by EAPO and MFX by the system — the correct answer
   was "VxAPO not installed")
2. SFX=VxAPO_PRE **and** MFX=VxAPO_POST → SfxMfx
3. SFX=VxAPO_PRE **and** EFX=VxAPO_POST → SfxEfx
4. No pair → `default_mode()` (SfxEfx, meaning VxAPO not installed)

**EAPO three-tier auto-detection (v8.9; CLI default `--mode` / APP entry)**:

```rust
/// Combines OS version + 5 slots + Bluetooth instance ID and hands them to
/// slots::detect_install_mode. Bluetooth is detected from the endpoint `Properties`
/// subkey's device instance ID (PKEY_DeviceInstanceId, `{b3f8fa53-...},2`) starting with
/// BTHENUM/BTHLE (case-insensitive).
pub fn detect_mode_for_device(endpoint_key: &RegKey) -> InstallMode;

/// Auto-detects by endpoint GUID (Render first / Capture fallback to locate the root key).
/// Location failure falls back to `detect_install_mode(true, all-empty, false)` → SfxEfx.
pub fn detect_mode_for_guid(device_guid: &str) -> InstallMode;
```

> **Removed**: the old `BLUETOOTH_CONTAINER_VALUE` constant
> (`{b3f8fa53-0004-438e-9003-51a46e139bfc},41`) no longer exists in the code (0 hits);
> Bluetooth detection goes through the device instance ID string, not a container-ID value.

## 5.5 `install/selector.rs` (module entry)

**Responsibility**: entry aggregation of the `selector` submodule — currently only `pub mod operation;`. The interactive selection flow is owned by the CLI (v9.19 deleted the `select` dead code).

```rust
pub mod operation;
```

## 5.5.1 ~~`install/selector/select.rs`~~ (deleted)

> **No longer exists since v9.19**: device selection interaction moved up to CLI/App; the
> driver no longer contains `select.rs`. Device enumeration still goes through
> `install/device/info::enumerate_devices`.

## 5.5.2 `install/selector/operation.rs`

**Responsibility**: device APO install/uninstall execution + transaction rollback (the former `install.rs` + `rollback.rs`, merged here; real implementation in `operation/execute.rs`).

**References**:
- `install/device/slots` (`ApoSlot` / `InstallMode` / `SlotValue` / `read_all_slots` / `read_slot_value` / `FX_PROPERTIES_KEY` / `INSTALL_VERSION`)
- `install/device/format`
- `install/device/sysfx` (CAPX takeover/restore, v9.0)
- `crate::sys::registry::{RegKey, RegValue}` (backup delegated to `save_to_file`)
- `crate::object::vx_reg_props::{CLSID_VXAPO_PRE_MIX, CLSID_VXAPO_POST_MIX}`
- `crate::object::dll_exports::register_apo_with_path` (refreshes global APO registration before install, completing the AudioEngine keys)
- `crate::sys::com::prelude::guid_to_string` (GUID formatting)
- `crate::utils::vx_error::{Result, VxApoError}`

**Exports to**: CLI (`vxapo-cli/src/commands.rs`), `object/apo.rs` (as needed).

**Public API**:

```rust
/// Install parameters.
pub struct InstallConfig {
    pub install_premix: bool,             // install the PreMix APO
    pub install_postmix: bool,            // install the PostMix APO
    pub install_mode: InstallMode,        // decides which two slots are used
    pub use_original_apo_premix: bool,    // keep the original PreMix APO as child APO
    pub use_original_apo_postmix: bool,   // keep the original PostMix APO as child APO
    pub allow_silent_buffer: bool,        // allow the silent-buffer fast path (Note 11)
    pub auto_adjust: bool,                // E3.3/EAPO InstallState autoAdjust-inspired;
                                          // independent of allow_silent_buffer; written as
                                          // the registry autoAdjust in install Step 4
}

impl InstallConfig {
    pub fn default_config() -> Self;  // SfxEfx, both directions, allow_silent = true, auto_adjust = false
}

/// Pure registry-write portion (7 steps + sysfx takeover, with transaction rollback).
/// Does NOT include the tail endpoint restart / AudioSrv ensure / self-check.
/// CLI install --verify calls this directly, then does the service restart + pipe
/// verification itself (avoids duplicating pnputil endpoint restart + full service restart).
pub fn write_install_config(
    device_guid: &str,
    device_name: &str,
    connection_name: &str,
    config: &InstallConfig,
) -> Result<()>;

pub fn install_endpoint(
    device_guid: &str,
    device_name: &str,
    connection_name: &str,
    config: &InstallConfig,
    verify: bool,   // E3.4 (v8.3): true = run an installation self-check at the end
) -> Result<()>;

pub fn uninstall_endpoint(device_guid: &str) -> Result<()>;
```

> **`auto_adjust` (E3.3)**: an independent `InstallConfig` field; the `autoAdjust` written
> to the registry in Step 4 reads this field instead of hard-coding. Default `false`
> (EAPO defaults to true, but VxAPO has no auto-correction implementation, so the
> conservative default is off).

> **Install self-check (E3.4, EAPO testAPOInstallation-inspired; v8.3 corrected description;
> implementation verified against code)**: with `verify = true`, `install_endpoint` runs the
> self-check after all steps commit: `CoInitializeEx(None, COINIT_MULTITHREADED)`
> (S_OK/S_FALSE both acceptable), then `CoCreateInstance` for `CLSID_VXAPO_PRE_MIX` and
> `CLSID_VXAPO_POST_MIX` (`CLSCTX_INPROC_SERVER`) — i.e. **verify the DLL can be
> instantiated via COM**, not an `IAudioClient` pipe activation (EAPO's own
> `testAPOInstallation` activates IAudioClient; VxAPO's check is the CoCreateInstance form).
> On failure: **no automatic rollback** (the registry is already written and the failure may
> be transient); returns `Err` with a precise message for the CLI/caller to present. The
> user can then explicitly `reinstall`/`uninstall`.

**Note 47 install flow** (7 steps; v8.5 added the full/non-full decision; v8.9 implementation alignment — minimal write permissions / independent info area / self-preserve / REG_SZ / EAPO alignment):

**0. Full-install decision (v8.5)**: `install_endpoint` starts by checking whether
`HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}` exists (`install/device/slots` provides the check):
- **Absent → full backup path** (initial install / install after full uninstall — the key
  was deleted at uninstall, so equivalent): full 5-slot snapshot (including NOKEY/NOVALUE
  placeholders) + PreMix/PostMixChild per useOriginal + AllowSilent/DisableAutoAdjust/Version;
- **Present → non-full path** (reinstall / re-takeover): on takeover (the flow detects the
  required install slots are not VxAPO CLSIDs) only **overwrite** `PreMixChild`/`PostMixChild`
  with the currently occupied slot values (latest former); without takeover keep the old
  child APO records; and **re-snapshot fully** to keep the fallback baseline current.
- **v8.9 addition: overwritten slot names/original values are backed up unconditionally** —
  regardless of full/non-full, before overwriting the PreMix/PostMix slots, back up
  "slot name + original value" into the info area (`PreMixSlot`/`PostMixSlot` +
  `PreMixSlotValue`/`PostMixSlotValue`); uninstall uses these to restore the overwritten
  third-party APO (EAPO etc.), otherwise after snapshot restore the slots would be NoValue
  (measured 2026-08-04: VxAPO made EAPO a child APO, then another product overwrote the
  parent slot → uninstall must restore EAPO).

**0a. Preconditions (v8.10, driver full-flow convergence)**:
- `DisableProtectedAudioDG=1` (`install/audiodg::disable`) — audiodg checks this before
  creating an APO instance; missing value → refuse to load;
- `refresh_global_registration()` — reads the existing CLSID→DLL binding and calls
  `object/dll_exports::register_apo_with_path` to complete/overwrite the
  `AudioEngine\AudioProcessingObjects` fields (old registrations missing MaxInstances etc.
  caused the parent slot not to load).

1. Create the **VxAPO-independent install info area** key
   `HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}` (v8.4 path-isolation fix: align with the
   **mechanism** of EAPO's `HKLM\SOFTWARE\EqualizerAPO\Child APOs` (childApoPath,
   RegistryHelper.h 33 + DeviceAPOInfo.cpp 43) but under VxAPO's own `SOFTWARE\VxAPO` root;
   **never write EAPO's install info area** — polluting it makes EAPO read VxAPO-written
   PreMixChild/PostMixChild and get overwritten/semantically confused on EAPO reinstall/uninstall)
2. Open/create FxProperties (v8.9 minimal write permission):
   - **Exists → `RegKey::open_for_write` (KEY_SET_VALUE + KEY_QUERY_VALUE)** — MMDevices
     endpoint key ACL grants Administrators only SetValue/ReadKey (no CreateSubKey);
     requesting KEY_ALL_ACCESS is rejected (0x80070005). The is_new decision reads the
     `version` value's existence.
   - Absent → `RegKey::create` (SAM_ALL; a newly created key has full permission) and
     record a delete-rollback in the transaction.
3. If it existed, back up FxProperties to a `.reg` file (`sys::registry::save_to_file`,
   best-effort) + record slot rollback in the transaction.
4. Write the child APO config into **`HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}`**:
   `PreMixChild` / `PostMixChild` (**v8.9 self-preserve filter: a slot value equal to
   VxAPO's own CLSID is not preserved** — otherwise a reinstall would treat VxAPO itself as
   the child APO; snapshot diff measured childPreMix=41C34613 self-occupied);
   otherwise per useOriginalAPOPreMix/PostMix (aligned with EAPO DeviceAPOInfo.cpp 558-563);
   on the non-full path with takeover, overwrite with the currently occupied slot values /
   `PreMixSlot` + `PreMixSlotValue` etc. backup values (**written unconditionally** —
   uninstall needs them to write third-party APOs back; child GUIDs are written only when
   preserved, original-value backups are always written) / `AllowSilentBufferModification` /
   `DisableAutomaticAdjustment` (autoAdjust) / `Version`.
5. Write the APO GUIDs per mode (v8.9 EAPO mutual-exclusion semantics + forced REG_SZ):
   - **Slots are written as REG_SZ (GUID string)** — the EAPO ecosystem (RegistryHelper.h)
     and the Windows audio enumerator expect REG_SZ; writing REG_BINARY reports "wrong
     type" and breaks EAPO's device enumeration (measured 2026-08-04).
   - **Delete old slots not in the current mode, keep EAPO's "untouched" slots**: LfxGfx
     deletes SFX/MFX/EFX (Legacy exclusive); SfxMfx deletes LFX/GFX **keeping EFX**; SfxEfx
     deletes LFX/GFX **keeping MFX** (the old implementation deleted every non-current-mode
     slot → SfxEfx wrongly deleted MFX; in Bluetooth scenarios MFX conflicts with EFX).
6. Write the default processing-mode GUID (`KSDATAFORMAT_SUBTYPE_DEFAULT_PROCESSMODE` =
   AUDIO_SIGNALPROCESSINGMODE_DEFAULT, REG_SZ).
7. Delete DisableEnhancements.
   - **v9.0 EAPO alignment**: also delete `{1da5d803-d492-4edd-8c23-e0c0ffee7f0e},5`
     (PKEY_AudioEndpoint_Disable_SysFx), forcing the enhancement chain on.
   - **Capture special case (v8.9, EAPO DeviceAPOInfo.cpp 583/607/632 alignment)**: capture
     endpoints (path contains Capture) install only PreMix, no PostMix (VxAPO does no capture
     enhancement); `PreMixChild` backup kept, PostMixChild forced None.
8. **Take over CAPX "device default effects"** (v9.0, `install/device/sysfx.rs`):
   - Locate `HKLM\SYSTEM\...\DeviceClasses\...\Device Parameters\MSFX\N` by device
     instance ID + JackSubType;
   - Microsoft StreamEffectClsid (`,5`) → VxAPO PreMix; delete ModeEffectClsid (`,6`) to
     avoid Microsoft MFX double-processing with VxAPO PostMix;
   - **Runtime self-healing (v9.4)**: the install-time takeover only covers templates that
     existed then; Windows re-enumeration/reboot may re-inject Microsoft CAPX into
     `MSFX\N`. On DLL `Initialize`, endpoints with VxAPO installed call
     `ensure_takeover_for_endpoint` again — touches only Microsoft CLSIDs, never
     third-party ones, idempotent (zero writes when no Microsoft entries), failures only
     downgrade to a log; also deletes DisableEnhancements / Disable_SysFx to force the
     enhancement chain on.
   - Leftover Microsoft `,6` in the endpoint FxProperties is deleted too; original values
     are written into the VxAPO install info area (`SysFxBackups`).
   - Uninstall restores the Microsoft default effects (backups first, CAPX defaults as fallback).
9. **Restart AudioSrv** (`install/audiodg` service helpers, EAPO install alignment) — makes
   the new slot topology/registration take effect; the CLI does not restart again itself.

The whole install runs under a `Transaction` (reverse-order rollback on Drop); any step
failure rolls back automatically.

**Uninstall flow** (v8.5 added key deletion; v8.9 implementation alignment — only delete VxAPO CLSIDs / restore only when empty / never override a taker):

**Uninstall semantics (clarified + measured 2026-08-04)**: uninstall only "the parts that
certainly belong to VxAPO", never touching other APOs:
0. **Stop AudioSrv first** (`install/audiodg::stop_audio_service`) — the purpose is **not**
   unlocking: writing/deleting `FxProperties` values only needs a handle with
   `KEY_SET_VALUE` (`open_for_write`); measured 2026-09-16: deleting slot values succeeds
   even during audio playback with the DLL loaded and audiodg holding the endpoint. The real
   reasons: ① **release the `vxapo_driver.dll` module image** (if audiodg does not exit, the
   immediately following reinstall/DLL swap fails because the file is in use; NSIS
   `installer-hooks.nsh` stops the service before install/uninstall for the same reason)
   ② make the endpoint restart (`pnputil /restart-device`) at the end of the flow take
   effect immediately (the engine caches the endpoint APO chain).
1. Locate the endpoint FxProperties (absent → treat as not installed, return success);
   **open with `open_for_write`** (deleting values needs write permission; SAM_ALL is
   over-privileged and rejected on MMDevices endpoint keys with 0x80070005).
2. **Delete only VxAPO CLSIDs**: walk the 5 slots and delete a slot value only when it
   equals the VxAPO PRE/POST CLSID (other APOs' slots untouched — EAPO can re-take them on
   reinstall). **Note**: do not use `read_all_slots` (it expects the endpoint root key and
   opens FxProperties itself); the current handle is already the FxProperties key, so use
   `read_slot_value` directly on fx_key (measured 2026-08-04: leftover VxAPO CLSIDs after
   uninstall came from exactly this mistake).
3. **Restore backups only into emptied slots**: after a VxAPO slot is deleted
   (NoValue/NoKey), write back the install-time backup (`PreMixSlotValue`/`PostMixSlotValue`);
   if the slot has been taken over by another APO (Guid present) → **respect the taker, do
   not overwrite**. Uninstall is not a blind snapshot restore: EAPO and other former APOs
   come back only through this slot-empty rule (or an explicit `snapshot restore`).
4. **Restore the CAPX MSFX templates** (v9.0): before deleting the info area, restore the
   Microsoft StreamEffect/ModeEffect from `SysFxBackups`; without backups, fall back to the
   CAPX default CLSIDs.
5. Delete the whole info area key `HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}` (v8.5
   product intent: uninstall always deletes the key → the next install takes the "full path").
6. Clean up legacy VxAPO config values on FxProperties (childGuid/allowSilentBuffer/
   autoAdjust/version, best-effort) + DisableEnhancements.
7. **Restart AudioSrv** (service helpers) to restore output.

**Transaction rollback** (the former rollback.rs duty, merged here):

```rust
pub enum RollbackAction {
    /// Deletes a whole subtree (idempotent) — e.g. the FxProperties key we created.
    DeleteKey { root: HKEY, path: String },
    /// Re-creates a key and restores one backed-up REG_SZ value.
    RestoreValue { key_path: String, name: String, backup: String },
}

struct Transaction { /* actions + committed flag */ }
// Drop: if not committed, executes all recorded actions in reverse order.
// DeleteKey uses delete_tree (idempotent, full path relative to root — the relative-handle
// delete_sub_key silently no-ops on a full path, review #7); RestoreValue re-creates the
// key and writes the backed-up value back.
```

**Forbidden**: does not depend on `pipeline/`, `config/`.

## 5.6 `install/audiodg.rs`

**Responsibility**: DisableProtectedAudioDG check/fix + Windows audio service control.

**References**:
- `crate::sys::registry::{RegKey, RegValue}`

**Exports to**: `object/apo.rs`, `install/selector/operation.rs`, `install/device/stale.rs`.

**Public API** (visibility verified against code):

```rust
// -- crate-internal (pub(crate)) ------------------------------------------
/// Whether protection is disabled: DisableProtectedAudioDG exists and == 1.
pub(crate) fn is_disabled() -> Result<bool>;

/// Sets DisableProtectedAudioDG = 1 (disable protection, allow third-party APOs).
/// Requires administrator rights (writes HKLM). Uses RegKey::create + write_dword.
pub(crate) fn disable() -> Result<()>;

/// Checks and ensures loading is allowed; fixes when not.
/// Called by object/apo.rs LockForProcess.
pub(crate) fn ensure_can_load() -> Result<()>;

/// Targeted restart of a single endpoint device (render / capture), no full service
/// restart. pnputil /restart-device semantics.
pub(crate) fn restart_endpoint_device(device_guid: &str, is_capture: bool) -> Result<()>;

// -- public ----------------------------------------------------------------
/// Stops the Windows audio service (stop only, no restart; uninstall precondition).
pub fn stop_audio_service() -> Result<()>;

/// Ensures the audio service is running (starts it only when stopped; idempotent).
pub fn ensure_audio_service_running() -> Result<()>;

/// Event-driven wait for all audiodg.exe processes to exit (Toolhelp snapshot for PIDs
/// then OpenProcess(SYNCHRONIZE) then WaitForSingleObject; returns as soon as a process
/// exits; at most two scan rounds). Purpose: after stopping the service / taskkill,
/// confirm the module image is released (if audiodg does not exit, vxapo_driver.dll stays
/// locked and a following reinstall/DLL swap would fail to overwrite). Slot write/delete
/// does not depend on this (a KEY_SET_VALUE handle suffices). Returns true = no audiodg
/// left; false = timeout with leftovers (callers continue best-effort).
pub fn wait_for_audiodg_exit(timeout_ms: u32) -> bool;

/// Dependency-aware stop of the audio service (stops AudioEndpointBuilder and other
/// dependents first, polls until STOPPED).
pub fn stop_audio_service_with_dependents(stop_timeout_secs: u32) -> Result<()>;

/// Dependency-aware start of the audio service (polls until RUNNING).
pub fn start_audio_service_with_dependents(start_timeout_secs: u32) -> Result<()>;

/// Stop + start with waits (the --verify install flow uses this).
pub fn restart_audio_service_wait(stop_timeout_secs: u32, start_timeout_secs: u32) -> Result<()>;
```

> **Removed (do not reference)**: `is_third_party_allowed()`, `restore()` and
> `restart_audio_service()` no longer exist — use `is_disabled()`, delete-the-value
> semantics in the caller, and `restart_audio_service_wait(...)` respectively.

**Registry path**: `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Audio`

**Value name**: `DisableProtectedAudioDG` (`REG_DWORD`)

**Semantics**: value absent or != 1 → blocked; value present and == 1 → allowed.

**Implementation notes**:
- `disable()` uses `RegKey::create()` for a writable handle and `write_dword()` with 1.
- The restore path deletes the value (a missing value is not an error).
- No unsafe pointer casts (the `RegKey::handle()` accessor already exposes the HKEY).

**Forbidden**: does not know `pipeline/` exists.

## 5.7 `install/device/stale.rs`

**Responsibility**: after Windows re-enumerates endpoints, the old GUID's slot keys disappear but VxAPO records remain (`HKLM\SOFTWARE\VxAPO\Child APOs\{oldGuid}`,
`C:\ProgramData\VxAPO\{oldGuid}`, `snapshots\{oldGuid}.json`). This module matches old records to currently active endpoints by **stable device identity** (endpoint history / device instance ID / hardware ID) and supports migration/cleanup.

**Layered matching (v9.24, fixed 2026-09-16)**: a Windows feature update re-rolls endpoint GUIDs and may delete the old endpoint key
`...\MMDevices\Audio\{Render|Capture}\{oldGuid}` **entirely** — the old "read the old endpoint key for the instance ID" pairing path then fails and records degrade to `unmatched` (measured: the App only had the cleanup exit; recovery never triggered). Now matching is layered by priority, with the hit source written to `matched_by`:

1. `endpoint_history`: the old GUID appears in an active endpoint's endpoint history
   property `{4b416b7d-8501-40c1-acfd-97aa9bdc17c8},1` (REG_MULTI_SZ, entries like
   `{0.0.0.00000000}.{guid}`); or the active GUID appears in the record's persisted
   `EndpointHistory` value;
2. `device_instance_id`: the instance ID from the old endpoint key while it is still
   readable (old path kept);
3. `stored_identity`: the `DeviceInstanceId` persisted in the record key at
   install/migration time;
4. `hardware_id`: intersecting hardware IDs and equal product name, and a **unique** hit
   (USB port-change fallback).

Multiple candidates (ambiguous) or no hit → `unmatched`: cleanup only; never guess, never auto-migrate.

**References**:
- `crate::install::device::{endpoint::query_endpoint, info::{enumerate_devices, detect_mode_for_guid}}`
- `crate::install::device::slots::{child_apo_key_exists, ChildApoKind, InstallMode, CHILD_APO_PATH_ROOT, FX_PROPERTIES_KEY, INSTALL_VERSION}`
- `crate::install::device::sysfx::{decode_backups, SYSFX_BACKUP_VALUE}`
- `crate::install::device::info::find_endpoint_path`, `install/selector/operation::{write_install_config, InstallConfig}`
- `crate::sys::registry::{delete_tree, split_key, RegKey, RegValue}`
- `crate::utils::guid::parse_guid_string`, `crate::utils::vx_error::{Result, VxApoError}`

**Exports to**: `vxapo-cli` (`stale list/migrate/cleanup/fix-acl`), App backend commands
`list_stale_installs` / `migrate_stale_install` / `cleanup_stale_install` / `repair_stale_acl`;
the residual-fallback path of `selector/operation.rs::uninstall_endpoint`.

**Public API**:

```rust
#[derive(Serialize)] #[serde(rename_all = "snake_case")]
pub struct StaleInstall {
    pub guid: String,
    pub device_instance_id: String,
    pub display_name: String,
    pub matched_by: Option<String>,
    pub config_path: Option<String>,
    pub config_mtime_ms: Option<u64>,
    pub snapshot_path: Option<String>,
    pub snapshot_mtime_ms: Option<u64>,
    pub premix_slot: Option<String>,
    pub postmix_slot: Option<String>,
    pub inferred_mode: String,
    pub has_child_backup: bool,
    pub has_sysfx_backup: bool,
    pub target_guid: Option<String>,
    pub target_name: Option<String>,
    pub target_state: String,          // matched_healthy | matched_partial | unmatched
    pub auto_repairable: bool,
}

#[derive(Serialize)] #[serde(rename_all = "snake_case")]
pub struct MigrationReport {
    pub success: bool,
    pub target_guid: String,
    pub config_from: Option<String>,
    pub snapshot_from: Option<String>,
    pub config_migrated: bool,
    pub snapshot_migrated: bool,
    pub install_repaired: bool,
    pub removed_guids: Vec<String>,
    pub warnings: Vec<String>,
}

/// Migration repair callback: builds and writes the target endpoint's install config.
/// Parameters (device_guid, device_name, install_mode). Implemented by the selector layer —
/// InstallConfig / write_install_config belong to selector and are not pulled into device.
pub type RepairInstallFn<'a> = &'a dyn Fn(&str, &str, InstallMode) -> Result<()>;

/// Scans stale GUID install records and matches them to active endpoints (read-only).
pub fn list_stale_installs() -> Result<Vec<StaleInstall>>;

/// Cleans up a stale GUID record that cannot be matched to an active endpoint.
pub fn cleanup_orphan(guid: &str) -> Result<()>;

/// Repairs the ACL of a migrated config/snapshot: grants Modify to the interactive user.
pub fn fix_config_acl(guid: &str) -> Result<()>;

/// Migrates an old-GUID install to a new GUID.
///
/// config_from / snapshot_from are explicit sources; by default the "newest config,
/// oldest snapshot" are chosen among the same device instance's old records and the
/// target's existing files.
pub fn migrate_install(
    from: &str,
    to: &str,
    config_from: Option<&str>,
    snapshot_from: Option<&str>,
) -> Result<MigrationReport>;
```

> **`auto_repairable`** (the App may run `stale migrate` at startup without asking):
> unique (non-ambiguous) target hit + the old record has **meaningful** config (> 64 B) +
> the target directory has **no** meaningful config (missing or still the APO's default
> placeholder). The driver is the single source of this judgment: when the target already
> has real config, never auto (would override the user's tuning); ambiguous / no source
> config / unmatched are also never auto (banner flow instead).

**Key paths**:
- Config root: `C:\ProgramData\VxAPO`; snapshots: `C:\ProgramData\VxAPO\snapshots`;
  migration backup: the `_migration_backup\{target_guid}` directory.
- Child APO root: `HKLM\SOFTWARE\VxAPO\Child APOs`.

**Implementation notes**:
- `target_state` is decided by "layered match result + target health"; `matched_by` records
  the hit source (`null` when unmatched) for CLI `stale list` and App diagnostics.
- The record key (`Child APOs\{guid}`) gains 4 **values** (not subkeys — migration's
  `copy_values` only copies values; subkeys would be silently dropped on migration):
  `DeviceInstanceId` (REG_SZ), `DeviceHardwareIds` (REG_MULTI_SZ),
  `DeviceProductName` (REG_SZ), `EndpointHistory` (REG_MULTI_SZ, lowercase with braces).
  They do **not** affect APO loading — loading reads only the `FxProperties` slots
  `{d04e05a6-…},0/3/5/6/7`; the key is still created only by install and deleted whole by
  uninstall.
- During install, `selector/operation.rs::write_child_apo_config` writes the identity
  values; after a successful migration they are refreshed with the active endpoint's
  identity, and `EndpointHistory` is written back as the union (persisted history, active
  endpoint history, each old GUID and the new GUID — deduped and sorted) so a future GUID
  refresh still recognizes the same device.
- Files that would be overwritten are backed up under the `_migration_backup\{target_guid}`
  directory before migration; `removed_guids` records deleted old GUIDs; `warnings` collects
  non-fatal problems (e.g. ACL repair failure).
- `fix_config_acl` uses `icacls` to grant the interactive-user SID `*S-1-5-4` Modify
  (directory recursive `(OI)(CI)M /T`).
- `stale migrate` **never stops the audio service** (measured 2026-09-16): writing/deleting
  endpoint `FxProperties` values only needs a `KEY_SET_VALUE` handle
  (`RegKey::open_for_write`), independent of audiodg holding the endpoint — deleting slot
  values on a live audio stream succeeds. Historical 0x80070005 errors came from the old
  implementation opening with `SAM_ALL` (CreateSubKey bit not granted by the ACL) or a
  read-only handle. The healthy path touches only ProgramData files and
  `HKLM\SOFTWARE\VxAPO\Child APOs` values; `config.toml` is never "held open" either
  (one-shot `fs::read` + directory-notification hot reload; the App keeps rewriting it with
  tmp+rename). The repair branch (unhealthy target install state → `write_install_config`
  rewrites the slots) **restarts that endpoint** afterwards
  (`audiodg::restart_endpoint_device`) so the change takes effect — the engine caches the
  endpoint APO chain; a registry-only change is not picked up by new streams (measured:
  after deleting a slot value, a new stream still loaded the old APO). The tail always calls
  `ensure_audio_service_running()` (idempotent when already running).
- `copy_file`'s tmp → target replace has bounded retries (5 × 50 ms, only for
  ACCESS_DENIED/SHARING_VIOLATION/LOCK_VIOLATION) — covering the microsecond window where a
  rename collides with a DLL read while the service is not stopped, so one transient sharing
  conflict does not hard-fail the whole migration.
- Calls needing administrator rights are elevated by the CLI / App paths; this module never
  pops UAC itself.

**Forbidden**: depends on the runtime semantics of `pipeline/`, `config/`.

## 5.8 `install/device/identity.rs`

**Responsibility** (new in v9.24): reading and persisting the **stable identity** that survives endpoint GUID refreshes; used by `stale.rs` pairing and written by
`selector/operation.rs` during install. Does not depend on the old endpoint key, so it
still works after Windows deletes the endpoint key entirely.

**Identity sources**:
- Active endpoint `Properties` subkey: `PKEY_DeviceInstanceId` (`{b3f8fa53-…},2`),
  `PKEY_Device_ProductName` (`{b3f8fa53-…},6`),
  `PKEY_DeviceInterface_FriendlyName` (`{a45c254e-…},2`),
  device-node hardware IDs (`{9dad2fed-3c19-4cde-b3c9-1bd56be25698},0`, REG_MULTI_SZ),
  endpoint history (`{4b416b7d-8501-40c1-acfd-97aa9bdc17c8},1`, REG_MULTI_SZ).
- VxAPO record-key values: `DeviceInstanceId` / `DeviceHardwareIds` / `DeviceProductName` /
  `EndpointHistory` (same value names `operation.rs` writes).

**Public API**:

```rust
pub const PROPERTIES_KEY: &str = "Properties";
pub const PKEY_DEVICE_INSTANCE_ID: &str = "{b3f8fa53-0004-438e-9003-51a46e139bfc},2";
pub const PKEY_DEVICE_PRODUCT_NAME: &str = "{b3f8fa53-0004-438e-9003-51a46e139bfc},6";
pub const PKEY_DEVICE_INTERFACE_FRIENDLY_NAME: &str = "{a45c254e-df1c-4efd-8020-67d146a850e0},2";
pub const PKEY_DEVICE_HARDWARE_IDS: &str = "{9dad2fed-3c19-4cde-b3c9-1bd56be25698},0";
pub const PKEY_ENDPOINT_HISTORY: &str = "{4b416b7d-8501-40c1-acfd-97aa9bdc17c8},1";
pub const VALUE_DEVICE_INSTANCE_ID: &str = "DeviceInstanceId";
pub const VALUE_DEVICE_HARDWARE_IDS: &str = "DeviceHardwareIds";
pub const VALUE_DEVICE_PRODUCT_NAME: &str = "DeviceProductName";
pub const VALUE_ENDPOINT_HISTORY: &str = "EndpointHistory";

pub struct EndpointIdentity { /* instance_id, hardware_ids, product_name, history, ... */ }

/// Pure helpers (unit-testable):
/// strips \\?\, {1}., {2}. prefixes **in a loop** (measured stacked forms like
/// {2}.\\?\usb#vid_…), maps # to \, uppercases, trims leading/trailing backslashes.
pub fn normalize_device_id(value: &str) -> String;
/// {0.0.0.00000000}.{guid} / {0.0.1.00000000}.{guid} / interface path / bare GUID
/// → lowercase {guid} (None on invalid input).
pub fn normalize_endpoint_guid(value: &str) -> Option<String>;
pub fn normalize_endpoint_history(values: &[String]) -> Vec<String>;
pub fn merge_endpoint_history(lists: &[Vec<String>]) -> Vec<String>;

pub fn read_endpoint_identity(endpoint_path: &str) -> EndpointIdentity;
pub fn identity_from_values(values: &HashMap<String, RegValue>) -> EndpointIdentity;
pub fn identity_from_key(key: &RegKey) -> EndpointIdentity;
pub fn write_identity_values(/* key, identity */);
```

**Implementation notes**:
- `read_endpoint_identity` degrades missing values to empty (no error), so a scan never
  fails because of one device's abnormal properties.
- Identity values are always written as **values**, never subkeys: `stale.rs::copy_values`
  copies only values during migration.
- Adding/reading identity does **not** change APO loading: loading reads only the
  `FxProperties` slots.

**Forbidden**: depends on `pipeline/`, `config/`; writes to `MMDevices` (endpoint keys are read-only here).

## Hard constraints

1. Never depend on pipeline/config.
2. Device enumeration must go through `install/device/info::enumerate_devices`.
3. Registry writes are transactional through `operation.rs`; no direct ad-hoc registry modification.





