# VxAPO Driver Code Review Summary

This document consolidates historical code-review findings. The current implementation is authoritative.

## Key findings

### `sys/`

- COM interfaces and IIDs are centralized.
- Registry access uses minimal SAM to match MMDevices ACLs.
- GUID formatting is centralized in `sys/com/prelude::guid_to_string`.

### `pipeline/`

- `Chain` does not own buffers; buffers are preallocated by the object layer.
- RT path forbids heap allocation, locks, and I/O.
- DSP numerical safety uses parse-layer clamp, f64 coefficient computation, and RT FTZ/DAZ.
- PEQ uses a 200 Hz crossover hybrid architecture.
- Latency is not reported to avoid frame negotiation misalignment.

### `install/`

- `enumerate_devices` is the single enumeration entry.
- Install/uninstall use transaction protection with automatic rollback.
- Only VxAPO-owned parts are uninstalled.
- `select.rs` is removed; interactive selection is handled by CLI/App.

### `config/`

- Old `config.txt` command system is removed; TOML is the only model.
- `FileModel -> ChainModel` conversion happens in config.
- Spec fingerprint contains only DSP fields.

### `object/`

- `ApoObject` is the multi-module glue layer.
- Child APOs are managed under `HKLM\SOFTWARE\VxAPO\Child APOs`.
- DLL registration must write `AudioEngine\AudioProcessingObjects`.

### `telemetry/` and `utils/`

- Logging uses a lock-free ring buffer, RT-safe.
- Errors are unified as `VxApoError`.
- GUID parsing and formatting responsibilities are separated.
