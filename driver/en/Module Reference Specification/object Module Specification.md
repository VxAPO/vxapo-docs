# object Module Specification

**Boundary**: Glue layer. Windows loads the DLL and creates COM objects. It is allowed to depend on all modules.

**Allowed dependencies**: all other modules (`sys/`, `pipeline/`, `install/`, `config/`, `utils/`, `telemetry/`).

**Forbidden dependencies**: none at the module level; `object/apo/*` submodules must avoid circular object/apo dependencies except as authorized.

## Module tree

```
object/
├── apo.rs               # ApoObject COM object definition (entry)
├── dll_exports.rs       # DllGetClassObject / DllRegisterServer / DllUnregisterServer
├── factory.rs           # ClassFactory (IClassFactory)
├── ref_count.rs         # reference-counting helper
├── vx_reg_props.rs      # VxAPO CLSIDs and registration property helpers
└── apo/
    ├── aggregate.rs     # outer object / aggregate
    ├── child.rs         # child APO COM lifetime management
    ├── config.rs        # config load / hot reload glue
    ├── init.rs          # Initialize / LockForProcess setup
    ├── inner.rs         # ApoObjectInner state
    ├── negotiate.rs     # format negotiation
    ├── process.rs       # APOProcess
    └── state.rs         # StateCell / TransitionError / ApoState (ApoObjectState removed)
```

## 7.1 `object/apo.rs` — ApoObject

The main COM object implementing:
- `IAudioProcessingObject`
- `IAudioProcessingObjectRT`
- `IAudioProcessingObjectConfiguration`
- `IAudioSystemEffects` (marker)
- `IAudioSystemEffects2` (if applicable)

Key responsibilities:
- `Initialize` / `LockForProcess` / `UnlockForProcess`
- `IsInputFormatSupported` / `IsOutputFormatSupported`
- `CalcInputFrames` / `CalcOutputFrames`
- `APOProcess`
- `GetLatency` / `Reset`
- `hot_reload` on config change

State is protected by:
- `self.mutex: Arc<Mutex<ApoObjectInner>>` — chain, transition state, pipeline context, temp buffers, `pending_reload`, `active_spec`, and format/channel state
- `StateCell` (AtomicU8 + CAS)
- latency atomics

> `self.ap_state: Mutex<ApoObjectState>` and the `ApoObjectState` type were removed (0 hits
> repo-wide); format and channel state now lives inside `ApoObjectInner`.

See `apo.rs Thread Safety Model.md` for details.

## 7.2 `object/apo/child.rs`

Manages child APO COM lifetime (PreMixChild/PostMixChild). It creates and releases the original APO that VxAPO preserves as a child, and delegates processing when needed.

## 7.3 `object/factory.rs`

`ClassFactory` implements `IClassFactory` for `CLSID_VXAPO_PRE_MIX` and `CLSID_VXAPO_POST_MIX`. It creates `ApoObject` instances with proper ref counting.

## 7.4 `object/ref_count.rs`

Minimal COM reference-counting helper.

## 7.5 `object/vx_reg_props.rs`

Defines VxAPO CLSIDs:
- PreMix: `{41C34613-D391-459D-A039-72B2B15A1A1D}`
- PostMix: `{B4A97313-ABC0-45ED-9C33-428B20D39428}`

Provides registration property helpers used by install and DLL exports.

## 7.6 `object/dll_exports.rs`

Implements:
- `DllGetClassObject`
- `DllRegisterServer`
- `DllUnregisterServer`
- `register_apo_with_path` (used by CLI to refresh CLSID->DLL binding)

## Hard constraints

1. `object/` is the only module allowed to depend on all other modules.
2. `object/apo/*` must not create circular dependencies with `object/apo.rs` except through the authorized entry/shell pattern.
3. RT paths must not allocate, block, or unwind.
4. Config hot reload must parse outside the lock and swap chain pointers inside the lock.
