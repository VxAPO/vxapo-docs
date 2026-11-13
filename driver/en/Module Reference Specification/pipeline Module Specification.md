# pipeline Module Specification

**Boundary**: Does not know `install/`, `config/`, `object/`. Only processes audio data; does not know who calls it.

**Allowed dependencies**: `sys/`, `utils/`.

**Forbidden dependencies**: `install/`, `config/`, `object/`.

## Data-flow architecture

**Normal mode** (APOProcess → process_audio):

```
Windows interleaved buffer (&[f32], from APO_CONNECTION_PROPERTY.p_buffer)
  │
  ▼
pipeline/process.rs — evaluate_buffer → extract input/output slices
  │
  ▼
pipeline/interleave.rs — deinterleave_into (interleaved → planar, zero allocation)
  │
  ▼
pipeline/chain.rs — call Filters one by one (planar space)
  │   └── filter.process(&mut [Vec<f32>], frame_count)
  │
  ▼
pipeline/interleave.rs — interleave_from (planar → interleaved, zero allocation)
  │
  ▼
Windows interleaved buffer (written back to APO_CONNECTION_PROPERTY.p_buffer)
```

**Transition mode** (APOProcess → process_chain_interleaved × 2 → mix_buffers):

```
Windows interleaved input (shared)
  │
  ├──▶ deinterleave_into ──▶ old_chain.process ──▶ interleave_from ──▶ temp_buffer_old
  │
  └──▶ deinterleave_into ──▶ new_chain.process ──▶ interleave_from ──▶ temp_buffer_new
                                                                        │
                                                                        ▼
                              mix_buffers(temp_old, temp_new, factor) ──▶ Windows output
```

> In transition mode the two `process_chain_interleaved` calls run **serially**, sharing the
> same set of planar work buffers (`temp_buffers`) and writing to separate interleaved output
> buffers. `transition.rs`'s `SmoothingProvider` only provides the mix factor; it does not
> manage chain state. Chain state is managed by `object/apo.rs`'s `ApoObjectInner` (object
> switching).

## Module tree

```
pipeline/
├── context.rs          # PipelineContext (static format info)
├── buffer.rs           # BufferInfo + buffer-state decision (BufferAction)
├── format.rs           # extract WAVEFORMATEX from IAudioMediaType
├── chain.rs            # Filter chain execution + latency accumulation
├── process.rs          # APOProcess dispatch entry + bridge functions + error policy
├── interleave.rs       # channel data movement (deinterleave/interleave)
├── realtime.rs         # real-time safety infrastructure module entry
├── realtime/
│   └── contract.rs     # RT context assertions (RtGuard / assert macros)
│                       # (the SPSC ring buffer lives in utils/ring.rs — see 4.8)
├── dsp.rs              # DSP algorithm module entry
└── dsp/
    ├── filter.rs       # Filter trait + DspContext + PassthroughFilter + ChannelScopedFilter (v9.11)
    ├── factory.rs      # v9.11 static dispatch create_from_model (match EffectType)
    ├── transition.rs   # SmoothingProvider + raised_cosine + mix_buffers
    ├── biquad.rs       # biquad filters
    ├── fir.rs          # SIMD dot / block FFT (v9.11)
    ├── model.rs        # ChainModel / EffectType (v9.11)
    ├── peq_hybrid.rs   # hybrid PEQ (200 Hz IIR + minimum-phase FIR, v9.11)
    ├── gain.rs         # gain (with internal smoothing interpolation)
    ├── loudness.rs     # ISO 226 equal-loudness curve
    ├── aural.rs        # Aural Enhancer (harmonic exciter)
    ├── compressor.rs   # Compressor (slow-axis loudness detection + negative-gain compression)
    ├── reverb.rs       # Dattorro plate reverb
    ├── wide.rs         # stereo widener
    ├── math.rs         # shared DSP math (dB conversion, stability, SIMD init, rate-limited warnings)
    └── specs.rs        # effect parameter spec tables (validation/defaults shared with config)
```

## Reference dependency table

> v9.17 (single source of truth): this table is no longer maintained independently — the main
> specification `Module Reference Specification (Without Detailed Modules).md`, section 11
> "Reference dependency table", is the only baseline; the per-file allowed and forbidden
> dependencies of pipeline are listed line by line there.

> **Key design decision**: `Chain` does not own buffers. Buffers are preallocated and held by
> `object/apo.rs`'s `ApoObjectInner`; `Chain` only accepts externally supplied planar buffer
> references to process. This guarantees:
> 1. **Buffer stability** — preallocated at `LockForProcess`, lifetime managed by `ApoObjectInner`
> 2. **process is non-blocking** — `Chain.process` is pure computation (Filter-by-Filter calls),
>    no locks, no allocation, no I/O
> 3. **Simple transition mode** — the two Chains share the same planar work buffers (used
>    serially), each producing its own interleaved output, mixed by `transition.rs`
> 4. **Hot-reload atomicity** — replacing a Chain is only a `Box<Chain>` pointer swap (inside
>    the mutex, sub-microsecond); buffers are unaffected

## 4.1 `pipeline/context.rs`

**Responsibility**: runtime context. Holds the static format info determined at `LockForProcess`; read-only during `APOProcess`.

**References**: no external dependencies.

**Exports to**: `pipeline/process.rs`, `pipeline/chain.rs`, `object/apo.rs`.

**Fields**:

```rust
pub struct PipelineContext {
    pub sample_rate: u32,
    pub input_channels: u32,
    pub output_channels: u32,
    pub channel_mask: u32,
    pub max_frame_count: usize,             // APO fitting upper bound (applies to both in/out)
    /// Actual bit depth at lock time (16/24/32); hot reload must match LockForProcess.
    pub bits_per_sample: u32,
}
impl Default for PipelineContext { /* new() — all zeros */ }
```

**Construction timing**: `LockForProcess` (object 7.1.9), after the format is negotiated. `APOProcess` only reads it.

## 4.2 `pipeline/buffer.rs`

**Responsibility**: `APO_CONNECTION_PROPERTY` raw pointer encapsulation + buffer validity/silence decision. **The only module allowed to hold the Windows raw pointer** (`sys/` provides the type definition; this module provides the safe encapsulation).

**References**:
- `crate::sys::com::apo_types::APO_CONNECTION_PROPERTY`
- `crate::pipeline::realtime::contract` (rt_index / RT assertions)

**Exports to**: `pipeline/process.rs`.

**Public API** (visibility verified against code — all `pub(crate)`, internal to the pipeline layer):

```rust
const SILENCE_THRESHOLD: f32 = 1e-10;

/// Windows buffer raw-pointer view (safe wrapper over APO_CONNECTION_PROPERTY).
pub(crate) struct BufferInfo {
    pub ptr: *mut f32,
    pub valid_frames: usize,
    pub flags: APO_BUFFER_FLAGS,
    pub channels: usize,
}

impl BufferInfo {
    /// Shared (input) buffer view.
    pub fn from_prop(prop: &APO_CONNECTION_PROPERTY, channels: usize) -> Self;
    /// Exclusive (output) buffer view.
    pub fn from_prop_mut(prop: &mut APO_CONNECTION_PROPERTY, channels: usize) -> Self;
    pub fn total_samples(&self) -> usize;
    /// # Safety: the caller must guarantee the pointer is valid and the length matches.
    pub unsafe fn as_slice(&self) -> &[f32];
    pub unsafe fn as_slice_mut(&mut self) -> &mut [f32];
    /// Zeroes the buffer (clamps frames to total capacity; does not trust the claimed count).
    pub fn zero(&mut self);
}

/// Buffer handling decision (v9.14 refactor: consumed by process_audio).
pub(crate) enum BufferAction { /* Process / ZeroOutputs / SkipInvalid / ... */ }

/// Unified entry: validates the buffer (pointer non-null, channels>0, frames>0) and
/// classifies silence (peak below SILENCE_THRESHOLD = 1e-10).
pub(crate) fn evaluate_buffer(/* info: &mut BufferInfo, allow_silent: bool */) -> BufferAction;

pub(crate) fn is_silent(samples: &[Vec<f32>], frame_count: usize) -> bool;
pub(crate) fn zero_buffers(buffers: &mut [Vec<f32>], frame_count: usize);
pub(crate) fn copy_buffers(src: &[Vec<f32>], dst: &mut [Vec<f32>], frame_count: usize);
```

