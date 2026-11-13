# Module Reference Specification v9.18

## 1. Layer hierarchy

```
sys/          -> depends only on windows-rs; knows no business concepts
pipeline/     -> depends on sys/ + utils/; does not know install/config/object
install/      -> depends on sys/ + utils/ + object/{vx_reg_props,dll_exports}.rs; does not know pipeline/config
config/       -> depends on sys/ + utils/ + pipeline/dsp/{filter,model,factory}.rs; does not know concrete Filter implementations or install/object
object/       -> depends on all modules (glue layer, called directly by Windows)
utils/        -> depends on no other module
telemetry/    -> depends on utils/ring.rs
```

> **No api/interface layer**: the driver's external surface is the crate root `lib.rs` and the
> modules it exposes — the CLI depends on the crate as a path dependency and reuses `install/`
> functions directly, and the App goes through CLI / Tauri commands. Neither goes through an extra
> API layer. The dependency flow stays `sys <- pipeline <- config`, plus `install/`,
> `object/` (the only glue layer), `telemetry/` and `utils/`. Keeping RT types out of the public
> surface is done by **visibility tightening** (`pub(crate)`, no re-exports; see D8), not by
> adding a layer.

## 2. Complete module tree

(code-verified against `vxapo-driver/src`; the zh sibling still lists the pre-split layout —
`realtime/ring.rs` now lives in `utils/ring.rs`, `selector/select.rs` was removed, and the
slots/stale/operation groups are directories)

```
src/
├── lib.rs
├── sys.rs
├── pipeline.rs
├── install.rs
├── config.rs
├── object.rs
├── telemetry.rs
├── utils.rs
│
├── sys/
│   ├── com.rs
│   ├── registry.rs
│   ├── audio_defs.rs
│   ├── consts.rs           # CLSIDs (single source) + APOERR_* HRESULTs
│   └── com/
│       ├── prelude.rs
│       ├── apo_interfaces.rs
│       └── apo_types.rs
│
├── pipeline/
│   ├── context.rs
│   ├── buffer.rs
│   ├── format.rs
│   ├── chain.rs
│   ├── process.rs
│   ├── interleave.rs
│   ├── realtime.rs
│   ├── dsp.rs
│   ├── realtime/
│   │   └── contract.rs     # RtGuard / rt_assert_in_rt! / rt_require_non_rt! (SPSC ring lives in utils)
│   └── dsp/
│       ├── filter.rs       # Filter trait + DspContext + PassthroughFilter + ChannelScopedFilter
│       ├── factory.rs      # create_from_model static dispatch (v9.11)
│       ├── transition.rs   # SmoothingProvider + raised_cosine + mix_buffers
│       ├── biquad.rs
│       ├── fir.rs          # FIR infrastructure (SIMD dot / block FFT, v9.11)
│       ├── model.rs        # ChainModel / EffectType (v9.11)
│       ├── peq_hybrid.rs   # hybrid PEQ (v9.11)
│       ├── gain.rs
│       ├── math.rs         # shared DSP math (dB, stability, SIMD init)
│       ├── loudness.rs
│       ├── aural.rs        # effects (Aural / Reverb / Compressor / Wide / HybridPeq)
│       ├── compressor.rs
│       ├── reverb.rs
│       ├── wide.rs
│       └── specs.rs        # effect parameter spec tables (defaults/validation)
│
├── install/
│   ├── device.rs
│   ├── selector.rs
│   ├── audiodg.rs
│   │
│   ├── device/
│   │   ├── endpoint.rs
│   │   ├── format.rs
│   │   ├── identity.rs     # stable identity surviving GUID refreshes (v9.24)
│   │   ├── info.rs
│   │   ├── slots.rs
│   │   ├── slots/          # types.rs / read.rs / child.rs
│   │   ├── sysfx.rs        # CAPX MSFX template locate/takeover/restore (v9.0)
│   │   ├── stale.rs
│   │   └── stale/          # detect.rs / migrate.rs / matching.rs / acl.rs
│   │
│   └── selector/
│       ├── operation.rs
│       └── operation/      # execute.rs / helpers.rs / capx.rs
│
├── config/
│   ├── parser.rs
│   ├── model.rs            # FileModel (serde, v9.11)
│   ├── error.rs
│   └── watcher.rs
│
├── object/
│   ├── apo.rs
│   ├── dll_exports.rs
│   ├── factory.rs
│   ├── ref_count.rs
│   ├── vx_reg_props.rs
│   │
│   └── apo/
│       ├── aggregate.rs    # COM aggregation shell (NApo, non-delegating IUnknown)
│       ├── child.rs
│       ├── config.rs       # per-device config path + diag log
│       ├── init.rs         # Initialize + GetRegistrationProperties
│       ├── inner.rs        # ApoObjectInner + build_dsp_context
│       ├── lock_key.rs     # config_stamp (mtime, size) relock-reuse key
│       ├── negotiate.rs    # format negotiation
│       ├── process.rs      # APOProcess / LockForProcess / Calc frames / Reset / GetLatency
│       ├── process/tests.rs
│       ├── reload.rs       # WatcherState + start/stop_watcher + hot_reload_impl
│       ├── rtdump.rs       # RT diagnostics dump (memory buffer, flushed at Unlock)
│       ├── state.rs        # ApoState / StateCell / TransitionError / LockGuard
│       └── test_pipe.rs    # DeviceTestPipeName test notifications
│
├── telemetry/
│   ├── logger.rs
│   └── panic.rs
│
└── utils/
    ├── guid.rs
    ├── ring.rs             # SPSC lock-free ring buffer (used by telemetry/logger.rs)
    └── vx_error.rs
```

