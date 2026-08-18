# VxAPO Driver Architecture and Module Specification

## 1. Module overview

`vxapo-driver` is a Windows APO DLL that provides real-time audio DSP to `audiodg.exe`. It is organized as follows:

```text
src/
├── lib.rs          # crate root
├── sys/            # FFI layer: COM, registry, audio definitions
├── pipeline/       # audio processing pipeline
├── install/        # device APO install/uninstall, device query
├── config/         # config file parsing (TOML model)
├── object/         # COM glue: ApoObject, class factory, DLL exports
├── telemetry/      # lock-free logging and panic hook
└── utils/          # alignment, GUID, error types
```

Dependency direction is one-way:

```text
sys  <- pipeline <- config
  ↑       ↑         ↑
  └───────┴─────── object (the only glue layer allowed to depend on everything)
```

## 2. Module responsibilities and dependencies

### 2.1 `sys/`

Depends only on `windows` / `windows-core`; no business awareness.

- `sys/com/prelude.rs`: COM base types, HRESULT constants, safe `guid_to_string`.
- `sys/com/apo_interfaces.rs`: APO interfaces and IID constants.
- `sys/com/apo_types.rs`: APO POD types.
- `sys/registry.rs`: RAII `RegKey` with minimal-permission access.
- `sys/audio_defs.rs`: channel masks, standard layouts, channel name mapping.

### 2.2 `pipeline/`

Processes audio only; does not know install, config, or COM objects.

- `context.rs`: `PipelineContext` with sample rate/channel names.
- `buffer.rs`: audio buffers and preallocation.
- `format.rs`: format extraction and conversion.
- `interleave.rs`: deinterleave/interleave.
- `chain.rs`: `Chain` holds `Vec<Box<dyn Filter>>` and total latency.
- `process.rs`: normal and transition processing.
- `realtime/`: `RealtimeContext` compile-time marker, lock-free ring buffer.
- `dsp/`: `Filter` trait, factory, concrete filters.

`pipeline` must not depend on `install`, `config`, or `object`.

### 2.3 `install/`

Handles device APO install/uninstall and device query.

- `device/endpoint.rs`: endpoint state/name.
- `device/format.rs`: WAVEFORMATEX parsing.
- `device/slots.rs`: 5 APO slots, install modes, child APO path.
- `device/info.rs`: `enumerate_devices` single entry.
- `device/sysfx.rs`: CAPX default-effect template takeover/restore.
- `selector/operation.rs`: `install_endpoint` / `uninstall_endpoint` with transaction rollback.
- `audiodg.rs`: `DisableProtectedAudioDG` check and audio service control.

`install` must not depend on `pipeline` or `config`.

### 2.4 `config/`

Parses `config.toml` into a DSP chain model without directly operating `Chain`.

- `model.rs`: `FileModel`, TOML deserialization target.
- `parser.rs`: `FileModel -> ChainModel -> Filter chain`, produces spec fingerprint.
- `error.rs`: configuration error type.
- `watcher.rs`: watches config directory and triggers hot reload.

`config` must not depend on `install`, `object`, `pipeline/process`, `pipeline/chain`, or `pipeline/context`.

### 2.5 `object/`

COM glue layer called directly by the Windows audio engine.

- `apo.rs`: `ApoObject` implementing APO interfaces.
- `apo/*.rs`: init, negotiate, process, child APO, state submodules.
- `factory.rs`: `IClassFactory`.
- `ref_count.rs`: reference counting.
- `vx_reg_props.rs`: VxAPO CLSIDs and registration properties.
- `dll_exports.rs`: `DllGetClassObject`, registration/unregistration.

`object` is the only layer allowed to depend on all modules.

### 2.6 `telemetry/` and `utils/`

- `telemetry/logger.rs`: lock-free ring log, zero heap allocation on RT.
- `telemetry/panic.rs`: panic hook.
- `utils/align.rs`: SIMD alignment.
- `utils/guid.rs`: GUID reverse parsing.
- `utils/vx_error.rs`: unified error type and HRESULT mapping.

## 3. Dependency rules

| Module | May depend on | Must not depend on |
|--------|---------------|--------------------|
| `sys/` | windows-rs / windows-core | all others |
| `pipeline/` | `sys/`, `utils/` | `install/`, `config/`, `object/` |
| `install/` | `sys/`, `utils/`, `object/vx_reg_props.rs` | `pipeline/`, `config/` |
| `config/` | `sys/`, `utils/`, `pipeline/dsp/{filter,model,factory}` | `install/`, `object/`, `pipeline/{process,chain,context}`, concrete DSP implementations |
| `object/` | all modules | none |
| `telemetry/` | `pipeline/realtime/ring` | `install/`, `config/`, `object/` |
| `utils/` | `windows-core` | all others |

Third-party dependencies:

- `rustfft`: FIR/PEQ block FFT.
- `thiserror`: error enum derive.
- `once_cell`: lazy init in `install/device/sysfx`.
- `serde` / `toml`: config TOML deserialization.
- `windows` family: COM and system APIs.

## 4. Data flow

### 4.1 Install flow

```text
CLI/App
  -> install::device::info::enumerate_devices
  -> install::selector::operation::install_endpoint
       -> audiodg check
       -> registry slot writes
       -> child APO backup/restore
       -> post-install verification
```

### 4.2 Config load and hot reload

```text
LockForProcess
  -> object/apo.rs
  -> pipeline/format.rs extract format
  -> config/parser.rs parse config.toml
       -> FileModel -> ChainModel -> factory::create_from_model
  -> pipeline/chain.rs build Filter chain
  -> preallocate temp buffers
  -> start watcher

watcher detects config.toml change
  -> parse new config outside lock
  -> compare with current spec fingerprint
  -> build new chain and enter dual-chain transition if different
```

### 4.3 Real-time processing

```text
Windows audio engine
  -> object/apo.rs::APOProcess
  -> check state and lock
  -> process_audio (normal) or dual-chain transition
  -> deinterleave -> Filter chain -> interleave
  -> error policy
```

## 5. Thread safety model

- `self.mutex`: protects `ApoObjectInner` (chain, transition, temp buffers).
- `self.ap_state`: protects format/channel/lock state.
- `StateCell`: `AtomicU8` + CAS lock-free state machine.
- Latency values use `AtomicU32`; RT path reads without locks.
- The two mutexes are never held simultaneously.
- `hot_reload` parses outside the lock and swaps chain pointers inside the lock.

## 6. Key design decisions

- `Chain` does not own buffers; buffers are preallocated by `ApoObjectInner`.
- Dual-chain transition is managed in the object layer; `SmoothingProvider` only provides the mix factor.
- `DspContext` is constructed by the caller; `config` does not directly operate `Chain`.
- Production builds must use `panic = "abort"` and `codegen-units = 1`; RT path must not unwind.
- Latency is not reported: `latency_frames_atomic` is always 0 to avoid frame negotiation misalignment.

> For the detailed per-module developer guide, see `driver/en/Module Reference Specification/`.