> **Silence short-circuit** (`evaluate_buffer` → `BufferAction`): a silent input writes zeros
> to the outputs and skips the DSP chain entirely (only when allowed — the transition period
> must keep processing so the mix factor converges; see process 4.6). The historical
> `BufferSummary` return type was removed in v9.14 (a decision enum carries the same
> information with less data).

## 4.3 `pipeline/format.rs`

**Responsibility**: extract WAVEFORMATEX/WAVEFORMATEXTENSIBLE from `IAudioMediaType` (COM interface), convert to the internal `AudioFormat` struct. **The only module allowed to hold the `IAudioMediaType` raw pointer.**

**References**:
- `windows::Win32::Media::Audio::Apo::IAudioMediaType`
- `crate::sys::com::apo_types::*`

**Exports to**: `object/apo.rs` (LockForProcess), `object/apo/child.rs`.

**Public API**:

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct AudioFormat {
    pub sample_rate: u32,
    pub channels: u16,
    pub bits_per_sample: u16,
    pub channel_mask: u32,
    pub is_float: bool,
    /// Conversion choice at LockForProcess (v9.11/v9.20):
    /// Convert (32f <-> 24/16-bit int) or Passthrough (already 32-bit float).
    pub conversion: FormatConversion,
}

/// # Safety: media_type must be a valid IAudioMediaType pointer (engine-provided).
pub unsafe fn extract_format(media_type: *mut IAudioMediaType) -> Result<AudioFormat>;

/// # Safety: media_type must be a valid IAudioMediaType pointer (engine-provided).
pub unsafe fn is_float_format(media_type: *mut IAudioMediaType) -> Result<bool>;
```

> Both functions are `unsafe` in the current code (FFI pointer dereference); the SAFETY
> contract is "the engine-provided pointer is valid for the call duration". `is_float_format`
> returns `Result<bool>` (errors propagate, no silent default).

## 4.4 `pipeline/interleave.rs`

**Responsibility**: planar ↔ interleaved channel data movement. All functions are zero-allocation `memcpy` loops (performance-critical path).

**References**: no external dependencies (pure functions).

**Exports to**: `pipeline/process.rs`, `object/apo.rs`.

**Public API**:

```rust
/// Interleaved → planar (allocating version; not for the RT path).
pub fn deinterleave(input: &[f32], channels: usize, frames: usize) -> Vec<Vec<f32>>;

/// Planar → interleaved (allocating version; not for the RT path).
pub fn interleave(channels: &[Vec<f32>], frames: usize) -> Vec<f32>;

/// Interleaved → planar, writing into preallocated buffers (RT path, zero allocation).
pub fn deinterleave_into(input: &[f32], output: &mut [Vec<f32>], channels: usize, frames: usize);

/// Planar → interleaved, writing into a preallocated buffer (RT path, zero allocation).
pub fn interleave_from(input: &[Vec<f32>], output: &mut [f32], channels: usize, frames: usize);

/// Planar → interleaved with tail protection (v9.15): copies only valid samples and zeroes
/// any remainder beyond frame_count, so stale data from previous blocks never leaks into
/// the output when the engine passes a smaller valid frame count than the buffer capacity.
pub fn interleave_from_guarded(/* input, output, channels, valid_frames, total_frames */);
```

## 4.5 `pipeline/chain.rs`

**Responsibility**: ordered `Filter` chain execution + latency accumulation + channel-slot dispatch (v9.11 channel selection). Does not own buffers.

**References**:
- `crate::pipeline::dsp::filter::Filter`

**Exports to**: `object/apo.rs` (`ApoObjectInner.current_chain` / `transition_chain`).

**Public API**:

```rust
pub struct Chain {
    filters: Vec<Box<dyn Filter>>,
    total_latency: u32,
}

impl Chain {
    pub fn new() -> Self;
    /// Adds a filter; rejects more than MAX_CHAIN_FILTERS (32).
    pub fn add_filter(&mut self, filter: Box<dyn Filter>) -> Result<()>;
    /// Calls filter.initialize in order and applies channel-slot dispatch
    /// (set_channel_indices for channel-scoped filters).
    pub fn initialize(&mut self, sample_rate: u32, channel_names: &[String]);
    /// Sums filter.latency() (cached at initialize; not re-computed per block).
    pub fn total_latency(&self) -> u32;
    pub fn filter_count(&self) -> usize;
    pub fn is_empty(&self) -> bool;
    /// Executes the chain in order on planar buffers (RT path: no alloc, no locks, no I/O).
    pub fn process(&mut self, samples: &mut [Vec<f32>], frame_count: usize);
    /// Resets every filter's internal state (order preserved).
    pub fn reset(&mut self);
    /// Whether every filter accepts this frame count (max_frame_count constraint).
    pub fn validate_frame_count(&self, frame_count: usize) -> bool;
}

impl Default for Chain { /* new() */ }
```

**Channel-slot dispatch** (`initialize`, v9.11): for each filter, if
`fixed_channel_indices()` returns `Some` (`ChannelScopedFilter` only) those slots are used
directly; otherwise the slots are derived from the filter's declared output channel names
(`initialize` return value) mapped against `channel_names`. `set_channel_indices` is called
before the filter's own `initialize` result is consumed, so `Channel:`-style selection and
config-model `channels` scoping share one mechanism.

> **Removed**: the old `is_fully_in_place()` helper no longer exists (0 hits). All built-in
> filters are in-place; the E1 fast-path note below is historical context, not a live API.
>
> **`is_in_place` and the `process_audio` fast path (E1, historical)**: when the whole chain
> reports `is_in_place() == true`, the planar work buffer `temp_buffers` is both chain input
> and chain output — no intermediate copies between filters (the existing flow is naturally
> zero-copy). All current built-in filters (including the Delay ring buffer and the
> Convolution skeleton) safely declare `true`.

## 4.6 `pipeline/process.rs`

**Responsibility**: `APOProcess` dispatch entry — error policy, silence short-circuit, normal/transition processing bridges. **This is the sole RT entry aggregation point** (called by `object/apo/process.rs::apo_process`).

**References**:
- `crate::pipeline::buffer::{self, BufferInfo, BufferAction}`
- `crate::pipeline::chain::Chain`
- `crate::pipeline::interleave`
- `crate::pipeline::dsp::filter::{DspContext, Filter}`
- `crate::pipeline::dsp::transition::{SmoothingProvider, mix_buffers, mix_plane_buffers}`
- `crate::pipeline::realtime::contract` (RT context assertions)

**Exports to**: `object/apo.rs` / `object/apo/process.rs`.

**Public API** (signatures verified against code):

```rust
/// Error policy (E6).
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ErrorPolicy {
    /// Copy input → output unchanged (safe degradation; the default choice for
    /// state errors).
    Bypass,
    /// Zero the outputs (mute). Dead cluster in the current code
    /// (#[allow(dead_code)]): production constructs Bypass only; Silence remains the
    /// documented policy for RT-safety breaches (never pass unprocessed audio then).
    Silence,
}

