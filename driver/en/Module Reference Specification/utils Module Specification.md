# utils Module Specification

**Boundary**: Does not depend on any other module.

**Allowed dependencies**: `windows-core` (`HRESULT` conversions in `vx_error.rs`; the old `check_hresult` helper no longer exists), `windows` (`windows::core::GUID` in `guid.rs`).

**Forbidden dependencies**: `sys/`, `pipeline/`, `install/`, `config/`, `object/`, `telemetry/`.

## Module tree

```
utils/
├── guid.rs        # Pure GUID parsing helpers (bytes/string -> GUID)
├── ring.rs        # SPSC lock-free ring buffer
└── vx_error.rs    # VxApoError business error type
```

## 8.1 (former `utils/align.rs`: removed)

> **Removed 2026-10-02.** The former `align.rs` provided a custom aligned allocator
> (`AlignedBuffer` / `SIMD_ALIGN`). Removal rationale (measured):
>
> 1. **Zero production consumers** — `AlignedBuffer` / `SIMD_ALIGN` appeared only in the
>    module's own `mod tests`; `pipeline/**` referenced them 0 times.
> 2. **The real SIMD path does not need it** — the AVX2 FMA dot product
>    (`dot_avx2_fma` in `pipeline/dsp/fir.rs`) uses **`_mm256_loadu_ps` (unaligned
>    load)**; aligned loads (`_mm256_load_ps` / `_mm_load_ps`) appear **0 times**
>    repository-wide. Alignment is therefore not a performance premise of this project.
> 3. **This section already disagreed with the code** (`SIMD_ALIGN = 16` vs the actual
>    32; `align_offset` / `is_aligned` do not exist anywhere in the repository) —
>    the module had long been unmaintained.
>
> The removal is a standalone commit and can be restored on its own with `git revert`.

## 8.2 `utils/vx_error.rs`

```rust
#[derive(Debug, thiserror::Error)]
pub enum VxApoError {
    #[error("registry error: {0}")] Registry(String),
    #[error("config parse error: {0}")] Config(String),
    #[error("unsupported format: {0}")] Format(String),
    #[error("I/O error: {0}")] Io(String),
    #[error("internal error: {0}")] Internal(String),
    #[error("real-time safety violation: {0}")] RtSafety(String),
    #[error("state transition error: {0}")] State(String),
    #[error("device not found: {0}")] DeviceNotFound(String),
}

pub type Result<T> = core::result::Result<T, VxApoError>;

pub fn succeeded(hr: HRESULT) -> bool { hr.0 >= 0 }
pub fn failed(hr: HRESULT) -> bool { hr.0 < 0 }
pub fn check_hresult(hr: HRESULT) -> Result<()> { /* ... */ }

impl From<VxApoError> for HRESULT { /* ... */ }
```

> GUID stringification is centralized in `sys/com/prelude::guid_to_string` (`StringFromGUID2` safe wrapper). `utils/guid` only performs reverse parsing (bytes/string -> GUID) and does not re-implement forward formatting.

## 8.3 `utils/guid.rs`

**Responsibility**: Pure GUID reverse parsing functions (16-byte little-endian raw data -> GUID, `{XXXXXXXX-...}` string -> GUID). No I/O, no business logic, no APO/registry/COM awareness.

**References**:
- `windows::core::GUID`

**Exports to**: modules that need to parse registry binary/string GUIDs, e.g. `install/device/slots.rs`.

**Public API**:
```rust
pub fn guid_from_bytes(bytes: &[u8]) -> Option<GUID>;
pub fn is_zero_guid(g: &GUID) -> bool;
pub fn parse_guid_string(s: &str) -> Option<GUID>;
```

> Reverse parsing (bytes/string -> GUID) is centralized in `utils/guid`; forward formatting (GUID -> string) remains in `sys/com/prelude::guid_to_string`, avoiding duplicated parsing logic in the install layer.
