# object Module Specification

**Boundary**: glue layer. The COM objects created when Windows loads the DLL. Allowed to depend on all modules.

**Allowed dependencies**: all other modules (`sys/`, `pipeline/`, `install/`, `config/`, `utils/`, `telemetry/`).

**Forbidden dependencies**: none at the module level; `object/apo/*` submodules must avoid circular `object/apo` dependencies except as authorized.

## Module tree

```
object/
├── apo.rs              # APO core module entry (ApoObject + the four _Impl interface impls)
│   └── apo/
│       ├── state.rs        # ApoState / StateCell / TransitionError / LockGuard
│       ├── inner.rs        # ApoObjectInner + build_dsp_context + startup fade
│       ├── init.rs         # Initialize + GetRegistrationProperties (+ sysfx self-heal)
│       ├── negotiate.rs    # format negotiation (extract_format_ref / check_format_supported)
│       ├── process.rs      # APOProcess / LockForProcess / Calc frames / Reset / GetLatency
│       ├── reload.rs       # WatcherState + start_watcher / stop_watcher / hot_reload_impl
│       ├── config.rs       # per-device config path resolution + diag log
│       ├── lock_key.rs     # config_stamp (mtime, size) — relock chain-reuse key
│       ├── rtdump.rs       # RT diagnostics dump (memory buffer, flushed at Unlock)
│       ├── test_pipe.rs    # DeviceTestPipeName notifications (test instrumentation)
│       ├── aggregate.rs    # COM aggregation shell (NApo / non-delegating IUnknown)
│       ├── child.rs        # child APO COM lifetime management (typed interface holder)
│       └── process/tests.rs
├── factory.rs          # #[implement(IClassFactory)] + LOCK_COUNT
├── ref_count.rs        # INST_COUNT atomic counter
├── vx_reg_props.rs     # REG_PROPS ×2 + query helpers + COM registration entries
└── dll_exports.rs      # DllMain + the five COM export functions + register_apo_with_path
```

> **Aggregation is mandatory (code-verified)**: the Windows audio engine always creates APOs
> **aggregated** (`pUnkOuter` non-null; probe-verified `punkouter_null=false`). windows-rs
> 0.62 `#[implement]` generates a self-contained IUnknown that does **not** delegate, so the
> engine's outer QI(IAudioProcessingObject) would never reach the inner object → the object is
> discarded → silence. `aggregate.rs` implements the EAPO aggregation semantics with a
> hand-written multi-interface offset layout (NApo): four independent vtable pointers, QI
> returning the matching field address, non-delegating AddRef/Release, methods forwarding to
> the inner `ApoObject`. See 7.7.

## Reference dependency table

> v9.17 (single source of truth): this table is no longer maintained independently — the main
> specification `Module Reference Specification (Without Detailed Modules).md`, section 11
> "Reference dependency table", is the only baseline; the per-file allowed and forbidden
> dependencies of object are listed line by line there.
>
> Change vs the original spec: `dll_exports.rs` no longer depends on
> `installation/clsid_entries`; it depends on `object/vx_reg_props` (the registration entries
> were merged), and the CLSIDs themselves are defined once in `sys/consts.rs`.

## 7.1 `object/apo.rs` — ApoObject definition

**Responsibility**: ApoObject implements the four APO interfaces and owns the full lifecycle state (atomic state machine + dual-chain transition + mutex-protected inner state). `apo.rs` is a thin **facade**: each `_Impl` trait method validates parameters, wraps the call in `catch_unwind` where required, and delegates to the `apo/` submodule functions.

**Interfaces implemented** (verified against the `#[implement]` attribute):
- `IAudioProcessingObject` (Initialize / LockForProcess / UnlockForProcess / Reset / GetLatency / GetRegistrationProperties)
- `IAudioProcessingObjectRT` (APOProcess / CalcInputFrames / CalcOutputFrames)
- `IAudioProcessingObjectConfiguration` (IsInputFormatSupported / IsOutputFormatSupported / GetInputChannelCount)
- `IAudioSystemEffects` (marker, empty impl)

**References**:
- `crate::sys::com::apo_interfaces::*` (the APO traits + IAudioMediaType)
- `crate::sys::com::apo_types::*` (POD types, APOInitSystemEffects)
- `crate::sys::com::prelude::*` (IUnknown, implement, HRESULT constants)
- `crate::sys::consts::{CLSID_VXAPO_PRE_MIX, CLSID_VXAPO_POST_MIX}`
- `crate::object::apo::child::ChildApo`
- `crate::object::vx_reg_props::props_for_clsid`
- `crate::object::ref_count`
- `crate::pipeline::process::{process_audio, process_chain_interleaved, apply_error_policy, ErrorPolicy, ProcessStatistics}`
- `crate::pipeline::chain::Chain`, `crate::pipeline::context::PipelineContext`
- `crate::pipeline::format::{extract_format, AudioFormat}`
- `crate::sys::audio_defs::get_channel_names`
- `crate::pipeline::dsp::filter::{DspContext, DeviceType, ProcessingStage}`
- `crate::pipeline::dsp::transition::{SmoothingProvider, mix_buffers, default_smoothing_length}`
- `crate::config::parser::ConfigParser`
- `crate::install::audiodg::ensure_can_load`
- `crate::object::apo::reload::hot_reload_impl`

### 7.1.1 Special GUID constants (historical)

> The `APOGUID_NOKEY` / `APOGUID_NOVALUE` / `GUID_NULL` sentinels and the
> `is_special_guid` / `is_valid_apo_guid` / `guid_matches` helpers described in the zh
> sibling **no longer exist in object code** (repo search: only comment references). The
> live equivalent lives in the install layer — `install/device/slots` models the "key
> missing / value empty / occupied by another APO" cases when reading child-APO slots (see
> the install spec 5.3). Kept here as historical design reference:

```rust
pub const APOGUID_NOKEY: GUID   = GUID::from_values(0, 0, 0, [0,0,0,0,0,0,0,0x01]);
pub const APOGUID_NOVALUE: GUID = GUID::from_values(0, 0, 0, [0,0,0,0,0,0,0,0x02]);
pub const GUID_NULL: GUID       = GUID::from_values(0, 0, 0, [0,0,0,0,0,0,0,0x00]);
```

### 7.1.2 ApoObjectState (removed)

> **Code-verified removal**: the `ApoObjectState` struct and the `ap_state: Mutex<ApoObjectState>`
> field described in the zh sibling **do not exist anymore** (0 hits repo-wide). Its format /
> channel / lock bookkeeping was absorbed as follows:
> - format and channel counts → `PipelineContext` (inside `ApoObjectInner`, mutex-protected)
> - lock state → `StateCell` (atomic, `ApoState::Locked`)
> - CLSID → `ApoObject.clsid` (immutable field)
> - `allow_silent_buffer_modification` → the silent-buffer policy is handled in
>   `pipeline/buffer.rs::evaluate_buffer` (BufferAction) and the process path.
>
> The zh struct below is kept as historical reference only.