pub struct ProcessParams {
    pub input_channels: u32,
    pub output_channels: u32,
    pub sample_rate: u32,
    pub max_frame_count: usize,
    pub valid_frame_count: usize,
    pub error_policy: ErrorPolicy,
}
// Currently a dead cluster (#[allow(dead_code)]): kept as the documented parameter shape;
// the live process_audio path takes its parameters from the object layer directly.

/// Per-call processing statistics (for diagnostics; not on the RT hot path).
pub struct ProcessStatistics { /* counters */ }

impl ProcessStatistics {
    pub fn new() -> Self;
}
impl Default for ProcessStatistics { /* new() */ }

/// Internal: plain sample-copy bridge (pub(crate)).
pub(crate) fn bypass_copy(src: &[f32], dst: &mut [f32]);

/// Applies the error policy to a raw interleaved buffer.
pub fn apply_error_policy(/* error, input slice, output slice, policy */) -> ();

/// Transition bridge: one chain pass over interleaved buffers
/// (deinterleave → chain.process → interleave), zero allocation.
pub fn process_chain_interleaved(
    /* chain: &mut Chain, input: &[f32], output: &mut [f32],
       temp_buffers: &mut [Vec<f32>], frame_count: usize, channels: usize, ctx: &DspContext */
);

/// Normal-path entry: buffer evaluation (validity/silence) → deinterleave → chain →
/// interleave → error policy. Silent inputs zero the outputs and skip the DSP chain
/// (except during an active transition, which must keep processing to converge).
pub fn process_audio(/* ... */);
```

**Processing pseudocode** (normal path):

```text
process_audio(input_prop, output_props, chain, temp_buffers, params, stats):
    1. BufferInfo::from_prop / from_prop_mut wrap the Windows pointers
    2. evaluate_buffer → validity + silence classification
       - invalid  → apply_error_policy (skip / bypass / silence per policy)
       - silent   → zero outputs (skip DSP) unless a transition is active
    3. deinterleave_into(input_slice → temp_buffers)          [zero allocation]
    4. chain.process(&mut temp_buffers, frame_count)          [pure computation]
    5. interleave_from_guarded(temp_buffers → output_slice)   [zero allocation]
    6. update statistics
```

**Thread safety**: `process_audio` / `process_chain_interleaved` are `&mut`-driven pure
functions — no shared state, no locks, no allocation, no I/O. RT context is asserted by the
caller (`object/apo/process.rs` creates an `RtGuard` and `rt_assert_in_rt!` guards the core).

**Panic policy**: the RT path must not panic; slice indexing on the hot path goes through
`rt_index` / `rt_index_mut` (checked, non-panicking contract helpers — see 4.7). Any bug
that would panic is instead caught by the object layer's error policy (E6: Bypass/Silence).

## 4.7 `pipeline/realtime/contract.rs` (v6.6; wired into production)

**Responsibility**: compile-time + runtime RT-context contracts. The RT path may only be entered through the RT entry; non-RT paths (locks, file I/O) assert they are **not** in RT context.

**References**: no external dependencies (core only).

**Exports to**: `pipeline/` (buffer/process), `object/apo/{process,reload}.rs`.

**Public API** (verified against code):

```rust
/// Types safe to use on the RT path (no heap, no locks, no syscalls).
pub unsafe trait RtSafe: Send + Sync {}

/// Types safe to copy on the RT path (plain POD).
pub unsafe trait RtCopy: Copy {}

/// Zero-sized compile-time witness for the real-time context.
pub struct RealtimeContext;

/// Marks the current thread as running in RT context (thread-local depth counter).
pub fn enter_rt_context();
/// Leaves the RT context (decrements the thread-local depth).
pub fn exit_rt_context();
/// Whether the current thread is inside an RT context.
pub fn is_rt_context() -> bool;

/// RAII RT guard: enter_rt_context() on new(), exit on Drop.
/// Nested guards are supported (depth counting); an inner Drop keeps the outer context.
pub struct RtGuard;
impl RtGuard {
    pub fn new() -> Self;
}
impl Drop for RtGuard { /* exit_rt_context() */ }

/// Asserts the current thread IS in RT context (debug builds panic otherwise).
rt_assert_in_rt!();

/// Asserts the current thread is NOT in RT context (debug builds panic otherwise) —
/// used by non-RT paths that take locks or do file I/O (e.g. hot_reload_impl).
rt_require_non_rt!();

/// Asserts the current thread is NOT in RT context (alias form).
rt_assert_not_in_rt!();

/// Checked indexing for the RT path (contract helpers; RT code must not panic).
pub unsafe fn rt_index<T>(slice: &[T], index: usize) -> &T;
pub unsafe fn rt_index_mut<T>(slice: &mut [T], index: usize) -> &mut T;
```

> **Implementation note (verified)**: the context is a **thread-local depth counter**, not a
> global `AtomicBool`. This fixes two historical defects: (1) a nested `RtGuard`'s Drop no
> longer clears the outer context; (2) a concurrent watcher thread no longer sees a leaked
> global flag and raises a **false** `rt_require_non_rt!` violation. Regression tests:
> `rt_context_is_thread_local`, `rt_depth_does_not_underflow`, `rt_guard_nested`.
> In release builds all assertions compile to no-ops (zero overhead).

> **Wiring**: `object/apo/process.rs::apo_process` creates an `RtGuard` at the RT entry;
> `apo_process_inner` runs under `rt_assert_in_rt!`;
> `object/apo/reload.rs::hot_reload_impl` (locks / file I/O) runs under `rt_require_non_rt!`.

## 4.8 The SPSC lock-free ring buffer — `utils/ring.rs`

> **Path correction (code-verified)**: older documents (and the zh sibling) place the ring at
> `pipeline/realtime/ring.rs`; the real file is **`utils/ring.rs`**, and
> `telemetry/logger.rs` imports `crate::utils::ring::RingBuffer`.

**Responsibility**: SPSC lock-free ring buffer for fixed-size `Copy + Default` items. Power-of-two capacity, atomic head/tail, no locks.

**References**: core only (`UnsafeCell`, `AtomicUsize`).

**Exports to**: `telemetry/logger.rs` (`RingBuffer<LogEntry>`).

**Public API** (verified against code):

```rust
/// SPSC lock-free ring buffer storing fixed-size, Copy + Default items.
pub struct RingBuffer<T: Copy + Default> {
    // UnsafeCell<Box<[T]>> + capacity + AtomicUsize write_pos / read_pos
}

// SAFETY: SPSC design — push/pop are called from different threads; the atomics provide
// the memory synchronization. &self never leaks cross-thread aliases (T: Copy).
unsafe impl<T: Copy + Default> Send for RingBuffer<T> {}
unsafe impl<T: Copy + Default> Sync for RingBuffer<T> {}

impl<T: Copy + Default> RingBuffer<T> {
    /// Creates a ring with the given capacity (rounded up to a power of two, min 2).
    pub fn new(capacity: usize) -> Self;

    /// Pushes one item (producer side). Returns false when full — never blocks,
    /// never allocates; the caller decides the drop policy.
    pub fn push(&self, value: T) -> bool;

    /// Pops one item (consumer side). Returns None when empty — never blocks.
    /// Test-only in the current code: production only pushes (the logger ring keeps
    /// the earliest entries and drops the newest ones once full).
    pub fn pop(&self) -> Option<T>;

