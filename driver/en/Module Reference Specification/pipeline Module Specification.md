# pipeline Module Specification

**Boundary**: Does not know `install/`, `config/`, or `object/`. It only processes audio data and does not know who is calling.

**Allowed dependencies**: `sys/`, `utils/`.

**Forbidden dependencies**: `install/`, `config/`, `object/`.

## Data-flow architecture

```
Windows audio engine -> object/apo.rs::APOProcess
  -> process_audio / process_chain_interleaved
  -> pipeline/process.rs
  -> evaluate_buffer -> deinterleave -> chain.process() -> interleave
  -> apply_error_policy
```

## Module tree

```
pipeline/
├── context.rs          # PipelineContext (sample_rate, channel_names, etc.)
├── buffer.rs           # AudioBuffer / temp buffer management
├── format.rs           # format conversion and APO media-type helpers
├── chain.rs            # Chain (Vec<Box<dyn Filter>> + total_latency)
├── process.rs          # process_audio / process_chain_interleaved
├── interleave.rs       # deinterleave/interleave helpers
├── realtime.rs
├── dsp.rs
├── realtime/
│   ├── ring.rs         # lock-free ring buffer
│   └── contract.rs     # RealtimeContext marker
└── dsp/
    ├── filter.rs       # Filter trait + DspContext
    ├── factory.rs      # create_from_model static dispatch
    ├── transition.rs   # SmoothingProvider
    ├── biquad.rs       # biquad core
    ├── fir.rs          # FIR infrastructure
    ├── model.rs        # ChainModel / EffectType / EffectConfig
    ├── peq_hybrid.rs   # hybrid PEQ
    ├── gain.rs
    ├── math.rs
    ├── loudness.rs
    ├── aural.rs
    ├── compressor.rs
    ├── reverb.rs
    └── wide.rs
```

## 4.1 `pipeline/context.rs`

`PipelineContext` holds sample rate, channel names, and other non-real-time state needed during processing setup. It must not reference `dsp/filter.rs` types (`DspContext`).

## 4.2 `pipeline/buffer.rs`

Buffer types and preallocation helpers. The Chain does not own buffers; buffers are owned by `ApoObjectInner`.

## 4.3 `pipeline/format.rs`

Format extraction from `IAudioMediaType`, channel mask handling, and APO format helpers.

## 4.4 `pipeline/interleave.rs`

Deinterleave/interleave helpers used by process and transition paths.

## 4.5 `pipeline/chain.rs`

`Chain` holds `Vec<Box<dyn Filter>>` and `total_latency`. It is a pure computation container: `process()` is lock-free, allocation-free, and I/O-free. `initialize()` must be called before first `process()`.

## 4.6 `pipeline/process.rs`

Implements `process_audio` (normal path) and `process_chain_interleaved` (transition path). Handles silence detection, mono upmix, deinterleave/process/interleave, and error policy.

## 4.7 `pipeline/realtime/contract.rs`

`RealtimeContext` is a zero-sized marker used at compile time to enforce RT safety.

## 4.8 `utils/ring.rs`

Lock-free ring buffer used by telemetry and real-time logging.

## 4.9 `pipeline/dsp/filter.rs`

Defines the `Filter` trait and `DspContext` (pure data). `DspContext` is constructed by the caller (object layer) and passed to config parser.

```rust
pub trait Filter: Send + Sync + std::fmt::Debug {
    /// Process one block in deinterleaved space: `samples[channel][frame]`, in place.
    /// Must not allocate, do I/O, or panic.
    fn process(&mut self, samples: &mut [Vec<f32>], frame_count: usize);

    /// Returns the output channel names (`Some` = channel selection, `None` = unchanged).
    fn initialize(&mut self, sample_rate: u32, channel_names: &[String]) -> Option<Vec<String>>;

    fn is_channel_select(&self) -> bool { false }
    fn set_channel_indices(&mut self, _indices: &[usize]) {}
    fn fixed_channel_indices(&self) -> Option<Vec<usize>> { None }
    fn is_in_place(&self) -> bool { true }
    fn latency(&self) -> u32 { 0 }
    fn max_frame_count(&self) -> Option<usize> { None }
    fn reset(&mut self) {}
}
```

## 4.10 `pipeline/dsp/factory.rs`

Static `match` dispatch from `EffectConfig`/`EffectType` to concrete filter instances. It is the only place allowed to reference concrete filter types.

## 4.11 `pipeline/dsp/transition.rs`

`SmoothingProvider` provides raised-cosine mix factors for dual-chain transitions. It is pure math and does not manage chain state.

## 4.12-4.21 Concrete DSP modules

- `biquad.rs`: biquad filter core.
- `gain.rs`: gain filter.
- `loudness.rs`: loudness compensation.
- `fir.rs`: FIR infrastructure (SIMD dot / block FFT).
- `peq_hybrid.rs`: hybrid PEQ (IIR + FIR).
- `aural.rs`, `reverb.rs`, `compressor.rs`, `wide.rs`: effect processors.
- Historical/removed: `graphic_eq.rs`, `convolution.rs`, `vst.rs` are deprecated or removed; current effects use the TOML model.

## 4.22 Effect processors (v9.11)

The v1 TOML effect set includes `peq`, `preamp`, `aural`, `reverb`, `compressor`, `wide`,
`loudness`. Each effect has an `EffectConfig` with `spec()` for fingerprinting and
`create_from_model` for instantiation. Legacy `maximizer` / `leveler` types map to `compressor`
with their old parameter keys ignored.

Parameter ranges/defaults, the PEQ band types (`peaking` / `low_shelf` / `high_shelf` /
`low_pass` / `high_pass`, with passband gain applied for the pass filters) and the per-effect
algorithms are documented in `driver/en/Configuration and DSP Design.md` and (in Chinese,
verbatim) `driver/zh/模块引用规范/pipeline 模块规范.md` 4.22.

## Hard constraints

1. `pipeline/` must not depend on `install/config/object`.
2. `chain.rs` must not depend on `dsp/transition.rs`; transition state is managed by object.
3. `context.rs` must not reference `dsp/filter.rs` types.
4. RT path must be allocation-free and non-blocking.
5. `Filter`/`Chain`/`DspContext`/`RealtimeContext` must not appear in the public API surface
   (D8 pending; enforce by crate visibility, not by adding an api layer).