```rust
pub struct ApoObjectState {   // REMOVED — historical
    pub clsid: GUID,
    pub child_apo_clsid: Option<GUID>,
    pub is_locked: bool,
    pub sample_rate: u32,
    pub input_channel_count: u32,
    pub output_channel_count: u32,
    pub channel_mask: u32,
    pub bits_per_sample: u32,
    pub allow_silent_buffer_modification: bool,
}
```

### 7.1.3 ApoObject fields (verified against code)

```rust
#[implement(
    IAudioProcessingObject,
    IAudioProcessingObjectRT,
    IAudioProcessingObjectConfiguration,
    IAudioSystemEffects
)]
pub struct ApoObject {
    /// This instance's CLSID (PreMix or PostMix; immutable).
    pub(crate) clsid: GUID,
    /// Atomic state machine (Created → Initialized → Locked).
    pub(crate) state_cell: StateCell,
    /// Core runtime state (dual-chain transition, buffers, spec fingerprint).
    /// Arc<Mutex>: spawn threads can clone it (hot_reload accesses independently).
    pub(crate) mutex: Arc<Mutex<ApoObjectInner>>,
    /// Latency in samples (control thread writes; GetLatency reads — actually unused
    /// since v8.3: GetLatency delegates to the child or returns 0).
    pub(crate) latency_samples: AtomicU32,
    /// Latency in frames (CalcInputFrames / CalcOutputFrames, lock-free).
    pub(crate) latency_frames_atomic: AtomicU32,
    /// Processing statistics (error_count).
    pub(crate) process_stats: ProcessStatistics,
    /// Config file path (fixed at Initialize; per-device
    /// C:\ProgramData\VxAPO\{GUID}\config.toml). Arc<Mutex> for the watcher thread.
    pub(crate) config_path: Arc<Mutex<String>>,
    /// Watcher runtime state (externally driven model): started at the end of Lock,
    /// stopped at Unlock. Arc<Mutex>: &self writable + movable into the thread.
    watcher_state: Arc<Mutex<WatcherState>>,
    /// Child APO (created at Initialize; degraded to None on failure).
    /// Arc<Mutex>: &self writable; RT path takes a short lock before locking inner.
    pub(crate) child_apo: Arc<Mutex<Option<ChildApo>>>,
}

impl ApoObject {
    pub fn new(clsid: GUID) -> Self { /* ref_count::increment(); ... */ }
    /// Config hot reload (watcher callback). Delegates to the module-level
    /// hot_reload_impl (shared Arc fields; passes Arc::as_ptr(&mutex) as a stable
    /// diagnostic address instead of &ApoObject).
    pub fn hot_reload(&self);
}
impl Drop for ApoObject { /* ref_count::decrement() */ }
```

**ApoObjectInner** (held by `ApoObject.mutex`; pure data + construction, no COM):

```rust
pub struct ApoObjectInner {
    pub current_chain: Box<Chain>,
    pub outgoing_chain: Option<Box<Chain>>,
    /// Retired chain (R1/v6.9, EAPO previousConfig-inspired): at the transition-completion
    /// frame the RT thread MOVES outgoing_chain here (.take(), zero destruction); the
    /// control thread (hot_reload / UnlockForProcess / Reset) drops it inside the lock.
    /// Heavy filter destructors never run on the RT thread.
    pub retired_chain: Option<Box<Chain>>,
    pub pipeline_context: PipelineContext,
    pub transition: Option<SmoothingProvider>,
    pub temp_buffers: Vec<Vec<f32>>,
    pub temp_buffer_old: Vec<f32>,
    pub temp_buffer_new: Vec<f32>,
    pub pending_reload: bool,
    /// Blocking-reload flag (R2/v6.9, EAPO loadSemaphore-aligned): at most one reload per
    /// transition cycle; new change requests during a reload are ignored.
    pub reloading: bool,
    /// Active config fingerprint (v7.9, P0-4) — the ordered filter_spec sequence of the
    /// chain currently in effect. LockForProcess establishes the baseline; hot_reload
    /// updates it as soon as parsing succeeds; UnlockForProcess keeps it (see lock key).
    pub active_spec: Vec<String>,
    /// Last successful Lock's (config_path, mtime, size, sample_rate, channel_names) key.
    /// A same-key Relock reuses the existing chain and keeps filter state (no rebuild
    /// transient/beep on endpoint renegotiation); mtime+size invalidate the key when the
    /// config file changes. UnlockForProcess deliberately does NOT clear it.
    pub last_lock_key: Option<(String, u64, u64, u32, Vec<String>)>,
    /// Transient RT dump (diagnostics): memory buffer + remaining frames + target path.
    /// The RT path only writes memory; Unlock (control thread) flushes to disk.
    pub rt_dump: Option<(PathBuf, Vec<f32>, usize)>,
    /// Longest single RT call in this lock cycle (ms; underrun diagnostics).
    pub rt_max_call_ms: u64,
    /// Startup fade totals (samples): the engine may still be loading the target APO
    /// chain when the stream starts — mute-hold then linear fade avoids the "first second
    /// stutter". STARTUP_FADE_HOLD_MS = 100, STARTUP_FADE_RAMP_MS = 0 (mute only).
    pub startup_fade_total: usize,
    pub startup_fade_remaining: usize,
    /// Last APOProcess calls (diagnostics, fixed RT array, zero allocation):
    /// (secs, in_frames, in_flags, in_peak, out_flags, out_peak) — logged at Unlock to
    /// distinguish "browser/engine injected huge input" from "DSP numerical explosion".
    pub last_calls: [(u64, u32, u32, f32, u32, f32); 8],
    pub last_call_idx: u64,
    /// Lock-cycle output peak high-water (RT writes, Unlock reads) + matching input peak
    /// and timestamp (v9.15 pop diagnosis).
    pub hot_out_peak: f32,
    pub hot_in_peak: f32,
    pub hot_secs: u64,
    /// Calls whose input flag was BUFFER_SILENT but whose content was non-zero
    /// (engine dirty-silent buffers) — count and max input peak (diagnostics).
    pub silent_dirty_calls: u32,
    pub silent_dirty_max_in: f32,
}
```

> **R1 (retired chain, v6.9)**: at the transition-completion frame the RT thread only does
> `inner.retired_chain = inner.outgoing_chain.take()` — moving Box ownership (zero
> destruction). The old chain hangs in `retired_chain` until the **control thread** (next
> hot_reload / UnlockForProcess / Reset, inside the lock) drops it. This is the essence of
> EAPO's `previousConfig` slot: the RT thread's destructor cost and race window are eliminated.
>
> **R2 (blocking reload, v6.9, EAPO semaphore-aligned)**: hot_reload returns immediately
> without building when `transition.is_some()` or `reloading` — no parsing during a
> transition; after the transition completes, APOProcess triggers one reload. With a 10 ms
> transition window the "blocked until transition completion" parse delay is negligible.
>
> **SAFETY (Send/Sync)**: the audio engine guarantees LockForProcess / UnlockForProcess /
> APOProcess do not overlap. Pipeline operations are protected by the mutex;
> `latency_samples` / `latency_frames_atomic` are `AtomicU32`.
> `unsafe impl Send for ApoObject {}` / `unsafe impl Sync for ApoObject {}`.

### 7.1.4 Atomic state machine