    /// Dead cluster (allow(dead_code)).
    pub fn capacity(&self) -> usize;
}
```

> **Thread-safety contract**: exactly one producer thread and one consumer thread.
> `push` uses Relaxed load + Acquire check + Release store (and vice versa for `pop`);
> the `UnsafeCell` slot access is justified by the SPSC protocol in the SAFETY comments.
> When full, `push` returns false — the logger then drops the **newest** message, keeping
> the earliest diagnostics.

## 4.9 `pipeline/dsp/filter.rs`

**Responsibility**: the `Filter` trait (the DSP processing abstraction) + `DspContext` (engine context) + `PassthroughFilter` + `ChannelScopedFilter` (v9.11 per-effect channel scoping).

**References**:
- `crate::pipeline::realtime::contract::{RealtimeContext, RtSafe}`

**Exports to**: `pipeline/chain.rs`, `pipeline/dsp/factory.rs`, `config/` (trait + DspContext only), `object/apo.rs`.

**Public API** (verified against code):

```rust
pub trait Filter: Send + Sync + std::fmt::Debug {
    /// Processes one block of audio (real-time path, planar space).
    /// samples[channel][frame], modified in place.
    /// Must not heap-allocate, do I/O, or panic.
    fn process(&mut self, samples: &mut [Vec<f32>], frame_count: usize);

    /// Initializes the filter.
    /// Returns the output channel-name list (Some = channel selection, None = unchanged).
    /// Performs no I/O and must not fail.
    fn initialize(&mut self, sample_rate: u32, channel_names: &[String]) -> Option<Vec<String>>;

    /// Whether this is a Channel:-type command (channel-selection marker). Default false.
    /// Test-only in the current code (#[cfg(test)]).
    fn is_channel_select(&self) -> bool { false }

    /// Sets the planar channel slots this filter acts on (called by Chain::initialize
    /// after a Channel: selection). indices are slots in the original planar buffer
    /// (e.g. Channel: R → [1]). Filters that process "the first N slots"
    /// (biquad/gain/delay/convolution...) must store the list and process only those
    /// slots in process; filters without channel semantics (Copy etc.) ignore it.
    fn set_channel_indices(&mut self, _indices: &[usize]) {}

    /// Fixed channel slots from the config model's per-effect channels field.
    /// Some → Chain skips auto-computation and uses the fixed slots (only
    /// ChannelScopedFilter returns Some); None → Chain derives slots from the
    /// current channel names.
    fn fixed_channel_indices(&self) -> Option<Vec<usize>> { None }

    /// Whether processing is in place. Default true. Dead cluster
    /// (allow(dead_code)): all built-in filters return true; kept as the documented
    /// extension point for future non-in-place effects.
    fn is_in_place(&self) -> bool { true }

    /// Filter latency in samples. Default 0.
    fn latency(&self) -> u32 { 0 }

    /// Maximum frame-count constraint. Default None. Dead cluster
    /// (allow(dead_code)): Some(n) means process handles at most n frames per call.
    fn max_frame_count(&self) -> Option<usize> { None }

    /// Resets internal filter state. Default no-op.
    fn reset(&mut self) {}
}

/// Pass-through filter — modifies no samples.
#[derive(Debug, Clone, Copy)]
pub struct PassthroughFilter;
impl Filter for PassthroughFilter { /* no-op process; initialize → None */ }

/// Wraps a filter with fixed per-effect channels from the config model (v9.11).
/// The config layer resolves the planar slots and wraps once; Chain::initialize
/// recognizes it via fixed_channel_indices and skips auto channel computation.
pub struct ChannelScopedFilter { /* inner: Box<dyn Filter>, indices: Vec<usize> */ }
impl ChannelScopedFilter {
    pub fn new(inner: Box<dyn Filter>, indices: Vec<usize>) -> Self;
}
impl Filter for ChannelScopedFilter { /* delegates everything; fixed_channel_indices → Some */ }
```

**DspContext** (the struct carries a dead-cluster annotation; it remains the documented engine context):

```rust
/// Unified engine context. Pure data; contains no Windows types.
/// Built by the caller from PipelineContext + extra parameters (the caller depends on
/// pipeline/context.rs). filter.rs does not depend on pipeline/context.rs.
#[derive(Debug, Clone)]
pub struct DspContext {
    pub sample_rate: u32,
    pub channel_count: u32,
    pub channel_mask: u32,
    pub channel_names: Vec<String>,              // channel-name list (Copy: command)
    pub max_frame_count: u32,                    // Convolution preallocation depends on it
    pub bits_per_sample: u32,
    pub device_type: DeviceType,
    pub stage: ProcessingStage,
    pub variables: HashMap<String, f64>,         // Eval: command variable store
    pub loudness_enabled: std::cell::Cell<bool>, // loudness effect enable flag (Cell: not Send)
    pub rt_marker: PhantomData<RealtimeContext>, // RT compile-time witness (v6.6)
}

/// Endpoint direction. Capture is currently a dead cluster (allow(dead_code)).
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum DeviceType { Render, Capture }

/// Processing stage. Only None is live; PreMix / PostMix / Capture are dead clusters
/// (allow(dead_code)) kept for the documented slot semantics.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ProcessingStage { None, PreMix, PostMix, Capture }
```

> **`rt_marker` (O1: RT compile-time witness)**: the `PhantomData<RealtimeContext>`
> zero-sized field moves the "this context serves the real-time path" semantic to
> **compile time**. `DspContext` is constructed by `object/apo.rs` on the non-RT path, but
> its methods (filter creation/initialization) all serve the RT `process`; `rt_marker` lets
> pipeline-internal methods require `&RealtimeContext` so misuse is a compile error.
> Evolution direction: new code declares `fn f(&self, rt: &RealtimeContext, ...)`; legacy
> `rt_assert_in_rt!` paths migrate batch by batch. Constructing `RealtimeContext` outside
> the RT path is forbidden (no public constructor).

> **Caller construction example** (`object/apo.rs` LockForProcess): builds `DspContext` from
> `PipelineContext` (sample_rate, input_channels, channel_mask, max_frame_count,
> channel_names, bits_per_sample: 32, DeviceType::Render, ProcessingStage::None,
> variables: HashMap::new()).

### Historical: FilterCreateResult / FilterFactory / ConfigLoader (pre-v9.11, removed)

> The dynamic factory registry below belonged to the pre-v9.11 text-command system.
> Since v9.11 the only factory is `factory::create_from_model` static dispatch.
> Kept as historical reference.

```rust
/// Factory creation result (four states).
#[derive(Debug)]
pub enum FilterCreateResult {
    Filter(Box<dyn Filter>),  // matched, produced a filter instance
    NoFilter,                 // matched, no filter needed (Device: match, Include: loaded)
    NoMatch,                  // no match, try the next factory
    AbortFile,                // stop parsing the current file (Device: mismatch)
}

/// Filter factory trait: one implementation per config command; the first factory
/// returning Filter or NoFilter wins.
pub trait FilterFactory: Send {
    fn create_filter(&self, params: &str, ctx: &DspContext, loader: &dyn ConfigLoader)
        -> FilterCreateResult;
    fn command_name(&self) -> &str;
}

