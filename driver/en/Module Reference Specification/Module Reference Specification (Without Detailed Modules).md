# Module Reference Specification v9.18 (English)

## 1. Layer hierarchy

```
sys/          -> depends only on windows-rs; knows no business concepts
pipeline/     -> depends on sys/ + utils/; does not know install/config/object
install/      -> depends on sys/ + utils/ + object/vx_reg_props.rs; does not know pipeline/config
config/       -> depends on sys/ + utils/ + pipeline/dsp/filter.rs + pipeline/dsp/model.rs + pipeline/dsp/factory.rs; does not know concrete Filter implementations or install/object
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
│   │   ├── ring.rs
│   │   └── contract.rs
│   └── dsp/
│       ├── filter.rs
│       ├── factory.rs
│       ├── transition.rs
│       ├── biquad.rs
│       ├── fir.rs
│       ├── model.rs
│       ├── peq_hybrid.rs
│       ├── gain.rs
│       ├── math.rs
│       ├── loudness.rs
│       ├── aural.rs
│       ├── compressor.rs
│       ├── reverb.rs
│       └── wide.rs
│
├── install/
│   ├── device.rs
│   ├── selector.rs
│   ├── audiodg.rs
│   ├── device/
│   │   ├── endpoint.rs
│   │   ├── format.rs
│   │   ├── slots.rs
│   │   ├── sysfx.rs
│   │   ├── stale.rs
│   │   └── info.rs
│   └── selector/
│       └── operation.rs
│
├── config/
│   ├── parser.rs
│   ├── model.rs
│   ├── error.rs
│   └── watcher.rs
│
├── object/
│   ├── apo.rs
│   ├── dll_exports.rs
│   ├── factory.rs
│   ├── ref_count.rs
│   ├── vx_reg_props.rs
│   └── apo/
│       ├── aggregate.rs
│       ├── child.rs
│       ├── config.rs
│       ├── init.rs
│       ├── inner.rs
│       ├── negotiate.rs
│       ├── process.rs
│       └── state.rs
│
├── telemetry/
│   ├── logger.rs
│   └── panic.rs
│
└── utils/
    ├── align.rs
    ├── guid.rs
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

## 4. Reference dependency table (summary)

| Module | May depend on | Must not depend on |
|--------|---------------|--------------------|
| `sys/audio_defs.rs` | windows-rs | all others |
| `sys/com/prelude.rs` | windows-rs | all others |
| `sys/com/apo_interfaces.rs` | prelude, apo_types | others |
| `sys/com/apo_types.rs` | windows-rs, prelude | others |
| `sys/registry.rs` | windows-rs, prelude (`guid_to_string`) | others |
| `pipeline/context.rs` | none | install/config/object |
| `pipeline/buffer.rs` | sys/com/apo_types | install/config/object |
| `pipeline/format.rs` | sys/com interfaces/types/prelude, sys/audio_defs, utils/vx_error | install/config/object |
| `pipeline/interleave.rs` | none | install/config/object |
| `pipeline/chain.rs` | dsp/filter, utils | install/config/object, dsp/transition |
| `pipeline/process.rs` | context, chain, buffer, interleave, dsp/filter, dsp/transition, realtime/contract, sys/com/apo_types, utils | install/config/object |
| `pipeline/realtime/contract.rs` | core | others |
| `utils/ring.rs` | core | others |
| `pipeline/dsp/filter.rs` | utils | config/install/object |
| `pipeline/dsp/factory.rs` | dsp/filter, dsp/model, dsp/* (static match), utils | config/install/object |
| `pipeline/dsp/transition.rs` | none | config/install/object |
| `pipeline/dsp/*.rs` (concrete filters) | dsp/filter, dsp/model, dsp/biquad, utils | config (direct reference forbidden) |
| `install/device/endpoint.rs` | sys/registry, prelude, utils/guid, utils/error | pipeline/config |
| `install/device/format.rs` | sys/registry, utils/error, sys/audio_defs | config |
| `install/device/slots.rs` | sys/registry, utils/guid, prelude | pipeline/config |
| `install/device/info.rs` | device/endpoint, device/format, device/slots, sys/registry, object/vx_reg_props, utils/error | pipeline/config |
| `install/device/sysfx.rs` | device/slots, object/vx_reg_props, sys/registry, prelude, utils/error | pipeline/config |
| `install/device/stale.rs` | device/endpoint, device/info, device/slots, device/sysfx, selector/operation, sys/registry, prelude, utils/guid, utils/error | pipeline/config |
| `install/selector.rs` | selector/operation (entry aggregation) | pipeline/config |
| `install/selector/operation.rs` | install/audiodg, device/slots, device/sysfx, device/format, sys/registry, object/vx_reg_props, object/dll_exports, prelude, utils/error | pipeline/config |
| `install/audiodg.rs` | sys/registry, utils/error | pipeline/config/object |
| `config/parser.rs` | config/error, config/model, pipeline/dsp/model, pipeline/dsp/filter, pipeline/dsp/factory | concrete dsp types, install/object, pipeline process/chain/context, sys/audio_defs |
| `config/model.rs` | config/error, pipeline/dsp/model, pipeline/dsp/{aural,compressor,reverb,wide} | install/object, pipeline process/chain |
| `config/watcher.rs` | config/error, utils | install/object |
| `object/apo.rs` | all modules (glue) | none |
| `object/apo/*.rs` | as authorized by the detailed table | object/apo circular dependencies |
| `object/factory.rs` | prelude, object/apo/aggregate, vx_reg_props, ref_count | none |
| `object/ref_count.rs` | core | none |
| `object/vx_reg_props.rs` | prelude, apo_types, apo_interfaces | none |
| `object/dll_exports.rs` | object/*, prelude, sys/registry, utils, telemetry | none |
| `utils/align.rs` | none | all others |
| `utils/guid.rs` | core | all others |
| `utils/vx_error.rs` | windows-core | all others |
| `telemetry/logger.rs` | pipeline/realtime/ring | install/config/object |
| `telemetry/panic.rs` | telemetry/logger | install/config/object |

### 4.1 Third-party dependencies

| Dependency | Used in | Purpose |
|------------|---------|---------|
| `rustfft` | `pipeline/dsp/{fir,peq_hybrid}` | FFT/convolution |
| `thiserror` | `utils/vx_error` | error enum derive |
| `once_cell` | `install/device/sysfx` | lazy init |
| `serde` / `toml` | `config/{model,parser}` | TOML deserialization |

### 4.2 Dependency rules D1-D8

| # | Rule | Status |
|---|------|--------|
| D1 | `sys/` depends only on windows-rs/core | ✅ |
| D2 | `utils/` has zero internal dependencies | ✅ |
| D3 | `pipeline/` must not depend on `config/install/object` | ✅ |
| D4 | `config/` must not depend on `install/object/pipeline/{process,chain,context}` or concrete Filters | ✅ |
| D5 | `install/` must not depend on `pipeline/config` (except `object/vx_reg_props`) | ✅ |
| D6 | `object/` is the only glue layer allowed to depend on all modules; `object/apo/*` only uses authorized sibling references | ⚠️ per table |
| D7 | `telemetry/` depends only on `pipeline/realtime/ring` | ✅ |
| D8 | RT path types (`Filter/Chain/DspContext/RealtimeContext`) must not appear in public API surface | pending (enforce via `lib.rs` visibility: `pub(crate)` RT types, no re-exports; **no api layer is introduced**) |

### 4.3 Automatic validation

`vxapo-driver/scripts/check_deps.ps1` runs before `cargo test`, parses `use crate::` / `crate::` / `super::` references, and asserts per-file whitelist rules from this table.

---

## 5. Data flow

### Install flow

```
CLI/App (user selection)
  -> device/info.rs::enumerate_devices (single device-enumeration entry)
  -> selector/operation.rs::install_endpoint (7-step install, transaction rollback)
      -> audiodg.rs (DisableProtectedAudioDG check)
      -> sys/registry.rs (registry writes + backup)
      -> object/vx_reg_props.rs (read CLSIDs)
```

### Config load (LockForProcess)

```
object/apo.rs::LockForProcess
  -> mutex.lock()
  -> state.transition(Initialized -> Locked)
  -> build RAII guard
  -> pipeline/format.rs (extract format from IAudioMediaType)
  -> build DspContext (pure data, read-only)
  -> config/parser.rs::ConfigParser.parse_file(path, &dsp_ctx)
      -> toml::from_str::<FileModel> -> into_chain_model
          -> factory::create_from_model (static match, construct Filter chain)
  -> pipeline/chain.rs::Chain::new() + add_filter x N + initialize(...)
  -> pipeline/context.rs (save PipelineContext)
  -> preallocate temp_buffers / temp_buffer_old / temp_buffer_new
  -> latency_frames_atomic.store(chain.total_latency())  # v9.12: always 0 reported
  -> ensure_can_load()
  -> guard.disarm(); mutex.unlock()
```

### Config hot reload (watcher triggered, v6.9 R2 blocking style)

```
config/watcher.rs detects config.toml change
  -> object/apo.rs::hot_reload
      -> short lock check transition.is_some() || reloading
          -> yes: return (blocking: do not build new chain during transition)
          -> no: parse outside lock (TOML -> FileModel -> ChainModel -> create_from_model, spec fingerprint short-circuit)
      -> mutex.lock()
      -> race fallback: if new transition started during parse -> pending_reload = true; return
      -> initialize new Chain, put into current_chain, old Chain into outgoing_chain
      -> transition = Some(SmoothingProvider::new(10ms))
      -> reloading = false
      -> mutex.unlock()
```

### Runtime processing (with dual-chain transition)

```
Windows audio engine
  -> object/apo.rs::APOProcess
      -> state.current() == Locked
      -> mutex.lock()
      -> check outgoing_chain

      [normal mode]
      -> process_audio(props, params, current_chain, stats, temp_buffers)
          -> evaluate_buffer -> deinterleave -> is_silent -> mono upmix
          -> chain.process()
          -> interleave -> apply_error_policy

      [transition mode]
      -> process outgoing chain -> temp_buffer_old
      -> process current chain -> temp_buffer_new
      -> transition.advance() -> factor
      -> mix_buffers(old, new, output, factor)
      -> when factor >= 1.0, drop outgoing_chain, clear transition
          -> if pending_reload, release lock and hot_reload()
      -> mutex.unlock()
```

---

## 6. Key design decisions

- Chain does not own buffers. Buffers are preallocated by `ApoObjectInner` in `LockForProcess`.
- Dual-chain transition is managed in the object layer; `SmoothingProvider` only provides the raised-cosine mix factor.
- `DspContext` is constructed by the caller; `config/` does not directly operate Chain.
- Thread-safety model: a single `self.mutex` (`Arc<Mutex<ApoObjectInner>>`, which now also holds format/channel state), `StateCell` (AtomicU8 + CAS), and latency atomics; no nested locks. The former `self.ap_state` lock was removed.

---

## 7. Production build constraints (O3)

```toml
[profile.release]
panic = "abort"      # required: RT panic must not unwind across FFI
codegen-units = 1    # required: predictable inlining/layout, minimal RT jitter
lto = "thin"         # recommended
strip = "debuginfo"  # recommended
```

- RT paths must never unwind.
- Development/test profiles may use `panic = "unwind"` for catch_unwind tests, but release must use abort.