```rust
/// APO lifecycle states (mirrors the Windows engine's driving order).
#[repr(u8)]
#[derive(Clone, Copy, PartialEq, Eq, Debug)]
pub enum ApoState {
    Created     = 0,   // object created; Initialize not yet called
    Initialized = 1,   // Initialize succeeded; LockForProcess not called (or already Unlocked)
    Locked      = 2,   // LockForProcess succeeded; APOProcess may be called
}

/// State-transition failure details (v6.6, tympan-apo-inspired).
/// Carries expected / attempted / actually-observed states for quick diagnosis.
pub struct TransitionError {
    pub expected: ApoState,
    pub attempted: ApoState,
    pub actual: ApoState,
}
impl std::fmt::Display for TransitionError { /* human-readable message */ }
/// Every transition failure maps to APOERR_ALREADY_INITIALIZED (0x887D_0001)
/// when converted for a COM boundary.
impl From<TransitionError> for HRESULT { /* APOERR_ALREADY_INITIALIZED */ }

/// Atomic state cell.
pub struct StateCell { state: AtomicU8 }

impl StateCell {
    pub fn new() -> Self;   // starts at Created
    /// CAS transition: Ok(()) on success, TransitionError on failure.
    pub fn transition(&self, from: ApoState, to: ApoState) -> Result<(), TransitionError>;
    /// Current state (Acquire ordering; safe to call on the RT path).
    pub fn current(&self) -> ApoState;
}
```

> **Code-verified API note**: the zh sibling additionally lists `is_locked()`,
> `release()`, `initialize()`, `lock()`, `unlock()` convenience methods — these do **not
> exist** in the current code. Callers use `transition(from, to)` and `current()` directly.
> The "release() resets to Created for DLL unload" behavior is covered by `Drop` +
> `ref_count::decrement` instead.

**Transition rules**:
- `Initialize`: requires `Created` → `Initialized`
- `LockForProcess`: requires `Initialized` → `Locked`
- `APOProcess`: requires `Locked` (state unchanged; checked via `current()`)
- `UnlockForProcess`: requires `Locked` → `Initialized`

### 7.1.5 RAII rollback guard

```rust
/// object/apo/state.rs — RAII guard over the state cell.
/// Constructed after a successful Initialized → Locked transition; on Drop (unless
/// disarmed) it transitions back Locked → Initialized, rolling back a failed
/// LockForProcess. Disarm via `disarm()` on success.
pub struct LockGuard<'a> { /* state_cell: &'a StateCell, armed */ }
impl<'a> LockGuard<'a> {
    pub fn new(state_cell: &'a StateCell) -> Self;
    pub fn disarm(self);
}
impl Drop for LockGuard<'_> { /* rolls back to Initialized when still armed */ }
```

> The zh sibling's `LockForProcessGuard` (defined in apo.rs) is the same concept by an
> older name; the live type is `LockGuard` in `state.rs`, and `process.rs::lock_for_process`
> creates it right after the state transition (`let _guard = LockGuard::new(&apo.state_cell);`).

### 7.1.6 Format-extraction helper

```rust
/// object/apo/negotiate.rs — extracts WAVEFORMATEX info from an IAudioMediaType
/// reference (windows-rs Ref<IAudioMediaType>). Uses the #[interface] trait method
/// (GetAudioFormat) — no raw vtable indexing.
pub(crate) fn extract_format_ref(media_ref: &Ref<IAudioMediaType>) -> Result<AudioFormat>;
```

The pipeline-level `pipeline::format::extract_format` (unsafe, raw pointer) remains the
single extraction implementation; `extract_format_ref` unwraps the COM reference and maps
errors for the negotiation paths.

### 7.1.7 Format-negotiation business types (historical)

> The `FormatSupportResult` / `FormatNegotiation` / `ConnectionFormat` / `LockConfig`
> types in the zh sibling **do not exist in code** (0 hits). The live negotiation API is:
>
> ```rust
> pub(crate) fn check_format_supported(p_requested: &Ref<IAudioMediaType>) -> Result<()>;
> pub(crate) fn is_input_format_supported(
>     p_opposite_format: &Ref<IAudioMediaType>,
>     p_requested: &Ref<IAudioMediaType>,
> ) -> Result<IAudioMediaType>;
> pub(crate) fn is_output_format_supported(
>     p_opposite_format: &Ref<IAudioMediaType>,
>     p_requested: &Ref<IAudioMediaType>,
> ) -> Result<IAudioMediaType>;
> ```
>
> Kept below as historical design reference.

```rust
pub enum FormatSupportResult { Supported, UnsupportedWithAlternative, Unsupported }  // historical
pub struct FormatNegotiation { /* in/out channels, rates, bits, masks */ }           // historical
pub struct ConnectionFormat { pub channel_count: u32, pub sample_rate: u32,
    pub bits_per_sample: u32, pub channel_mask: u32 }                                // historical
pub struct LockConfig { pub inputs: Vec<ConnectionFormat>, pub outputs: Vec<ConnectionFormat> } // historical
```

### 7.1.8 `Initialize` (with per-device config-path resolution, v7.2)

**Responsibility**: parse `APOInitSystemEffects` (endpoint GUID via
`pAPOSystemEffectsProperties` → `PKEY_AudioEndpoint_GUID` → `VT_CLSID`), resolve the
per-device config path, ensure `C:\ProgramData\VxAPO\{GUID}\config.toml` exists, create
the child APO, and run the sysfx self-heal.

**Implementation location (code-verified)**: `object/apo/init.rs` —
`pub(crate) fn initialize(apo: &ApoObject_Impl, cb_data_size: u32, pby_data: *const u8) -> Result<()>`
(+ `get_registration_properties`). The flow:

1. Parameter check: `pby_data` non-null, `cb_data_size >= size_of::<APOInitSystemEffects>()`.
   Invalid → **degrade to the default config** (warn log + `resolve_config_path(None)`) and
   still return Ok (v7.6: Initialize failure would keep the APO from loading and stall the
   stream; degrading to passthrough is more robust — SDK fault-tolerance convention).
2. `state_cell.transition(Created, Initialized)`; failure → `APOERR_ALREADY_INITIALIZED`.
3. Parse `APOInitSystemEffects` (cast `pby_data`):
   - `pAPOSystemEffectsProperties` (IPropertyStore) → GetValue(PKEY_AudioEndpoint_GUID) →
     PROPVARIANT (VT_CLSID) → `puuid` → endpoint GUID.
   - **The child-APO GUID is NOT in APOInitSystemEffects** (v8.4, P0-6): the struct only has
     APOInit / pAPOEndpointProperties / pAPOSystemEffectsProperties / pReserved /
     pDeviceCollection. The child GUID is read from the registry:
     `HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}\{PreMixChild|PostMixChild}`
     (VxAPO's own install-info area — **never** EAPO's
     `HKLM\SOFTWARE\EqualizerAPO\Child APOs`).
   - Empty/special child GUID → None (degraded mode, no child APO).
4. If a child GUID exists, create `ChildApo::create(clsid)` (failure degrades to no child;
   never blocks initialization).