/// Config-loader callback trait: the Include: command recurses into sub-config files
/// through it (cmd_include.rs must not import parser.rs directly).
pub trait ConfigLoader {
    /// Loads the config file at path; returns the filters it produced.
    /// Missing file or parse failure → empty list (degrade to passthrough).
    fn load_config(&self, path: &str, ctx: &DspContext) -> Vec<Box<dyn Filter>>;
}
```

## 4.10 `pipeline/dsp/factory.rs` (v9.11)

**Responsibility**: since v9.11, **model → Filter static dispatch**: `create_from_model(EffectConfig, DspContext) -> Box<dyn Filter>`, a `match EffectType` exhaustive over 7 types. The old `FilterFactory` / `FilterRegistry` / `FilterCreateResult` / `OutcomeKind` / `index` constants and registration-order tests are deleted (no longer EAPO-aligned).

**References**:
- `crate::pipeline::dsp::filter::*`
- `crate::pipeline::dsp::model::*` (EffectConfig / EffectType, v9.11)
- `crate::pipeline::dsp/*.rs` — **the necessary exception**: the factory must name concrete
  filter types to instantiate them
- `crate::utils::vx_error::VxApoError`

**Exports to**: `config/parser.rs`, `object/apo.rs`.

**Public API**:

```rust
/// Builds one filter from a validated EffectConfig.
/// Unknown EffectType is impossible by construction (the enum is closed);
/// param extraction is a per-type match with no fallthrough.
pub fn create_from_model(config: &EffectConfig, ctx: &DspContext)
    -> Result<Box<dyn Filter>, crate::utils::vx_error::VxApoError>;
```

> Dispatch table (v1 effect set): peq → HybridPeqFilter, preamp → GainFilter,
> aural → AuralFilter, reverb → ReverbFilter, compressor → CompressorFilter,
> wide → WideFilter, loudness → LoudnessFilter. Legacy maximizer / leveler map to
> compressor defaults at the config layer before reaching the factory.

### Historical: FilterRegistry (pre-v9.11, removed)

> The registry below is the pre-v9.11 dynamic implementation, deleted; the live behavior is
> the `create_from_model` static match above. Kept as historical reference.

```rust
pub struct FilterRegistry { /* factories: Vec<Box<dyn FilterFactory>> */ }
impl FilterRegistry {
    pub fn new() -> Self;
    pub fn register(&mut self, factory: Box<dyn FilterFactory>);
    pub fn len(&self) -> usize;
    pub fn is_empty(&self) -> bool;
    pub fn get(&self, index: usize) -> Option<&dyn FilterFactory>;
    pub fn factory_names(&self) -> Vec<&str>;
    /// Tries factories in order; the first returning Filter or NoFilter wins.
    pub fn try_create(&self, params: &str, ctx: &DspContext, loader: &dyn ConfigLoader)
        -> TryCreateOutcome;
    /// v9.1 exact dispatch by command name (case-insensitive); Unmatched when no factory
    /// matches — the parser default branch uses this so tolerant factories cannot swallow
    /// invalid parameters of known commands.
    pub fn try_create_named(&self, command: &str, params: &str, ctx: &DspContext,
        loader: &dyn ConfigLoader) -> TryCreateOutcome;
}

#[derive(Debug)]
pub struct TryCreateOutcome {
    pub result: OutcomeKind,
    pub factory_index: Option<usize>,
    pub factory_name: Option<String>,
}
pub enum OutcomeKind {
    FilterAdded(Box<dyn Filter>), MatchedNoFilter, Aborted, Unmatched,
}

pub const FACTORY_COUNT: usize = 19;
pub mod index { /* DEVICE IF EVAL INCLUDE STAGE CHANNEL IIR BIQUAD PREAMP DELAY
                     COPY CONVOLUTION GRAPHIC_EQ VST_PLUGIN LOUDNESS_CORRECTION
                     AURAL_ENHANCER REVERB MAXIMIZER WIDE */ }
pub fn create_default_registry() -> Vec<Box<dyn FilterFactory>>;
pub fn register_builtin_filters(registry: &mut FilterRegistry);
```

> **Registration order (v9.2)**: IIR → Biquad → Preamp → Delay → Copy → Convolution →
> GraphicEQ → VSTPlugin → LoudnessCorrection → AuralEnhancer → Reverb → Maximizer → Wide.
> The EAPO-style command syntax (AuralEnhancer: / Reverb: / Maximizer: / Wide:) and the
> factory registration order no longer exist; the current configuration is TOML
> `[[effects]]` — effects and parameters are in section 4.22.

## 4.11 `pipeline/dsp/transition.rs`

**Responsibility**: raised-cosine mixing + transition-factor generation (pure algorithms; no state machine). `mix_buffers` operates on interleaved buffers.

**References**: no external dependencies.

**Exports to**: `object/apo.rs`, `pipeline/process.rs`.

**Public API** (verified against code):

```rust
pub fn raised_cosine(counter: u32, length: u32) -> f32;

pub struct SmoothingProvider { /* counter, length */ }
impl SmoothingProvider {
    pub fn new(length: u32) -> Self;
    pub fn begin(&mut self);
    /// Next mix factor, or None when the transition has finished.
    pub fn advance(&mut self) -> Option<f32>;
    pub fn is_active(&self) -> bool;
    pub fn length(&self) -> u32;
    pub fn counter(&self) -> u32;
    pub fn progress(&self) -> f32;
    pub fn reset(&mut self);
    pub fn set_length(&mut self, length: u32);
}

/// sample_rate / 100 ≈ 10 ms (v6.9, EqualizerAPO aligned).
pub fn default_smoothing_length(sample_rate: u32) -> u32;
/// Same formula as the default; kept as a semantic alias.
pub fn short_smoothing_length(sample_rate: u32) -> u32;

/// Interleaved-buffer mix: output = old × (1 - factor) + new × factor
///
/// # Safety
/// - old / new / output must point to valid f32 buffers of length ≥ count
/// - the three buffers must not overlap
/// - factor must be in [0.0, 1.0]
pub unsafe fn mix_buffers(
    old: *const f32, new: *const f32, output: *mut f32,
    count: usize, factor: f32,
);

/// Planar-buffer variant of the mix (same semantics, planar slices).
pub fn mix_plane_buffers(/* old, new, output: planar slices, frame_count, factor */);
```

> **Transition period (R4, v6.9)**: the default smoothing length was lowered from
> `sample_rate/20` (50 ms) to `sample_rate/100` (10 ms), aligned with EqualizerAPO:
> - during the transition every frame is processed twice (dual-chain model); 10 ms keeps
>   the CPU peak shortest and RT stability best
> - a 10 ms raised cosine has been proven inaudible in EAPO for years
> - "chopping at will is fine" — the transition is short and cheap, so frequent config
>   changes are no longer a burden.
>
> In `object/apo.rs` transition mode, `temp_buffer_old` / `temp_buffer_new` are already in
> interleaved space, produced by `process_chain_interleaved`.

## 4.12 `pipeline/dsp/biquad.rs`

**Responsibility**: biquad filter implementation (RBJ cookbook coefficients, DF2T production path).

**References**: `crate::pipeline::dsp::filter::Filter`, `crate::utils::vx_error::VxApoError`.

**Exports to**: `pipeline/dsp/peq_hybrid.rs`, `pipeline/dsp/loudness.rs`, `pipeline/dsp/wide.rs` (the old `peq.rs` / `hp_lp.rs` are deleted; not exported to `config/`).

**Public API** (verified against code):

```rust
pub struct BiquadCoeffs {
    pub b0: f32, pub b1: f32, pub b2: f32,
    pub a1: f32, pub a2: f32,
}
impl BiquadCoeffs {
    /// Identity coefficients (b0 = 1, rest 0).
    pub const BYPASS: Self;
    pub fn is_valid(&self) -> bool;
}
// NOTE (code-verified): the per-type design constructors (lowpass/highpass/peak/...)
// listed in the zh sibling do not exist in code — design goes through compute_coeffs.