## 3. Detailed per-module specs

See the per-module files (Chinese in `driver/zh/模块引用规范/`, English in `driver/en/Module Reference Specification/`):
- `sys Module Specification.md`
- `pipeline Module Specification.md`
- `install Module Specification.md`
- `config Module Specification.md`
- `object Module Specification.md`
- `utils Module Specification.md`
- `telemetry Module Specification.md`
- `apo.rs Thread Safety Model.md`

App and CLI reference specifications are under `app/` and `cli/` respectively.

## 11. Reference dependency table

| Module | May depend on | Must not depend on |
|--------|---------------|--------------------|
| `sys/audio_defs.rs` | windows-rs | all others |
| `sys/com/prelude.rs` | windows-rs | all others |
| `sys/com/apo_interfaces.rs` | prelude, apo_types | others |
| `sys/com/apo_types.rs` | windows-rs, sys/com/prelude (HRESULT) | others |
| `sys/consts.rs` | windows-rs (GUID) | others |
| `sys/registry.rs` | windows-rs, sys/com/prelude (`guid_to_string`) | others |
| `pipeline/context.rs` | none | install/config/object |
| `pipeline/buffer.rs` | sys/com/apo_types, realtime/contract | install/config/object |
| `pipeline/format.rs` | sys/com/apo_interfaces, sys/com/apo_types, sys/com/prelude, sys/audio_defs, utils/vx_error | install/config/object |
| `pipeline/interleave.rs` | none | install/config/object |
| `pipeline/chain.rs` | dsp/filter, utils | install/config/object, dsp/transition |
| `pipeline/process.rs` | context, chain, buffer, interleave, dsp/filter, dsp/transition, realtime/contract (RealtimeContext), sys/com/apo_types, utils | install/config/object |
| `pipeline/realtime/contract.rs` | core | others |
| `utils/ring.rs` | core | others |
| `pipeline/dsp/filter.rs` | utils | config/install/object |
| `pipeline/dsp/factory.rs` | dsp/filter, dsp/model (EffectConfig/EffectType), dsp/* (static match instantiation, v9.11), utils | config/install/object |
| `pipeline/dsp/transition.rs` | none | config/install/object |
| `pipeline/dsp/*.rs` (concrete filters) | dsp/filter, dsp/model (v9.11), dsp/biquad (loudness internal), dsp/math, dsp/fir, utils | **config/ (config must not reference them directly)** |
| `install/device/endpoint.rs` | sys/registry, sys/com/prelude (`guid_to_string`), utils/guid, utils/error | pipeline/config |
| `install/device/format.rs` | sys/registry, utils/error, sys/audio_defs (`default_channel_mask` only) | config |
| `install/device/slots.rs` | sys/registry, utils/guid, sys/com/prelude (`guid_to_string`) | pipeline/config |
| `install/device/identity.rs` | sys/registry, utils | pipeline/config |
| `install/device/info.rs` | device/endpoint, device/format, device/slots, sys/registry, object/vx_reg_props, utils/error | pipeline/config |
| `install/device/sysfx.rs` | device/slots, object/vx_reg_props, sys/registry, sys/com/prelude, utils/error | pipeline/config |
| `install/device/stale.rs` | device/endpoint, device/info, device/slots, device/sysfx, device/identity, selector/operation, sys/registry, sys/com/prelude, utils/guid, utils/error | pipeline/config |
| `install/selector.rs` | selector/operation (entry aggregation) | pipeline/config |
| `install/selector/operation.rs` | install/audiodg, device/slots, device/sysfx, device/format, device/identity, sys/registry, object/vx_reg_props, object/dll_exports (`register_apo_with_path`), sys/com/prelude (`guid_to_string`), utils/error | pipeline/config |
| `install/audiodg.rs` | sys/registry, utils/error | pipeline/config/object |
| `config/parser.rs` | config/error, config/model, pipeline/dsp/model, pipeline/dsp/filter, pipeline/dsp/factory | **concrete dsp types (peq_hybrid etc.)**, install/object, pipeline process/chain/context, sys/audio_defs |
| `config/model.rs` | config/error, pipeline/dsp/model, pipeline/dsp/{aural,compressor,reverb,wide} (param structs) | install/object, pipeline process/chain |
| `config/watcher.rs` | config/error, utils | install/object |
| `config/commands.rs` | — (removed in v9.11) | — |
| `config/commands/*.rs` | — (removed in v9.11) | — |
| `object/apo.rs` | **all modules** (incl. transition, telemetry/logger, sys/audio_defs, pipeline/dsp/filter [DspContext]); module entry aggregating `object/apo/*` | none |
| `object/apo/aggregate.rs` | object/apo (entry ApoObject), sys/com/prelude, sys/com/apo_interfaces (IIDs) | object/apo (no cycles; entry/shell exception only) |
| `object/apo/child.rs` | sys/com/prelude, sys/com/apo_interfaces, sys/com/apo_types | object/apo (no cycles) |
| `object/apo/config.rs` | config/parser, config/watcher, pipeline/chain, pipeline/dsp/transition, sys/com/apo_types, sys/com/prelude, super (ApoObject_Impl), super/inner | object/apo (no cycles) |
| `object/apo/init.rs` | install/device/slots, install/device/sysfx, install/selector/operation (`find_endpoint_path`), object/vx_reg_props, object/apo/aggregate, object/apo/config, object/apo/child, object/apo/state (ApoState), super (ApoObject_Impl), sys/com/prelude, sys/com/apo_types | object/apo (no cycles) |
| `object/apo/inner.rs` | pipeline/chain, pipeline/context, pipeline/dsp/filter, pipeline/dsp/transition, sys/audio_defs | object/apo (no cycles) |
| `object/apo/negotiate.rs` | sys/com/apo_interfaces, sys/com/apo_types, pipeline/format, utils | object/apo (no cycles) |
| `object/apo/process.rs` | config/parser, install/audiodg, object/vx_reg_props, object/apo (entry), object/apo/config, object/apo/aggregate, object/apo/inner, pipeline/*, sys/audio_defs, sys/com/apo_interfaces, sys/com/apo_types, sys/com/prelude | object/apo (no cycles) |
| `object/apo/state.rs` | core, sys/com/apo_types (APOERR_*), sys/com/prelude (HRESULT) | object/apo (no cycles) |
| `object/factory.rs` | sys/com/prelude, object/apo/aggregate, vx_reg_props, ref_count | none |
| `object/ref_count.rs` | core | none |
| `object/vx_reg_props.rs` | sys/com/prelude, sys/com/apo_types, sys/com/apo_interfaces (IID_IAPO), sys/consts (CLSIDs) | none |
| `object/dll_exports.rs` | object/*, sys/com/prelude, sys/registry (CLSID key writes), utils, telemetry | none |
| `utils/guid.rs` | core | all others |
| `utils/vx_error.rs` | windows-core | all others |
| `telemetry/logger.rs` | utils/ring | install/config/object |
| `telemetry/panic.rs` | telemetry/logger | install/config/object |

> General note (v9.17): `pipeline/dsp/*.rs` concrete filters may depend on sibling modules
> like `dsp/math` and `dsp/fir` (the older "allowed list" did not cover such internal
> collaboration); the row-level authorization for `object/apo/*` sharing `inner`/`aggregate`
> state is governed by this table — unregistered callbacks are forbidden.

### 11.1 Third-party dependencies

| Dependency | Used in | Purpose |
|------------|---------|---------|
| `rustfft` | `pipeline/dsp/{fir,peq_hybrid}` | FFT/convolution |
| `thiserror` | `utils/vx_error` | error enum derive |
| `once_cell` | `install/device/sysfx` | lazy init |
| `serde` / `toml` | `config/{model,parser}` | TOML deserialization |

> windows / windows-core / windows-implement / windows-interface follow the existing
> per-row registrations for `sys/*` and `object/*` and are not repeated line by line.

### 11.2 Dependency rules D1-D8

| # | Rule | Status | Verified by |
|---|------|--------|-------------|
| D1 | `sys/` has zero dependencies besides windows-rs/core | ✅ | script |
| D2 | `utils/` has zero internal dependencies | ✅ | script |
| D3 | `pipeline/` must not depend on `config/install/object` | ✅ | script |
| D4 | `config/` must not depend on `install/object/pipeline/{process,chain,context}` or concrete Filters | ✅ | script |
| D5 | `install/` must not depend on `pipeline/config` (except `object/vx_reg_props`) | ✅ | script |
| D6 | `object/` is the only glue layer allowed to depend on all modules; `object/apo/*` only uses the sibling references authorized row by row in this table | ⚠️ per table | script |
| D7 | `telemetry/` depends only on `utils/ring` | ✅ | script |
| D8 | RT path types (`Filter/Chain/DspContext/RealtimeContext`) must not appear in the public API surface | pending (enforce via `lib.rs` visibility: `pub(crate)` RT types, no re-exports; **no api layer is introduced**) | compile + review |

### 11.3 Automatic validation

`vxapo-driver/scripts/check_deps.ps1` runs before `cargo test`: it parses each src file's
`use crate::` / `crate::` / `super::` references (ignoring `cfg(test)` and `mod tests`) and
asserts this table's per-file whitelists (`/*` acts as a prefix wildcard). The script's
embedded whitelist and this section are maintained **in lockstep across the three files** —
whenever the table changes, the script must be updated too.

## 12. Forbidden dependency-chain check

| Source module | Forbidden dependency | Reason |
|---------------|----------------------|--------|
| `sys/` | any other module | the FFI layer knows no business |
| `pipeline/` | `install/`, `config/`, `object/` | the runtime knows nothing of installation and configuration |
| `pipeline/chain.rs` | `pipeline/dsp/transition.rs` | the transition state machine is managed by the object layer |
| `pipeline/context.rs` | `pipeline/dsp/filter.rs` (DspContext) | context.rs must not reference dsp/ types |
| `pipeline/dsp/filter.rs` | `pipeline/context.rs` (PipelineContext) | filter.rs must not reference context.rs types; the caller constructs DspContext directly |
| `install/` | `pipeline/`, `config/` | installation knows nothing of the runtime and configuration |
| `config/` | `install/`, `object/`, `pipeline/process.rs`, `pipeline/chain.rs`, `pipeline/context.rs` | config parsing knows nothing of installation and runtime dispatch |
| `config/parser.rs` (v9.11) | `pipeline/dsp/factory.rs` | dispatch goes through `create_from_model`; concrete Filters are never touched directly |
| `telemetry/logger.rs` | `alloc::string::String` | the real-time path forbids heap allocation (fixed-size arrays) |
| `utils/` | any other module | the utility layer must not depend back |

## 13. Data flow

### Install flow

```
CLI/App (user choice)
  → device/info.rs::enumerate_devices (query DeviceInfo; the single device-enumeration entry)
  → selector/operation.rs::install_endpoint (7-step install incl. Transaction rollback)
      → audiodg.rs (check DisableProtectedAudioDG)
      → sys/registry.rs (registry writes + save_to_file backup)
      → object/vx_reg_props.rs (read CLSIDs)
```

### Config load (LockForProcess)

```
object/apo.rs::LockForProcess
  → state.transition(Initialized → Locked)   (+ RAII guard; auto-rollback on failure)
  → pipeline/format.rs (extract format from IAudioMediaType)
  → build PipelineContext (static format info) and DspContext (pure data, read-only)
  → config/parser.rs::ConfigParser.parse_file_with_spec(path, &dsp_ctx)   (v9.11 TOML)
      → toml::from_str::<FileModel> → to_chain_model (validation)
          → factory::create_from_model (static match, build the Filter chain)
      → returns (filters, spec fingerprint)
  → relock short-circuit: same (config_path, mtime, size, sample_rate, channel_names) key
    and a non-empty current_chain → reuse the existing chain (keep filter state);
    PostMix instances always take the empty-chain passthrough path (no config parse)
  → pipeline/chain.rs::Chain::new() + add_filter × N + initialize(sample_rate, channel_names)
  → pipeline/context.rs (store PipelineContext)
  → preallocate temp_buffers / temp_buffer_old / temp_buffer_new
  → latency_samples / latency_frames_atomic = chain.total_latency() (not reported outward;
    GetLatency delegates to the child or returns 0)
  → install/audiodg.rs::ensure_can_load()
  → guard.disarm(); start the watcher (except PostMix)
```

### Config hot reload (watcher-triggered, v6.9 R2 blocking)

```
config/watcher.rs (detects config.toml changes; directory watch, 10 ms dedup)
  → object/apo.rs::hot_reload
      → short-lock check transition.is_some() || reloading
          ├─ yes → return (blocking: no chain build during a transition)
          └─ no → parse OUTSIDE the lock: ConfigParser.parse_file_with_spec
                   (TOML → FileModel → ChainModel → create_from_model,
                    spec-fingerprint short-circuit, v9.11)
      → mutex.lock()
      → race fallback: if a new transition started during parsing → pending_reload = true, return
      → new Chain (after initialize) into current_chain, old Chain into outgoing_chain
      → transition = Some(SmoothingProvider::new(10 ms))
      → reloading = false
      → mutex.unlock()

APOProcess transition-completion frame (RT):
  → retired_chain = outgoing_chain.take()   // R1: old chain into the retired slot, zero destruction
  → if pending_reload && !reloading → drop the lock → hot_reload()
```

### Runtime processing (incl. dual-chain transition)

```
Windows audio engine
  → object/apo.rs::APOProcess
      → state.current() check == Locked
      → child-APO pre-delegation (childRT->APOProcess on the input buffer; its output is
        the VxAPO chain input)
      → mutex.lock()
      → check whether outgoing_chain exists

      ├─ [normal mode]
      │   → process_audio(props, params, current_chain, stats, temp_buffers)
      │       → evaluate_buffer → deinterleave → silence detection → mono upmix
      │       → chain.process() (planar space)
      │       → interleave → apply_error_policy (policy-based recovery on failure, counter++)
      │
      └─ [transition mode]
          → extract the input slice from input_props[0]
          → process_chain_interleaved(outgoing) → temp_buffer_old
          → apply_error_policy (policy-based recovery on failure)
          → process_chain_interleaved(current_chain) → temp_buffer_new
          → apply_error_policy (policy-based recovery on failure)
          → transition.advance() → factor
          → mix_buffers(old, new, output, factor) (interleaved-space mix)
          → factor >= 1.0 → retired_chain = outgoing_chain.take(), transition = None
                → if pending_reload → drop the lock → hot_reload()
      → mutex.unlock()
```

## 14. Key design decisions

### Chain does not own buffers

`Chain` only holds `Vec<Box<dyn Filter>>` and `total_latency`. Buffers are preallocated and
held by `object/apo.rs`'s `ApoObjectInner` at `LockForProcess`. `Chain::process()` is pure
computation (Filter-by-Filter calls) — no locks, no allocation, no I/O. `initialize()` must
be called before the first `process()`, otherwise filters that rely on precomputed state
(GraphicEQ/PEQ/IIR/Delay/Convolution) silently idle.

### Dual-chain transition is managed in the object layer

`SmoothingProvider` only provides the raised-cosine mix factor (pure algorithm). Chain state
(`current_chain` / `outgoing_chain` / `retired_chain` / `transition` / `pending_reload`) is
managed by `ApoObjectInner`. The `process_chain_interleaved` bridge encapsulates the full
interleave → deinterleave → process → interleave flow, called once per old/new chain in
transition mode and then mixed.

### DspContext is constructed by the caller

`pipeline/dsp/filter.rs` defines the `DspContext` struct (pure data) without depending on
`pipeline/context.rs`'s `PipelineContext`. `object/apo.rs` builds `DspContext` from
`PipelineContext` + extra parameters at `LockForProcess` and passes it to `ConfigParser`.

### config/ does not operate Chain directly

`config/parser.rs` returns `Vec<Box<dyn Filter>>`; `object/apo.rs` adds them to the Chain.
Channel selection is requested through `Filter::initialize()` returning
`Some(channel_names)`; config never touches Chain's channel-management interface directly.

### Thread-safety model

- `self.mutex` (`Arc<Mutex<ApoObjectInner>>`, `object/apo/inner.rs`): protects Chain, transition
  state, `PipelineContext`, temp buffers, and the format/channel/lock bookkeeping
  (`pending_reload` / `active_spec`)
- `config_path` / `watcher_state` / `child_apo` are separate `Arc<Mutex<…>>` fields
  (`object/apo.rs`), never held crossing the main lock
- **removed**: `self.ap_state` (`Mutex<ApoObjectState>`) — the field and the
  `ApoObjectState` type have 0 hits repo-wide; format/channel state now lives in
  `ApoObjectInner`, and the state machine is `StateCell` (`object/apo/state.rs`)
- `self.state_cell` (`AtomicU8` + CAS): lock-free state machine (Created → Initialized → Locked)
- `self.latency_samples` / `self.latency_frames_atomic` (`AtomicU32`): lock-free latency values
- the two locks are never held simultaneously; `CalcInputFrames` / `CalcOutputFrames` read
  the atomics lock-free

## 15. Production build constraints (v6.6, O3)

Hard requirements for the Windows-APO production **release** build (tympan-apo-inspired baseline):

```toml
[profile.release]
panic = "abort"      # required: an RT-path panic must not unwind across FFI (UB risk)
codegen-units = 1    # required: predictable inlining/layout, minimal RT jitter
lto = "thin"         # recommended: smaller + cross-crate inlining
strip = "debuginfo"  # recommended: production DLL size
```

- **RT paths must never unwind**: all real-time callbacks (`APOProcess` / `Filter::process`)
  must not unwind the stack. The RT entries are `object/apo/process.rs::apo_process` and
  `pipeline/dsp/filter.rs::Filter::process` (the original row's `accept_aux_input` does not
  exist in this project — `Filter` has no such method — and was removed).
  `panic = "abort"` makes this a hard boundary: a panic terminates the process, it never
  unwinds across an `extern "system"` FFI boundary.
- Development/test profiles may use `panic = "unwind"` (to exercise catch_unwind defense
  paths), but **production release must abort**.

## Key design decisions, continued (v6.6, tympan-apo-inspired)

| # | Decision | Lands in |
|---|----------|----------|
| O1 | **RT compile-time witness**: `RealtimeContext` zero-sized marker + `DspContext::rt_marker` (`PhantomData<RealtimeContext>`) — problems solvable at compile time are never deferred to runtime | pipeline 4.7 / 4.9 |
| O2 | **StateCell completion**: `TransitionError{expected, attempted, actual}` + semantic transitions. (Historical note: the `release()` / `initialize()` / `lock()` / `unlock()` convenience methods no longer exist — the live API is `transition` / `current` plus the `LockGuard` RAII.) | object 7.1.4 |
| O3 | **Production build constraints**: release must use `panic="abort"` + `codegen-units=1` (RT unwind across FFI = UB) | section 15 |
| O4 | **AEC interface reservation**: `feature = "aec"`-gated 3 AEC interfaces + IID constants (declared, not implemented) | sys 3.2 |
| E1 | **In-place declaration** (EqualizerAPO `IFilter::getInPlace`-inspired): `Filter::is_in_place()` (default true). (Historical note: `Chain::is_fully_in_place()` was removed; all built-in filters are in-place, so the planar work buffer is always the final output — zero-copy fast path.) | pipeline 4.9 / 4.5 / 4.6 |
| E3.1 | **Default-device decision boundary**: the driver layer (`enumerate_devices`) only returns legal install containers; default/valid devices are resolved by the user layer through COM `GetDefaultAudioEndpoint` and matched against the driver results | install 5.4 |
| E3.2 | **Device physical-state predicates** (EAPO `isDisabled/isUnplugged`-inspired): `DeviceInfo::is_disabled()/is_unplugged()` derived from the existing `EndpointState`, zero extra I/O | install 5.4 |
| E3.3 | **autoAdjust as an independent field** (EAPO `InstallState::autoAdjust`-inspired): `InstallConfig::auto_adjust` (default false); install Step 4 reads the field instead of hard-coding | install 5.5.2 |
| E3.4 | **Install self-check** (EAPO `testAPOInstallation`-inspired): `install_endpoint(..., verify)` — after commit, `CoCreateInstance` verifies the DLL instantiates; failure does not auto-rollback, it reports and lets the caller decide | install 5.5.2 |
| R1 | **Retired-chain deferred destruction** (EAPO `previousConfig`-inspired): at the transition-completion frame the RT thread only does `retired_chain = outgoing_chain.take()` (zero destruction); the control thread drops it inside the lock — heavy filter destructors never run on the RT thread | object 7.1.3/7.1.9/7.1.10/7.1.11/7.1.13 |
| R2 | **Blocking reload** (EAPO `loadSemaphore`-aligned): hot_reload returns without building while a transition is active/loading; APOProcess triggers the reload after the transition. The `reloading` flag prevents overwrite — eliminating queued duplicate parses and the overwrite hole | object 7.1.3/7.1.11/7.1.18 |
| R3 | **Empty-chain fast path** (EAPO `isEmpty`-inspired): when `Chain::is_empty()`, `process_audio` copies the deinterleaved result directly (near-memcpy), skipping chain traversal; stacks with the E1 in-place fast path | pipeline 4.5/4.6 |
| R4 | **10 ms transition period** (EAPO-aligned): `default_smoothing_length` went from `sample_rate/20` (50 ms) to `sample_rate/100` (10 ms) — the dual-processing window shrinks 5×, "chopping at will is fine" | pipeline 4.11 |

## 16. Change and feedback governance

- New features or spec changes are recorded first in the current-status section of
  the project overview (`overview/zh/项目概览.md` / `overview/en/Project Overview.md`).
- Spec issues found during implementation are recorded in the code-review summary or
  the project overview (`driver/zh/代码审查摘要.md` / `driver/en/Code Review Summary.md`; `overview/zh/项目概览.md` / `overview/en/Project Overview.md`), and this document is revised in sync.
- The state machine stays: `Backlog -> Spec-Drafting -> Spec-Finalized -> Implementing -> Done`.
- Hard gate: only `Spec-Finalized` may enter `Implementing`.

## 18. EAPO alignment and differentiation (v8.1, P0-6 child-APO alignment)

> **Purpose**: make explicit how VxAPO aligns with and differs from EqualizerAPO on the two
> mechanisms "child APO" and "transition", and the design impact of the differences.
> Basis: direct reading of EAPO `EqualizerAPO.cpp` / `FilterEngine.h/.cpp` /
> `FilterConfiguration.cpp` (2026-08-03).

### 18.1 Alignment (EAPO factual behavior == VxAPO decision)

| # | Mechanism | EAPO fact (source) | VxAPO alignment decision | Lands in |
|---|-----------|--------------------|--------------------------|----------|
| A1 | Child-APO creation flow | CoCreateInstance(IAPO) → QI RT → QI Cfg → Initialize with the parent's APOInit data (EqualizerAPO.cpp 186-215) | **fully aligned**: object/child.rs create three steps + Initialize pass-through | object 7.1.8 / 7.2 |
| A2 | Creation-failure degradation | any step fails → resetChild (Release the three interfaces) → **return S_OK, never block the parent** (191-209) | **fully aligned**: child_apo = None degraded mode | object 7.1.8 Note 57 |
| A3 | APOProcess delegation order | **childRT->APOProcess runs first** (on the input buffer) → engine.process processes its output (472-478) | **fully aligned**: child runs first, once per frame; its output is the VxAPO chain input | object 7.2 / 7.1.11 |
| A4 | GetLatency delegation | `*pTime=0` → delegate to childAPO->GetLatency **only when a child exists** (91-94); **no child → return 0** | **aligned (v8.3 correction)**: delegate to the child; no child → 0 (object 7.2's "no child → own value" corrected to "return 0") | object 7.2 |
| A5 | Child output buffer | childRT->APOProcess runs first on the input/output buffers (472-477); **whether the child is in-place is the child's own is_in_place decision** — EAPO does not force "no separate buffer" | **aligned (v8.3 correction)**: child follows is_in_place semantics; non-in-place needs `temp_buffer_child` (preallocated at Lock, zero RT allocation) | object 7.1.9 |

### 18.2 Differentiation (VxAPO-specific, absent in EAPO)

> **Preamble — APO-chain format-grant semantics (v8.1 correction; the root cause of 18.2's
> mechanics)**: inFormat/outFormat are the parent APO's **two independent connection
> descriptors** (granted separately by the engine):
> - **no child**: the parent processes in→out itself; the parent **rejects downmix**
>   (IsInputFormatSupported returns S_FALSE + outFormat as the supported format for
>   in>out, 277-282) + **only mono→stereo upmix is compensated** (FilterConfiguration.cpp
>   126-128, restoring the system default behavior) — except that special case, in==out
>   after negotiation.
> - **with child**: the parent passes the in/out descriptors **verbatim** to the child at
>   LockForProcess (`childCfg->LockForProcess(ppInputConnections, ppOutputConnections)`);
>   the child's **input = inFormat** (upstream), **output = outFormat** (downstream / what
>   the parent wants); the child is responsible for the in-place in→out conversion; what
>   the parent really processes is **the buffer after the child's output = the out layout**,
>   hence `realChannelCount = outFormat` (processing dimension = output).
> - **negotiation delegation**: with a child, the parent first asks
>   `childAPO->IsInputFormatSupported` (258-261); if the child succeeds, its verdict is
>   used; only on child failure/absence does the parent fall back to its own check
>   (272-283) — **the child's format interfacing is the child's own responsibility**; the
>   parent does not pre-reject on the child's behalf.
> - **the only upmix special case = mono→stereo**: FilterConfiguration.cpp 126-128 comments
>   "emulate the Windows audio system's automatic upmix when no APO is present" — after
>   taking over the whole APO instance, EAPO must restore that system default; all other
>   up/down mixes are not performed by the parent (except explicit `Copy:` mapping).

| # | Mechanism | EAPO | VxAPO difference | Design impact |
|---|----------|------|------------------|---------------|
| D1 | **Transition mechanism** | single chain + config-level transition: FilterConfiguration mixes old/new allSamples planes via `doTransition` (157-176); child unchanged | **in-parent dual-chain transition**: outgoing/current chains mixed in interleaved space via mix_buffers | the child runs first, once per frame; both chains share the same child output as input — the child is in neither chain (held separately); the transition only switches the parent's internal chains; no "transition-period child double instance" needed |
| D2 | **Channel semantics** | `realChannelCount` takes in/out depending on child presence; **rejects downmix** (in>out and >2ch → S_FALSE + outFormat, 277-282) + **only mono→stereo upmix compensated** (126-128; all other up/down mixes are not performed by the parent) | **equivalent stance (v8.3 corrected wording)**: "received input == output channel count" is equivalent to "**reject downmix + only mono→stereo upmix**" (the parent does not convert); **no realChannelCount dimension-switching mechanism is introduced** (EAPO's branch is a redundant path after negotiation) — **implementation simplification, not extra rigor** | (1) dual-chain temp_buffers/deinterleave dimensions stay stable (input==output except mono→stereo → both chains' input dimensions identical); (2) no downmix/upmix support (except mono→stereo) — equivalent to EAPO; (3) if channel conversion is ever needed, see EAPO plan B (dimension switch + mono upmix special case + explicit `Copy:` mapping), deferred as P1 |
| D3 | **Child output channel count is unknowable** | the child has no report-channel-count interface; the parent assumes the child already produces the outFormat layout (trusting the **APO-chain format-passing convention**) | **equivalent**: the child output is read at the outFormat dimensions; the child's channel layout is a contract declared by the configuration side (guaranteed by whoever installs/configures the child); **neither side does runtime layout detection** | the child itself cannot be "constrained" — layout anomalies are the configurator's contractual responsibility (same as EAPO); VxAPO only backstops with the "received input == output channel count" check (the parent does not convert) |
| D4 | **Unlock failure semantics** | childCfg->UnlockForProcess FAILED → **return hr** (parent unlock fails, 393-401) | **VxAPO fault tolerance**: a child unlock failure does not block the parent's unlock (object 7.1.10 semantics) | intentional difference: UnlockForProcess has no retry semantics; the child may be partially unlocked already; the parent continues its own flow + logs (same degraded stance as Note 57) |
| D5 | **IsInputFormatSupported delegation failure** | child delegation failure → resetChild + fall back to the parent's own check (258-283) | **aligned + narrowed wording**: a successful child delegation uses the child's verdict; failure falls back to the parent's own check (VxAPO's three-stage whitelist already gives precise command/parameter errors) | no extra difference |

> **v8.1 correction note**: the v8.1 draft's "explicit constraints beat EAPO's implicit trust"
> wording is **retracted** — after analysis, EAPO achieves complete channel semantics via
> **downmix rejection** (S_FALSE for in>out) + **mono→stereo system-default compensation** +
> **negotiation delegation to the child**; VxAPO is **equivalent**, the only difference being
> that it does not introduce the realChannelCount mechanism (whose in/out branches are
> redundant/dead paths after negotiation) — an **implementation simplification**.
> **v8.3 correction note (line-by-line source check)**: (1) "negotiation locks in==out" was
> inaccurate — EAPO actually does **downmix rejection + mono→stereo upmix only** (not a full
> lock); (2) GetLatency with no child returns 0 (not "falls back to its own value"); (3) a
> failing child is resetChild-destroyed (not just fallen back). See
> `driver/en/Configuration and DSP Design.md` §5 (EqualizerAPO behavior reference).

### 18.3 Impact of the differentiation (D1-D3 design consequences)

1. **D1 → RT cost stacking**: child pre-run ×1 + parent dual chains ×2 = 3 DSP executions per
   frame (within the 10 ms transition window); R4 (10 ms, chop at will) still holds with a
   child present, but the in-window CPU peak is 1.5× the no-child case — **accepted**
   (the control thread switches chains; the RT thread only computes).
2. **D2 → dimension stability**: the child does not change channels ⇒ both chains' input
   slices have identical dimensions and temp_buffers / temp_buffer_old / temp_buffer_new share
   the same child output as input, with no reshuffle logic — **this is the precondition for
   VxAPO's dual chains to stack with a child**; relaxing it in the future (plan B) would
   require the dual chains to support two channel dimensions (significantly more complex).
3. **D3 → check up front (equivalent stance)**: the LockForProcess "received input == output
   channel count" check (the parent's no-conversion stance, zero RT cost on the control
   thread); the child layout contract is the configurator's responsibility (same as EAPO).
   **VxAPO detects no more child-layout problems than EAPO, and never silently misaligns —
   equivalent.**

### 18.4 Reference relationships

- This section's P0-6 (roadmap): child-APO delegation implementation — the convergence basis
  for open decisions (1)(2)(3) is exactly this chapter (P0-6 Spec-Drafting finalization only
  needs to confirm against it; the design is not reopened).
- Pairs with `object 7.1.3` (child_apo field), `object 7.1.8` (Initialize creation +
  degradation), `object 7.1.11` (APOProcess dual chains + child pre-run), `object 7.2`
  (child.rs delegation methods).
- EAPO source reference columns: `EqualizerAPO.cpp` (180-215/258-283/341-369/391-424/455-495),
  `FilterEngine.h` (129-135), `FilterConfiguration.cpp` (81-215).