5. Runtime self-heal (v9.4): after Windows re-enumeration the driver template may re-install
   Microsoft CAPX into `MSFX\N` alongside VxAPO (stuttering/slow playback). By endpoint
   GUID, locate the MMDevices path and call
   `install/device/sysfx::ensure_takeover_for_endpoint`: acts only when the endpoint's
   FxProperties already has VxAPO installed; replaces/deletes only Microsoft CAPX CLSIDs,
   never third-party APOs; also deletes `DisableEnhancements` / `Disable_SysFx` to
   re-enable the enhancement chain. Failure only logs, never blocks initialization.
6. Resolve the per-device config path (v8.9 plan A — system-wide ProgramData):
   - `CONFIG_ROOT = C:\ProgramData\VxAPO` (shared by all users; readable/writable by the
     service account **and** the current user. Not user Documents — the APO really runs in
     **audiodg** as `NT AUTHORITY\LOCAL SERVICE` (with the `NT SERVICE\Audiosrv` service
     SID), whose Documents folder cannot see files the CLI (a user process) writes).
   - device GUID → uppercase string (`guid_to_string`), path =
     `{CONFIG_ROOT}\{GUID}\config.toml`.
   - Directory missing → `std::fs::create_dir_all`; on failure fall back to
     `DEFAULT_CONFIG_PATH = C:\ProgramData\VxAPO\config.toml`.
   - `config.toml` missing → write a default passthrough config (the DLL never ships
     presets; presets are the App/CLI's job).
   - No endpoint GUID at all → fall back to the shared
     `C:\ProgramData\VxAPO\_default\config.toml` (`DEFAULT_DEVICE_DIR`), with a warning.
7. Store `config_path`. The watcher does **not** start here (v7.9: nothing is locked yet —
   no current_chain/active_spec for hot_reload to install into); it starts at the end of
   LockForProcess (7.1.9) with the lock-cycle lifetime.

### 7.1.9 `LockForProcess`

**Implementation location (code-verified)**: `object/apo/process.rs::lock_for_process(apo: &ApoObject_Impl, num_input, pp_inputs, num_output, pp_outputs) -> Result<()>`.

**Complete flow** (structure verified against code):

```text
Step 0  state_cell.transition(Initialized, Locked)  (failure → APOERR_ALREADY_INITIALIZED)
        LockGuard::new(&state_cell)                 (RAII rollback on any ? failure)
        validate num_input/pp_inputs/num_output/pp_outputs (APOERR_NUM_CONNECTIONS_INVALID)

Step 1  extract input + output AudioFormat from the connection descriptors
        (desc.pFormat is ManuallyDrop<Option<IAudioMediaType>>; None →
        APOERR_INVALID_CONNECTION_FORMAT)
        EAPO render-mask semantics: channel_mask = output mask; when the output mask is 0
        and in/out channel counts match → fall back to the input mask.
        (Input/output sample-rate mismatch → E_INVALIDARG; channel-count mismatch →
        APOERR_INVALID_CONNECTION_FORMAT.)

Step 2  build PipelineContext { sample_rate, input_channels, output_channels, channel_mask,
        max_frame_count (usize), bits_per_sample }
        build DspContext { ..., channel_names = get_channel_names(channel_mask),
        device_type: DeviceType::Render, stage: ProcessingStage::None, variables: {} }

Step 3  config_path = apo.config_path.lock()  (poison-aware: into_inner)
        is_postmix = clsid == CLSID_VXAPO_POST_MIX
        lock_key = (config_path, config_stamp mtime+size, sample_rate, channel_names)
        reuse_cached = last_lock_key == Some(lock_key) && !current_chain.is_empty()
        → PostMix instances: empty chain (passthrough), no config parse, no watcher.
          Reason (v9.4): Windows attaches BOTH SFX(PreMix) and EFX(PostMix) VxAPO instances
          to a render device; parsing the same config twice would apply the user config
          twice (GraphicEQ double convolution: too-quiet + double latency/CPU). PostMix
          keeps the child-APO delegation (the previous EFX APO still runs) but loads no
          user config.
        → cache hit: reuse the existing chain, KEEP filter state (fixes the rebuild
          transient/beep on every song change / new stream); mtime+size invalidate the key
          when the config changed, forcing a re-parse.
        → cache miss: ConfigParser::new().parse_file_with_spec(&config_path, &dsp_ctx)
          → (filters, spec_chain) (the spec chain becomes the active_spec baseline)

Step 4  build Chain: add_filter for each, chain.initialize(sample_rate, channel_names)
        (precompute DSP coefficients/state), total_latency = chain.total_latency()

Step 5  preallocate transition buffers (v7.8: the RT thread must never resize):
        temp_buffers = vec![vec![0.0; max_frame_count]; max(in,out)_channels]
        temp_buffer_old/new = vec![0.0; max_frame_count × max_channels]

Step 6  install::audiodg::ensure_can_load()?   (DisableProtectedAudioDG check/fix)

Step 7  write inner (under the mutex): current_chain / outgoing=None / retired=None /
        transition=None / pipeline_context / temp buffers / pending_reload=false /
        reloading=false / active_spec = spec_chain / last_lock_key = lock_key
        latency_samples.store(total_latency); latency_frames_atomic.store(total_latency)
        (NOTE: latency is computed but NOT reported outward — GetLatency delegates to the
        child or returns 0, see 7.1.14)

Step 8  child APO lock delegation (child may refuse; degradation never blocks the parent)

Step 9  guard.disarm()  (success — no rollback)
        start the watcher (reload::start_watcher) unless PostMix; failure only logs
        (hot reload unavailable, audio path unaffected)
```

> **Watcher conventions (v7.9/v7.10, externally driven model)**:
> - watched object = the config_path **parent directory**
>   (`C:\ProgramData\VxAPO\{GUID}`), not the file — directory watches survive
>   delete+recreate of the file (EAPO `FindFirstChangeNotificationW` alignment).
> - **event-driven** (v7.8): `FindFirstChangeNotificationW` + `WaitForMultipleObjects`
>   blocking wait (no 2000 ms polling) — ~10 ms latency, no idle CPU.
> - **10 ms dedup window** (v7.8): editor "write temp + rename" notification bursts merge;
>   content-unchanged changes are short-circuited by the spec-fingerprint comparison (7.1.18).
> - **lifetime follows the lock cycle**: started at the end of LockForProcess, stopped at
>   UnlockForProcess (SetEvent + join). Changes while unlocked take effect at the next Lock
>   (which re-reads the config and rebuilds the baseline).
> - **anti-spin (v9.4)**: a failing `FindNextChangeNotification` reset closes the handle and
>   exits the loop cleanly — never an infinite reload loop.
> - v9.15 simplification: the (mtime,size) per-file pre-check table and the experimental
>   process-wide `RELOAD_LOCK` were removed — per-instance reload + 10 ms dedup + spec
>   fingerprint short-circuit remain; `diag_append` gained a process-wide write lock.
> - Implementation (code-verified): `object/apo/reload.rs` — `WatcherState`,
>   `start_watcher(apo)`, `stop_watcher(apo)`, `hot_reload_impl(...)`, plus the
>   `ReloadingClear` RAII that resets `reloading`.

### 7.1.10 `UnlockForProcess`

```text
1. child APO UnlockForProcess delegation (failure only logs; unlock has no retry semantics)
2. state_cell.transition(Locked, Initialized)   (failure → HRESULT)
3. clear inner: current_chain = Chain::new(), outgoing = None, retired = None (control-thread
   drop inside the lock, R1), transition = None, pipeline_context = PipelineContext::new(),
   temp buffers cleared, active_spec cleared
4. latency_samples / latency_frames_atomic = 0
5. last_lock_key is deliberately KEPT (chain reuse across relocks; see 7.1.9 Step 3)
6. stop the watcher (reload::stop_watcher — SetEvent + join + close handles; idempotent)
7. flush the RT dump to disk (rtdump::rt_dump_flush) and log the cycle diagnostics
   (rt_max_call_ms, last_calls ring, hot peaks, silent-dirty counters)
```

### 7.1.11 `APOProcess`

> **catch_unwind entry protection (v8.2/v8.3)**: APOProcess is an `extern "system"` RT
> entry (generated by `#[implement]`); unwinding across the FFI boundary is UB. Three
> layers with distinct jobs:
> 1. **compile-time constraint (O1)**: the RT path does no allocation / locking / I/O and
>    normally cannot panic — the primary defense;
> 2. **debug-test `catch_unwind` (`panic="unwind"`)**: capture → safe degraded output,
>    proving the "even a panic must not cross FFI" defense (the DoD tests exercise this);
> 3. **release backstop (`panic="abort"`, O3)**: any escaped panic deterministically
>    terminates the process — "recoverable process restart" instead of UB;
>    `telemetry/panic.rs`'s hook records the stack before abort — a diagnostic tool, not a
>    defense line.
> Capture behavior (debug only; release is a no-op): zero the output buffer +
> `output_props[0].u32BufferFlags = BUFFER_SILENT` + `stats.error_count++` + log
> (RT zero-allocation: telemetry's fixed ring buffer, no `format!`/`Box`).

**Implementation location (code-verified)**: `object/apo/process.rs::apo_process` →
`apo_process_inner` (under `rt_assert_in_rt!`), with `apo_process_panic_fallback` as the
degraded output writer.

**Pseudocode** (structure verified against code):

```text
apo_process(apo, input_props, output_props, params):
    catch_unwind(AssertUnwindSafe(|| apo_process_inner(...)))
        on panic → apo_process_panic_fallback (zero outputs, BUFFER_SILENT,
                   stats.error_count++, RT-safe log)

apo_process_inner:
    1. state check: state_cell.current() != Locked → return (no output write obligation
       beyond the fallback contract)
    2. extract the input slice — BufferInfo::from_prop(&input_props[0]).as_slice()
       (system APO hard requirement: single input)
    3. child-APO pre-delegation (v8.1, P0-6 — EAPO childRT->APOProcess runs first):
       runs once per frame on the input buffer (in place or into temp_buffer_child); its
       output is the VxAPO chain's input. The child is NOT part of current/outgoing chains
       (held separately); both chains share the same child output. No child (degraded) →
       skip, pure VxAPO processing. (childRT->CalcInputFrames is NOT called per frame —
       it is an engine-driven query method; calling it per frame wastes a vtable call.)
    4. lock inner (apo.mutex)
    5. transition mode (inner.outgoing_chain.is_some()):
       - old chain  → process_chain_interleaved → temp_buffer_old (+ apply_error_policy)
       - new chain  → process_chain_interleaved → temp_buffer_new (+ apply_error_policy)
       - factor = transition.advance().unwrap_or(1.0)
       - mix_buffers(old, new, output, frames × channels, factor)   [unsafe, non-overlap]
       - output_flags = BUFFER_VALID
       - on completion (finished):
           retired_chain = outgoing.take()        (R1: move, zero destruction)
           transition = None
           if pending_reload && !reloading:       (R2: trigger once, do NOT set reloading
           pending_reload = false; drop(inner);      here — hot_reload sets it itself)
           hot_reload(); return
       - defensive: pending set but transition already cleared (no mixer) → copy input to
         output (bypass — the APO contract requires writing output every frame) + retire
         the old chain + reload if allowed
       - advance() == None (transition exhausted) → factor = 1.0 (pure new chain)
    6. normal mode:
       process_audio(input_props, output_props, params, &mut current_chain, stats,
                     &mut temp_buffers)   (buffer evaluation → silence short-circuit →
       deinterleave → chain → interleave; errors → apply_error_policy + log)
    7. startup fade: apply_startup_fade (mute-hold 100 ms, ramp 0 ms — see inner fields)
    8. RT diagnostics: record_rt_call (last_calls ring), rt_max_call_ms, hot peaks,
       silent-dirty detection (SILENT_DIRTY_THRESHOLD = 1e-3), optional rt_dump_push
```

> **`apply_error_policy`** (pipeline 4.6): `Ok` → nothing; `Err` → `error_count++` then
> Bypass (copy input → output, zero the tail) or Silence (zero the output). Production
> constructs Bypass only (the Silence variant is a dead cluster kept as the documented
> policy for RT-safety breaches).

### 7.1.12 `CalcInputFrames` / `CalcOutputFrames` (lock-free design)

> **catch_unwind conservative returns (v8.2/v8.3)**: both entries are `extern "system"` RT
> calls (the engine asks for frame counts every block). Under debug (`panic="unwind"`)
> they are wrapped in catch_unwind; on panic they return **conservative values** (prefer
> over-requesting input, never write out of bounds):
> - `CalcInputFrames` panic → `output_frames + latency_frames_atomic` computed from a fresh
>    load where possible, otherwise `output_frames` (implementation choice locked by DoD
>    tests; the panic branch must not re-panic);
> - `CalcOutputFrames` panic → `0` (never overrun the output buffer).
> Release (`panic="abort"`) makes catch_unwind a no-op (O3) — an escaped panic aborts the
> process deterministically; the panic hook only records the scene beforehand.

```rust
// object/apo/process.rs::calc_input_frames / calc_output_frames (code-verified)
fn CalcInputFrames(&self, output_frames: u32) -> u32 {
    let latency = self.latency_frames_atomic.load(Ordering::Acquire);
    output_frames + latency
}
fn CalcOutputFrames(&self, input_frames: u32) -> u32 {
    let latency = self.latency_frames_atomic.load(Ordering::Acquire);
    if input_frames >= latency { input_frames - latency } else { 0 }
}
```

> Lock-free via `latency_frames_atomic` (AtomicU32) — the RT path never takes the mutex
> here. Per the thread-safety doc, the stored value is always 0 in practice (latency is not
> reported outward), so both functions effectively pass frame counts through unchanged.

### 7.1.13 `Reset`

```text
(process.rs::reset, code-verified structure)
lock inner;
current_chain = Chain::new(); outgoing = None; retired = None (control-thread drop);
transition = None; pipeline_context = PipelineContext::new();
temp_buffers / temp_buffer_old / temp_buffer_new cleared;
active_spec cleared; latency_samples / latency_frames_atomic = 0;
(last_lock_key kept — chain reuse on the next identical lock)
S_OK
```

### 7.1.14 `GetLatency`

> **v8.3 correction (EAPO source alignment, EqualizerAPO.cpp 82-95)**: EAPO sets
> `*pTime = 0` and only delegates to `childAPO->GetLatency` **when a child exists**; with
> no child it returns 0 (EAPO does not maintain its own latency value). VxAPO aligned.

```rust
fn GetLatency(&self, p_latency: *mut REFERENCE_TIME) -> HRESULT {
    if p_latency.is_null() { return E_POINTER; }
    unsafe { *p_latency = 0 };                       // v8.3: init to 0
    if let Some(child) = self.child_apo.lock() {     // short lock (Arc<Mutex<Option<ChildApo>>>)
        if let Some(ref child) = *child {
            unsafe { *p_latency = child.get_latency() };  // delegate
        }
    }
    S_OK                                             // no child → 0
}
```

### 7.1.15 `GetRegistrationProperties`

```text
(init.rs::get_registration_properties, code-verified location)
pp_props null → E_POINTER;
props = props_for_clsid(&self.clsid)  (from object/vx_reg_props; None → E_FAIL);
CoTaskMemAlloc(size_of::<APO_REG_PROPERTIES>()) (null → E_OUTOFMEMORY);
copy_nonoverlapping the static props; *pp_props = mem; S_OK.
```

> **Memory contract**: the returned `APO_REG_PROPERTIES` is allocated with
> `CoTaskMemAlloc`; the caller must free it with `CoTaskMemFree` (standard COM convention).

### 7.1.16 `IsInputFormatSupported` / `IsOutputFormatSupported`

> **Timing constraint (v7.7)**: these are called by the engine during **format negotiation,
> before LockForProcess** — `pipeline_context` is still `PipelineContext::new()` (all
> zeros). The checks must therefore be **absolute property checks** on the requested format,
> never comparisons against the (not yet existing) locked context.

**Verified checks** (`object/apo/negotiate.rs`):
- `check_format_supported`: the format must be **IEEE float** (`is_float_format` —
  WAVE_FORMAT_IEEE_FLOAT), channel count 1–8. Failure → `APOERR_FORMAT_NOT_SUPPORTED`.
- `is_input_format_supported` (EAPO-aligned, EqualizerAPO.cpp 228-306): reads input +
  output formats and rejects only **downmix** (input > 2 ch and input > output channels).
  EAPO returns S_FALSE + the device output format; windows-rs `Result` cannot express
  S_FALSE, so the downmix rejection degrades to `Err(APOERR_FORMAT_NOT_SUPPORTED)`
  ("rather reject than mis-accept"). On success returns the **requested format** (INPLACE
  mode: input and output formats are identical). A missing/unparseable opposite format
  skips the downmix check (conservatively accept).
- `is_output_format_supported`: symmetric — rejects **upmix** (output > 2 ch and
  output > input channels).
- The zh sibling's "sample-rate range 44.1 kHz–192 kHz" check **does not exist in code** —
  there is no sample-rate range validation in the negotiation path (code-verified). Sample
  rates pass through if the float + channel-count checks pass.

### 7.1.17 `GetInputChannelCount`

```rust
fn GetInputChannelCount(&self, p_count: *mut u32) -> HRESULT {
    if p_count.is_null() { return E_POINTER; }
    if self.state_cell.current() != ApoState::Locked { return APOERR_NOT_INITIALIZED; }
    let inner = self.mutex.lock().unwrap();
    unsafe { *p_count = inner.pipeline_context.input_channels };
    S_OK
}
```

### 7.1.18 `hot_reload` (watcher callback, background thread)

**Implementation location (code-verified)**: `ApoObject::hot_reload` →
`object/apo/reload.rs::hot_reload_impl(&config_path, &mutex, clsid, addr)`.

**Flow**:

```text
1. short-lock check: transition.is_some() || reloading → return (R2 blocking semantics;
   after the transition completes, APOProcess triggers the next hot_reload)
2. 128 KB gate (v7.9, P0-4): config file larger than MAX_CONFIG_FILE_SIZE (128 KB) →
   keep the old chain + error log (per-file check happens again inside
   parse_file_with_spec for every Include; this is an early short-circuit)
3. parse OUTSIDE the lock: ConfigParser::parse_file_with_spec(&config_path, &dsp_ctx)
   → (filters, new_spec); Err (syntax / Include failure / any file over 128 KB) →
   keep the old chain, do NOT touch active_spec, error log (v7.9: Include failure =
   whole-file failure; only a FIRST load failure at LockForProcess builds an empty chain)
4. spec-fingerprint short-circuit (v7.9, P0-4): short-lock compare new_spec ==
   active_spec → identical means the config is effectively unchanged (comments/whitespace/
   resave/unrelated-directory changes) → skip (no new chain, no transition, debug log).
   The comparison happens in a short lock BEFORE building (avoids a TOCTOU between the
   lock-free read and the swap; O(n) string equality is far cheaper than parse+build).
5. build the new chain OUTSIDE the lock (add_filter each; initialize(sample_rate,
   channel_names)); add_filter failure → keep the old chain
6. short-lock swap (pointer swap only):
   transition still active (race fallback) → pending_reload = true; return
   else: active_spec = new_spec (updated together with the chain pointer);
   outgoing_chain = Some(old chain); current_chain = new chain;
   transition = Some(SmoothingProvider::new(default_smoothing_length(sample_rate)));
   pending_reload = false; reloading = false
   (the next change is triggered by APOProcess after this transition completes)
```

### 7.1.19 Compile-time asserts

> The zh sibling's GUID-uniqueness `const` asserts are **not present in object code**
> (code-verified — the special GUIDs themselves were removed, see 7.1.1). The equivalent
> invariants are covered by `vx_reg_props.rs`'s static REG_PROPS construction and tests.
> Kept as historical reference:

```rust
const _: () = {
    assert!(APOGUID_NOKEY.data1 != APOGUID_NOVALUE.data1 || APOGUID_NOKEY.data4[7] != APOGUID_NOVALUE.data4[7]);
    assert!(GUID_NULL.data1 != APOGUID_NOKEY.data1 || GUID_NULL.data4[7] != APOGUID_NOKEY.data4[7]);
    assert!(GUID_NULL.data1 != APOGUID_NOVALUE.data1 || GUID_NULL.data4[7] != APOGUID_NOVALUE.data4[7]);
};
```

## 7.2 `object/apo/child.rs` — child APO COM lifetime management

**Responsibility**: hold the child APO's COM interfaces and delegate calls.

**Child-APO provenance (v8.1 + v8.4 path isolation, P0-6 EAPO alignment)**:
- the child APO = the **previous owner of the slot VxAPO took over** at install time
  (`PreMixChild` / `PostMixChild` values, stored in VxAPO's **own** install-info area
  `HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}` — the EAPO `DeviceAPOInfo`
  childApoPath **mechanism**, but path-isolated: **never** write EAPO's
  `HKLM\SOFTWARE\EqualizerAPO\Child APOs`; polluting it would corrupt EAPO's install
  info. All slots are backed up for fallback; the child corresponds to the actually
  occupied slot only).
- at runtime `Initialize` (7.1.8) reads that value; non-empty and non-special →
  `ChildApo::create`, otherwise `child_apo = None` (degraded passthrough, Note 57).
- **slot-loss detection (v8.1/v8.5)** is the App/CLI's job at startup / device switch: check
  only the slots the install mode uses (`InstallMode::premix_slot` + `postmix_slot`, e.g.
  SfxEfx → SFX+EFX — not all 5 slots). Before reinstalling, the current (non-VxAPO) value
  of the occupied slot is overwritten as the new `PreMixChild`/`PostMixChild` (latest
  predecessor — correct fallback even after several takeovers), then the slot is set to the
  VxAPO CLSID (`install_endpoint`). Full vs partial backup is decided by
  `install/device/slots::child_apo_key_exists`; uninstall always deletes the childapo key
  (next install takes the full-backup path).
- **runtime delegation**: childRT->APOProcess runs **first, once per frame** (on the input
  buffer) → VxAPO's dual chains process its output; the child is in neither chain.
- **GetLatency delegation (v8.3)**: `*pTime = 0`, delegate only when a child exists, else 0.

**Struct** (verified against code):

```rust
/// Holds typed COM interface references; calls go through #[interface] trait methods
/// (no manual vtable indexing). Drop releases all three references automatically.
pub struct ChildApo {
    iapo: IAudioProcessingObject,
    iapo_rt: IAudioProcessingObjectRT,
    iapo_cfg: IAudioProcessingObjectConfiguration,
}
unsafe impl Send for ChildApo {}
unsafe impl Sync for ChildApo {}
```

**Methods** (verified against code — note the code exposes `calc_output_frames`, while the
zh sibling lists `calc_input_frames`; both wrap the child's RT queries where used):

```rust
impl ChildApo {
    /// CoCreateInstance → IUnknown → QI the three interfaces → drop IUnknown.
    /// # Safety: COM must be initialized; clsid must be a valid APO CLSID.
    pub unsafe fn create(clsid: &GUID) -> Result<Self, HRESULT>;
    pub fn is_valid(&self) -> bool;

    // IAudioProcessingObject delegation
    pub fn get_latency(&self) -> REFERENCE_TIME;
    pub fn reset(&self) -> HRESULT;
    pub unsafe fn get_registration_properties(&self, pp_props: *mut *mut APO_REG_PROPERTIES) -> HRESULT;
    pub unsafe fn initialize(&self, cb_data_size: u32, pby_data: *const u8) -> HRESULT;
    // v8.6: input params are Option<&IAudioMediaType> (nullable borrows); the COM output
    // pp_supported stays a raw pointer (caller-allocated) — hence unsafe.
    pub unsafe fn is_input_format_supported(&self, p_opposite: Option<&IAudioMediaType>,
        p_requested: Option<&IAudioMediaType>, pp_supported: *mut *mut IAudioMediaType) -> HRESULT;
    pub unsafe fn is_output_format_supported(&self, /* same shape */) -> HRESULT;
    pub fn get_input_channel_count(&self, p_count: *mut u32) -> HRESULT;

    // IAudioProcessingObjectRT delegation
    pub fn calc_output_frames(&self, input_frames: u32) -> u32;

    // IAudioProcessingObjectConfiguration delegation
    pub unsafe fn lock_for_process(&self, num_input: u32,
        pp_inputs: *const *const APO_CONNECTION_DESCRIPTOR, num_output: u32,
        pp_outputs: *const *const APO_CONNECTION_DESCRIPTOR) -> HRESULT;
    pub unsafe fn apo_process(&self, /* APO_CONNECTION_PROPERTY arrays */);
    pub fn unlock_for_process(&self) -> HRESULT;
}
```

## 7.3 `object/factory.rs` — ClassFactory

**Responsibility**: `IClassFactory` implementation + `LOCK_COUNT` management.

**LOCK_COUNT** (verified against code):

```rust
static LOCK_COUNT: AtomicU32 = AtomicU32::new(0);
pub fn lock_increment() -> u32;
pub fn lock_decrement() -> u32;
pub fn lock_count() -> u32;
pub fn lock_is_zero() -> bool;
#[cfg(test)]
pub fn lock_reset_for_test();
```

**ClassFactory** (verified against code):

```rust
#[implement(IClassFactory)]
pub struct ClassFactory { target_clsid: GUID }

impl IClassFactory_Impl for ClassFactory_Impl {
    fn CreateInstance(&self, punkouter: Ref<'_, IUnknown>, riid: *const GUID,
        ppvobject: *mut *mut c_void) -> windows_core::Result<()> {
        // 1. validate riid/ppvobject (E_INVALIDARG) BEFORE writing
        // 2. *ppvobject = null (COM convention)
        // 3. aggregation SUPPORTED (EAPO-aligned): the engine always aggregates; when
        //    punkouter is non-null, riid must be IUnknown (else E_NOINTERFACE)
        // 4. create the aggregate shell NApo via aggregate::create_aggregate(outer_raw,
        //    target_clsid) — BOTH aggregated and non-aggregated creation go through it
        //    (null → null_mut outer); null NApo → E_OUTOFMEMORY
        // 5. QI the NApo for riid (vtable slot 0); failure → release_aggregate(na) + Err
        // 6. release the factory's temporary reference (release_aggregate) — EAPO
        //    ClassFactory.cpp:73 alignment; the caller's QI reference keeps it alive
    }
    fn LockServer(&self, flock: BOOL) -> windows_core::Result<()> {
        flock ? lock_increment() : lock_decrement(); Ok(())
    }
}

/// Builds a ClassFactory for a VxAPO CLSID; None for anything else.
pub fn create_factory(clsid: &GUID) -> Option<IClassFactory>;
```

## 7.4 `object/ref_count.rs`

**Responsibility**: `INST_COUNT` — the number of live APO COM object instances.

```rust
static INST_COUNT: AtomicU32 = AtomicU32::new(0);
/// Called from ClassFactory::CreateInstance / ApoObject::new. Returns the new value.
pub fn increment() -> u32;
/// Called from ApoObject::drop. saturating_sub (INST_COUNT only feeds DllCanUnloadNow).
pub fn decrement() -> u32;
pub fn get() -> u32;
pub fn is_zero() -> bool;
#[cfg(test)]
pub fn reset_for_test();
```

**`DllCanUnloadNow` rule**: `ref_count::is_zero() && factory::lock_is_zero()` → `S_OK`, else `S_FALSE`.

> **Test serialization (repo AGENTS.md §11)**: `INST_COUNT` and `LOCK_COUNT` are
> process-wide globals written by two test modules; both must take the shared
> `crate::object::ref_count::serial_lock()` as their first statement (a per-module lock
> still raced cross-module in release — measured).

## 7.5 `object/vx_reg_props.rs`

**Responsibility**: APO registration properties, query helpers and COM registration entries. The CLSIDs themselves are **defined once in `sys/consts.rs`** (code-verified single source; the zh sibling's local `pub const CLSID_VXAPO_*:` definitions are historical):

- PreMix `{{41C34613-D391-459D-A039-72B2B15A1A1D}}`
- PostMix `{{B4A97313-ABC0-45ED-9C33-428B20D39428}}`

**Registration properties** (verified construction — built by a `make_reg_props(CLSID)` helper rather than the zh struct literal):

```rust
pub static REG_PROPS_PRE_MIX: APO_REG_PROPERTIES = make_reg_props(CLSID_VXAPO_PRE_MIX);
pub static REG_PROPS_POST_MIX: APO_REG_PROPERTIES = make_reg_props(CLSID_VXAPO_POST_MIX);
// APO_FLAGS = FRAMESPERSECOND_MUST_MATCH | BITSPERSAMPLE_MUST_MATCH | INPLACE = 0x0D
// min/max connections 1/1, max_instances 1, APO_NAME "VxAPO", APO_COPYRIGHT "VxAPO Project"
// const fn str_to_u16_256(s: &str) -> [u16; 256] — compile-time UTF-16 name/copyright
```

**Query helpers** (verified):

```rust
pub fn props_for_clsid(clsid: &GUID) -> Option<&'static APO_REG_PROPERTIES>;
pub fn is_vxapo_clsid(clsid: &GUID) -> bool;
pub fn supported_clsids() -> &'static [GUID];
```

**COM registration entries**:

```rust
pub struct ClsidEntry { pub clsid: GUID, pub clsid_str: String }
impl ClsidEntry {
    pub fn new(clsid: GUID) -> Self;               // clsid_str = format!("{clsid}")
    pub fn clsid_key_path(&self) -> String;        // CLSID{GUID}
    pub fn inproc_server_path(&self) -> String;    // CLSID{GUID}InprocServer32
    pub fn audio_engine_path(&self) -> String;     // CLSID{GUID}AudioEngine (code-verified extra)
    pub fn registration_entries(&self, dll_path: &str) -> Vec<(&'static str, String, String)>;
}
/// Registration order: PostMix → PreMix (Note 29 — register PostMix first, roll back on failure).
pub fn registration_order() -> Vec<ClsidEntry>;
/// Unregistration order: PreMix → PostMix (reverse, Note 30).
pub fn unregistration_order() -> Vec<ClsidEntry>;
```

**Registry layout**: `HKCR\CLSID\{GUID}\InprocServer32` → `(Default) = "C:\...\vxapo.dll"`, `ThreadingModel = "Both"`.

## 7.6 `object/dll_exports.rs`

**Responsibility**: the DLL export functions. All are `#[unsafe(no_mangle)]` (edition-2024 attribute form; the zh `#[no_mangle]` spelling is the pre-2024 form).

**Global state**: `static MODULE_HANDLE: AtomicPtr<c_void>` — saved by `DllMain` on `DLL_PROCESS_ATTACH` (atomic write only; no COM init, no thread spawn, no lazy static forcing, no `#[ctor]` — everything deferred to the first `DllGetClassObject`/`CreateInstance`). Always returns `TRUE`; thread attach/detach ignored.

**Exports** (all verified):

```rust
pub unsafe extern "system" fn DllMain(h_module: HMODULE, reason: u32, _reserved: *mut c_void) -> BOOL;

pub unsafe extern "system" fn DllGetClassObject(rclsid: *const GUID, riid: *const GUID, ppv: *mut *mut c_void) -> HRESULT;
// flow: validate args → *ppv = null → create_factory(rclsid) (unknown →
// CLASS_E_CLASSNOTAVAILABLE) → QI riid → release the factory temp ref.
// ALSO: installs the telemetry logger (Logger::install, OnceLock — see the telemetry spec).

pub extern "system" fn DllCanUnloadNow() -> HRESULT;   // S_OK when INST_COUNT==0 && LOCK_COUNT==0, else S_FALSE

pub extern "system" fn DllRegisterServer() -> HRESULT;
// regsvr32 has no device context: this ONLY does global COM class registration
// (CoCreateInstance-ability). Device mounting / FxProperties binding belongs to
// install_endpoint (install spec 5.5.2, vxapo-cli install -d <device>).
// flow: get_dll_path (GetModuleFileNameW + MODULE_HANDLE; failure → SELFREG_E_CLASS)
// → registration_order() (PostMix → PreMix) → per CLSID create/open
// HKCRCLSID{GUID}InprocServer32, write (Default)=dll path + ThreadingModel="Both"
// (idempotent overwrite) → any failure rolls back in reverse order → SELFREG_E_CLASS; all → S_OK.
// FORBIDDEN: touching MMDevices/FxProperties; creating threads; initializing COM.

pub extern "system" fn DllUnregisterServer() -> HRESULT;
// deletes only the global COM class keys: unregistration_order() (PreMix → PostMix),
// InprocServer32 subkey first then CLSID{GUID}; missing keys count as success (idempotent);
// best-effort continues after individual failures → S_OK.

pub fn register_apo_with_path(dll_path: &str) -> HRESULT;
// CLI helper: refreshes the CLSID → DLL binding for an explicit path (re-registration).
```

**Internal helpers**: `get_dll_path()`, `register_com_class(entry, dll_path)`, `unregister_com_class(entry)`, `to_registry_bytes(s)` (null-terminated UTF-16).

## 7.7 `object/apo/aggregate.rs` — COM aggregation shell

**Responsibility** (code-verified; not covered by the zh sibling beyond a tree line): the engine **always** creates APOs aggregated (`pUnkOuter` non-null). windows-rs `#[implement]`'s self-contained IUnknown does not delegate, so aggregation would silently fail (object discarded → no sound). `NApo` implements the EAPO aggregation semantics with a hand-written multi-interface offset layout:

```rust
pub struct NApo { /* 4 independent vtable pointers: IAPO, RT, Config, IAudioSystemEffects
                     + inner ApoObject + refcount + outer pointer */ }
// NonDelegatingQI: IUnknown → outer identity; IAPO/RT/Config/ASE → the matching vtable
// field address (distinct this pointers — a shared this would give the engine a misaligned
// vtable, e.g. RT.APOProcess landing on IAPO.GetLatency → object discarded).
// AddRef/Release: self-managed (non-delegating, EAPO 541-555).
// Interface methods: forward to the inner ApoObject (macro forward_method!).

pub unsafe fn create_aggregate(p_unk_outer: *mut c_void, clsid: GUID) -> *mut c_void;
pub unsafe fn release_aggregate(obj: *mut c_void);

// Diagnostics: AGG_CREATED / AGG_DESTROYED AtomicU32 — instance-leak probes
// (settings-page stutter that vanished after restart was suspected to be a leak).
```

**Support modules** (all `pub(crate)`, verified):
- `apo/lock_key.rs` — `config_stamp(path) -> (mtime, size)` for the relock reuse key.
- `apo/rtdump.rs` — `rt_dump_open / rt_dump_push / rt_dump_flush`: RT diagnostics dump into
  a memory buffer, flushed to disk by the control thread at Unlock (no RT disk I/O).
- `apo/test_pipe.rs` — `notify(device_guid, stage, phase)`: best-effort named-pipe
  notification via the `DeviceTestPipeName` registry value under `SOFTWARE\VxAPO`
  (test instrumentation; dead-pipe caching avoids repeated open attempts).

## Hard constraints

1. `object/` is the only module allowed to depend on all other modules.
2. `object/apo/*` must not create circular dependencies with `object/apo.rs` except through the authorized entry/shell pattern.
3. RT paths must not allocate, block, or unwind.
4. Config hot reload must parse outside the lock and swap chain pointers inside the lock.