pub enum BiquadType {
    Peaking, LowShelf, HighShelf, LowPass, HighPass,
    BandPass,  // dead cluster (allow(dead_code)); implemented + test-covered
    Notch,     // dead cluster
    AllPass,   // dead cluster
}

/// Computes biquad coefficients (Robert Bristow-Johnson Audio EQ Cookbook).
/// - fc auto-clamps to [FILTER_FREQ_MIN_HZ, 0.45 × sample_rate]
/// - gain_db passes clamp_gain_db: Peaking / LowShelf / HighShelf use it as the target
///   level; LowPass / HighPass scale it into the passband (0 dB keeps pure filtering)
/// - q auto-clamps to [Q_MIN, Q_MAX]; sample_rate 0 → bypass
/// - all formulas use f64 intermediates, converted to f32 at the end; after
///   normalization a finiteness + pole-stability check falls back to BYPASS (P0 guard)
pub fn compute_coeffs(
    filter_type: BiquadType, fc: f32, gain_db: f32, q: f32, sample_rate: u32,
) -> BiquadCoeffs;

pub enum BiquadStructure {
    DirectFormI,             // dead cluster (test path; two delay lines)
    DirectFormII,            // dead cluster (test path; one delay line)
    DirectFormIITransposed,  // production (numerically most stable; EAPO default)
}

/// Per-channel DF2T state (f64, FMA, NaN guard).
pub struct BiquadState { /* z1, z2 */ }
impl BiquadState {
    pub fn new() -> Self;
    pub fn clear(&mut self);
    pub fn process_sample(&mut self, coeffs: &BiquadCoeffs, input: f32) -> f32;
}

/// Biquad filter — independent state per channel.
/// DF1 / DF2 are compatibility/test paths (f32 state arrays); DF2T is the production
/// path (Vec<BiquadState>, f64, FMA, NaN fallback).
pub struct BiquadFilter { /* coeffs, structure, num_channels, channel_indices, state */ }
impl BiquadFilter {
    pub fn new(coeffs: BiquadCoeffs, structure: BiquadStructure) -> Self;
    pub fn set_coeffs(&mut self, coeffs: BiquadCoeffs);
    pub fn coeffs(&self) -> BiquadCoeffs;
    pub fn reset_state(&mut self);
}
impl Filter for BiquadFilter { /* processes only its channel slots (channel_indices) */ }
```

---

> **Obsolete since v9.11**: the files described by 4.13 / 4.14 / 4.16 / 4.17 / 4.18 / 4.19 /
> 4.20 **have all been deleted from the code** — PEQ is now `pipeline/dsp/peq_hybrid.rs`,
> biquad/filtering live in `biquad.rs` / `filter.rs`, convolution and delay are removed.
> Those sections keep their struck-through titles as historical reference; **the current
> implementation is section 4.22**. Current file list (2026-09): `aural biquad compressor
> factory filter fir gain loudness math model peq_hybrid reverb specs transition wide`
> (15 files).

### 4.13 ~~`pipeline/dsp/peq.rs`~~ (deleted)

**Responsibility**: parametric EQ (cascaded biquads). **References**: Filter + biquad. **Exports to**: only the old `graphic_eq.rs`.

```rust
pub struct PeqFilter { ... }
impl PeqFilter {
    pub fn new() -> Self;
    pub fn add_band(&mut self, freq: f32, gain_db: f32, q: f32, sample_rate: f32);
}
impl Filter for PeqFilter { ... }
```

### 4.14 ~~`pipeline/dsp/hp_lp.rs`~~ (deleted)

**Responsibility**: high-pass / low-pass filters. **References**: Filter + biquad. **Exports to**: only `pipeline/dsp/` internally.

```rust
pub struct HighLowPassFilter { ... }
impl HighLowPassFilter {
    pub fn new(filter_type: BiquadType, fc: f32, q: f32) -> Self;
}
impl Filter for HighLowPassFilter { ... }
```

## 4.15 `pipeline/dsp/gain.rs`

**Responsibility**: gain filter (with internal smoothing interpolation).

**References**: `crate::pipeline::dsp::filter::Filter`.

**Exports to**: only `pipeline/dsp/` internally (constructed via `factory::create_from_model`).

**Public API** (verified against code — the zh sibling's `set_gain` / `set_smoothing` do not exist; smoothing is internal and fixed):

```rust
pub struct PreampParams {
    pub gain_db: f32,
}

pub struct GainFilter { /* target/linear gain + smoothing state */ }
impl GainFilter {
    pub fn new(gain_db: f32) -> Self;
    /// Sets the target gain in dB (smoothed internally, no zipper noise).
    pub fn set_gain_db(&mut self, gain_db: f32);
    /// Sets the target linear gain directly (smoothed internally).
    pub fn set_gain_linear(&mut self, target: f32);
    pub fn current_gain_linear(&self) -> f32;
}
impl Filter for GainFilter { ... }

pub fn db_to_linear(db: f32) -> f32;
pub fn linear_to_db(linear: f32) -> f32;
```

### 4.16 ~~`pipeline/dsp/delay.rs`~~ (deleted)

**Responsibility**: delay line (ring-buffer implementation). **References**: Filter. **Exports to**: only `pipeline/dsp/` internally.

```rust
pub struct DelayFilter { ... }
impl DelayFilter {
    pub fn new(delay_ms: f32) -> Self;
    pub fn set_delay(&mut self, delay_ms: f32, sample_rate: f32);
    pub fn set_mix(&mut self, dry: f32, wet: f32);
}
impl Filter for DelayFilter { ... }
```

### 4.17 ~~`pipeline/dsp/copy.rs`~~ (deleted)

**Responsibility**: channel copy / mixdown. **References**: Filter + interleave. **Exports to**: only `pipeline/dsp/` internally.

```rust
pub struct CopyFilter { ... }
impl CopyFilter {
    pub fn new(mappings: Vec<ChannelMapping>) -> Self;
}
impl Filter for CopyFilter { ... }

pub struct ChannelMapping {
    pub src: String,
    pub dst: String,
    pub gain: f32,
}

pub fn parse_copy_ops(spec: &str, channel_names: &[String]) -> Option<Vec<ChannelMapping>>;
```

### 4.18 ~~`pipeline/dsp/graphic_eq.rs`~~ (deleted)

> **Obsolete since v9.11**: the `GraphicEQ:` command and `graphic_eq.rs` were removed,
> replaced by `pipeline/dsp/peq_hybrid.rs` (hybrid PEQ, TOML `[[effects]] type = "peq"`).
> Kept as a historical record.

**Responsibility**: graphic EQ (EqualizerAPO-aligned: log-frequency interpolation + minimum-phase FIR + direct time-domain convolution).

**Behavior highlights (v9.0 established; v9.5 block FFT; v9.6 back to direct FIR; v9.7 1024 taps + SIMD)**:
- node gains are linearly interpolated on `log(freq)`; out-of-band takes the endpoint gains;
- cepstrum-generated **1024-point minimum-phase FIR**; `initialize` builds a direct
  time-domain convolution via `ConvolutionFilter::with_ir_direct` — **no block buffering**:
  every input sample immediately produces output, so a stopped stream loses no tail
  (block FFT kept the last ≤128 samples in a block buffer that a hard engine stop discards —
  measured as a hum when switching away from M16+; v9.6 reverted to direct FIR);
- **performance (v9.7)**: direct FIR rewritten as "old→new contiguous-slice + reversed-
  coefficient dot product", AVX2+FMA 8-wide vectorized (runtime-detected, scalar mul_add
  fallback) — 1024 taps now cost less CPU than the old 512-tap scalar implementation;
  low-frequency resolution ≈46.9 Hz bin @48k below 200 Hz;
- FIR generation cached in-process by (band fingerprint, sample rate) since v9.6 — batch
  instantiation on the settings page / multi-stream no longer repeats the FFT;
- **reports no latency to the engine** (GetLatency without a child returns 0, EAPO-aligned);
- an **empty `GraphicEQ:` parameter = explicit EQ removal** (no filter, equivalent to
  passthrough) and still produces a spec fingerprint (distinct from "no command") — hot
  reload uses it to switch the old EQ chain to an empty chain (v9.5);
- the old "multi-band biquad cascade" is deprecated: Q=1.414 cascades make adjacent negative
  gains add up — measured center attenuation of −11 to −12 dB instead of the target −3 dB.

```rust
pub struct GraphicEqFilter { ... }
impl GraphicEqFilter {
    pub fn new(bands: Vec<EqBand>) -> Self;
    pub fn band_count(&self) -> usize;
}
impl Filter for GraphicEqFilter { ... }

pub struct EqBand { pub frequency: f32, pub gain_db: f32 }
pub fn parse_graphic_eq_params(spec: &str) -> Option<Vec<EqBand>>;
```

### 4.19 ~~`pipeline/dsp/convolution.rs`~~ (deleted)

> **Obsolete since v9.11**: `convolution.rs` was deleted (no IR convolution; the SIMD dot /
> block FFT moved to `fir.rs`). Kept as historical reference.

**Responsibility**: convolution (short IR direct time-domain FIR + long IR partitioned FFT, caller-injectable IR).

**v9.0 API addition**: `pub fn with_ir_direct(ir: Vec<f32>, gain_db: f32) -> Self;` — in-memory IR forced to direct time-domain FIR (GraphicEQ use; no block delay).

```rust
pub struct ConvolutionFilter { ... }
impl ConvolutionFilter {
    pub fn new(path: &str, gain_db: f32) -> Self;
    pub fn with_ir(ir: Vec<f32>, gain_db: f32) -> Self;
    pub fn with_ir_direct(ir: Vec<f32>, gain_db: f32) -> Self;
}
impl Filter for ConvolutionFilter { ... }

/// Parses Convolution: parameters (v7.11 strictness — caller decides on errors).
/// Syntax: <path> [gain dB]; the path may contain spaces/quotes.
/// - 1 token (bare path) → Ok((path, 0.0)) (gain defaults to 0)
/// - 2 tokens (path + number) → Ok((path, gain)) (the 2nd must parse as f64)
/// - ≥3 tokens / non-numeric 2nd → Err(ParseError) — extra tokens are no longer ignored
///   (ir.wav -6 abc now errors instead of ignoring abc)
pub fn parse_convolution_params(spec: &str) -> Result<(String, f32), ParseError>;
```

> **Lenient-syntax retraction (v7.11)**: v1's lenient `Convolution:` parsing (first 2 tokens,
> extras ignored) once let colon-less lines be mis-attached — the `config 6.1` single-colon
> validation (no colon → SyntaxError) sealed the entry; this function itself keeps the
> lenient "path may contain spaces" syntax (quoted paths) but counts parameters **strictly**.
> `ParseError` is wrapped by the config layer into `ConfigError::SyntaxError` (file + line)
> and written to the `log/` diagnostics.

### 4.20 ~~`pipeline/dsp/vst.rs`~~ (deleted entirely)

> **Since v9.11 `vst.rs` and the VST host were removed entirely**: `FACTORY_COUNT`,
> `index::VST_PLUGIN`, `register_builtin_filters`, `VstFactory`,
> `FilterCreateResult::NoMatch` mentioned below **no longer exist** (repo-wide `vst` search
> = 0 hits). The current handling of unknown `type` values: model validation fails → the
> whole file is rejected and the old chain is kept (see the config module spec).

> **v9.11 deprecation**: kept as historical reference. Before v9.11 (v6.5 decision + v7.11
> strictness), VST had degenerated to NoMatch mode — a `VSTPlugin:` line made `VstFactory`
> always return `FilterCreateResult::NoMatch`. After v7.11 (config 6.1 Unmatched →
> SyntaxError) the parser no longer skips silently: `VSTPlugin:` → `SyntaxError "unknown
> command 'VSTPlugin'"`, failing the whole parse (old chain kept).

**Recovery cost (no longer "add one factory")**: after TOML-ization there is no dynamic
factory registry to attach to. Restoring VST2/VST3 requires a new `EffectType` branch, a
`factory.rs::create_from_model` match branch, and the corresponding `Filter`
implementation (dynamic-library loading needs `libloading` and protocol bindings again);
the `[features] vst` flag was removed with v9.11.

## 4.21 `pipeline/dsp/loudness.rs`

**Responsibility**: ISO 226 equal-loudness curve compensation.

**References**: `crate::pipeline::dsp::filter::Filter`.

**Exports to**: only `pipeline/dsp/` internally (constructed via `factory::create_from_model`).

**Public API** (verified against code — the zh sibling's `parse_loudness_params` no longer exists; parsing/validation lives in the config layer):

```rust
pub struct LoudnessParams {
    pub phon: f32,
    pub reference_phon: f32,
}

pub struct LoudnessFilter { /* ... */ }
impl LoudnessFilter {
    pub fn new(phon: f32, reference_phon: f32) -> Self;
    pub fn set_enabled(&mut self, enabled: bool);
    pub fn enabled(&self) -> bool;
}
impl Filter for LoudnessFilter { ... }
```

## 4.22 `pipeline/dsp/` effects (peq / preamp / aural / reverb / compressor / wide / loudness)

> The current configuration model is TOML (`config/model.rs` → `ChainModel`); effects are
> constructed by `pipeline/dsp/factory.rs::create_from_model` via a static `match` on
> `EffectType`. The old EAPO-style `Key Value` text-command parsing and the `XxxFactory`
> dynamic registry are deleted; parameter ranges, key whitelists and PEQ band-count
> validation all happen in the config layer (see `driver/en/Configuration and DSP Design.md`).

`peq_hybrid.rs`: hybrid PEQ — 200 Hz crossover; bands with `Fc < 200` use cascaded IIR
biquads, bands with `Fc ≥ 200` use a sample-rate-adaptive minimum-phase FIR (1024–8192
taps; ≤2048 direct FIR / >2048 block FFT); the FIR target = total target − IIR response
(cascaded exact fitting).

**Provenance and licensing**: all effects are independent implementations (original code,
no AGPL headers) — `reverb.rs` (after Jon Dattorro's 1997 paper), `compressor.rs`
(all-channel linked RMS compressor), `wide.rs` (linear-phase FIR crossover + high-frequency
M/S decorrelation + air absorption + soft limiting, original), `aural.rs` (2nd-order
Butterworth high-pass + level follower + tanh odd soft saturation). The files sit flat in
`pipeline/dsp/` — no `fxsound/` subdirectory, no `mod.rs`.

| File | Effect | `type` |
|------|--------|--------|
| `aural.rs` | Aural Enhancer (2nd-order Butterworth HP + peak level follower + tanh soft odd harmonics + half-wave even harmonics, Wet/Dry) | `aural` |
| `reverb.rs` | Dattorro plate reverb (4-stage input allpass diffusion + dual-tank cross feedback + 14 output taps; `low_cut_hz` low-frequency transient protection) | `reverb` |
| `compressor.rs` | Compressor (all-channel two-stage detection: slow-axis loudness main detector — mean-square asymmetric τ 0.5 s up / 0.2 s down + 10 dB crest compensation; peak envelope only caps lift; fixed 3 dB soft knee; dual-pole 100/600 ms release smoothing; bounded negative-gain output; `mix` dry/wet) | `compressor` |
| `wide.rs` | Wide (linear-phase FIR crossover + high-frequency M/S decorrelation + ITD + air absorption + output soft-knee limiter) | `wide` |
| `peq_hybrid.rs` | Hybrid PEQ (IIR biquad cascade + minimum-phase FIR; band types peaking / low_shelf / high_shelf / low_pass / high_pass) | `peq` |
| `gain.rs` / `loudness.rs` | global gain / equal-loudness compensation | `preamp` / `loudness` |
| `model.rs` / `factory.rs` | `EffectType` / `EffectParams` / spec fingerprint / `create_from_model` static dispatch | — |

**References**: `crate::pipeline::dsp::filter::Filter` (every effect filter implements it).

**Exports to**: `pipeline/dsp/factory.rs::create_from_model` (static `match EffectType`); `config/` must not reference concrete implementations.

**RT constraints**:
- `initialize` precomputes coefficients and allocates state/delay lines (Reverb reserves
  "current delay + modulation headroom + maximum tap" so output taps stay valid at small
  RoomSize; modulation depth converted from the paper's EXCURSION = 16 samples @29761 Hz);
- `process` does zero allocation, no locks, no I/O, no panics; non-finite outputs are set
  to 0;
- `latency()`: implemented per filter internally, but `pipeline/process.rs` zeroes
  `latency_frames_atomic` at initialization — **latency is not reported outward** (avoids
  frame-negotiation misalignment; see the thread-safety doc);
- parameter changes go through config hot reload (the filter is rebuilt); in-stream live
  parameter writes are not supported.

**Public API** (parameter structs live in each file; the config layer deserializes and validates them):

```rust
pub struct AuralParams { pub tune_hz: f32, pub drive: f32, pub odd: f32,
    pub even: f32, pub wet: f32, pub dry: f32 }
pub struct AuralEnhancerFilter { ... }   // impl Filter

pub struct ReverbParams { pub room_size: f32, pub decay: f32, pub damping: f32,
    pub bandwidth: f32, pub density: f32, pub lat5: f32, pub lat6: f32,
    pub pre_delay_ms: f32, pub motion_rate: f32, pub motion_depth: f32,
    pub low_cut_hz: f32, pub wet: f32, pub dry: f32 }
pub struct ReverbFilter { ... }          // impl Filter

pub struct CompressorParams { pub threshold_db: f32, pub ratio: f32, pub lift: f32,
    pub attack_ms: f32, pub mix: f32 }
pub struct CompressorFilter { ... }      // impl Filter

pub struct WideParams { pub gain: f32, pub air: f32, pub side_itd: f32,
    pub crossover_hz: f32, pub low_shelf_depth_db: f32 }
pub struct WideFilter { ... }            // impl Filter

pub struct HybridPeqFilter { ... }       // impl Filter (peq_hybrid.rs)
pub struct GainFilter { ... }            // impl Filter (preamp)
pub struct LoudnessFilter { ... }        // impl Filter (loudness)
```

**Default values** (`Default` impls; Wet/Dry are overridable):
- Aural: `tune_hz 1760` / `drive 1.76993` / `odd 1` / `even 0.2` / `wet 0.5` / `dry 0.5`;
- Reverb: `room_size 1.0` / `decay 0.41` / `damping 0.408290` / `bandwidth 0.350110` /
  `density 1.0` / `lat5 0.70` / `lat6 0.50` / `pre_delay_ms 0` / `motion_rate 0.110871` /
  `motion_depth 0.63` / `low_cut_hz 100` / `wet 0.27` / `dry 0.73`;
- Compressor: `threshold_db -12` / `ratio 3` / `lift 0` / `attack_ms 10` / `mix 1.0`
  (release is fixed-automatic, no parameter);
- Wide: `gain 0.05` / `air 0.2` / `side_itd 0.2` / `crossover_hz 200` / `low_shelf_depth_db 6`.

**Key algorithms**:
- Aural: 2nd-order Butterworth high-pass (`omega = 2π·tune_hz/sr`) + peak level follower
  (instant attack / exponential release, τ≈120 ms, shared across channels);
  `s = filt/env`, `odd = env·tanh(drive·s)`, `even = env·HP20(0.5·(y+|y|))`, Wet/Dry mix.
- Reverb (Dattorro): reference sample rate 29761 Hz; input diffusion 142/107/379/277;
  in-tank 672/908 (quadrature LFO-modulated APFs, depth 16 samples @29761 Hz), 4453/4217,
  1800/2656, 3720/3163; output taps per the paper's Table 2; below the `low_cut_hz`
  crossover each channel bypasses the reverb and passes through unchanged (20 ≈ off).
- Compressor: all-channel two-stage detection — the main detector is **slow-axis loudness**
  (mean-square asymmetric: rise τ 0.5 s preserves translation, fall τ 0.2 s preserves
  recovery; + 10 dB crest compensation aligns the old peak-based scale: note-level gain is
  roughly constant, peaks and bodies shift together, crest preserved); the peak envelope
  (instant attack + depth-weighted release rate: deep 30 dB/s, shallow 180 dB/s) only caps
  `lift` (never above threshold, transients release instantly). The static curve has a fixed
  3 dB soft knee, judged on the slow-axis reading; `ratio` compresses more / `lift`
  attenuates less (Hermite C1-continuous in the knee); gain reduction is smoothed in the dB
  domain by attack/release (release fixed-automatic: dual-pole 100 ms/600 ms, the slow pole's
  weight cubes down to 0 with compression depth — deep keeps 0.3/0.7 for the tail, shallow
  rides the fast pole only); output is bounded negative gain; `mix` blends dry/wet
  proportionally.
- Wide: linear-phase FIR crossover (Kaiser window, tap count scales with sample rate /
  crossover frequency); the low-frequency reduction is applied on the **reconstructed dry
  sum** (RBJ low shelf, Q fixed 0.707, corner = crossover, depth =
  `low_shelf_depth_db` (default 6 dB) × `gain`; applying it to only the low-pass branch
  would break the two-branch symmetry and expose linear-phase ringing); Mid takes air
  absorption (high shelf + 2nd-order Bessel low-pass); Side does lateral decorrelation above
  1.5 kHz (`side_itd` = fractional delay length of the high-pass section: 0 = phase
  untouched, 1 = maximum 0.10 / 0.15 ms, Δτ = 0.05 ms scaled by sample rate). The dry path
  takes that 1.5 kHz splitter's low-pass section (same delay as the high-pass section, to
  avoid an x[n]−x[n−D2] residue); the center reference and dry signal each shift by their
  group delay D2 ⇒ the chain's total latency is D2 more than the main crossover; the side
  channel is neither boosted nor separately attenuated; the processing increment passes a
  1st-order high-pass at the crossover frequency first; the output has a soft-knee limiter
  as a backstop. With `gain`/`air`/`side_itd` all 0 it is strictly passthrough; mono is
  passthrough.

## Hard constraints

1. `pipeline/` must not depend on `install/config/object`.
2. `chain.rs` must not depend on `dsp/transition.rs`; transition state is managed by object.
3. `context.rs` must not reference `dsp/filter.rs` types.
4. RT path must be allocation-free and non-blocking.
5. `Filter`/`Chain`/`DspContext`/`RealtimeContext` must not appear in the public API surface
   (D8 pending; enforce by crate visibility, not by adding an api layer).







