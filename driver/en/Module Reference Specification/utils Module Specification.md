# utils Module Specification

**Boundary**: Does not depend on any other module.

**Allowed dependencies**: `windows-core` (`HRESULT` conversions in `vx_error.rs`; the `E_FAIL` / `E_UNEXPECTED` constants are defined locally), `windows` (`windows::core::GUID` in `guid.rs`).

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

// ── APO-specific HRESULT codes (single source: sys/consts.rs) ──────────────
pub(crate) use crate::sys::consts::{APOERR_ALREADY_INITIALIZED, APOERR_FORMAT_NOT_SUPPORTED};

const E_FAIL: windows_core::HRESULT = windows_core::HRESULT(0x8000_4005u32 as i32);
const E_UNEXPECTED: windows_core::HRESULT = windows_core::HRESULT(0x8000_FFFFu32 as i32);

// ⚠ The following three helpers have been removed entirely (0 hits repo-wide):
//   succeeded / failed / check_hresult.
// Current practice: HRESULT checks use windows-rs methods directly; the reverse
// mapping is the `impl From<VxApoError> for HRESULT` below.

// VxApoError -> HRESULT mapping (returned from COM methods)
impl From<VxApoError> for HRESULT {
    fn from(err: VxApoError) -> Self {
        match err {
            VxApoError::Registry(_) => E_FAIL,
            VxApoError::Config(_) => E_FAIL,
            VxApoError::Format(_) => APOERR_FORMAT_NOT_SUPPORTED,   // 0x887D_0003
            VxApoError::Io(_) => E_FAIL,
            VxApoError::Internal(_) => E_UNEXPECTED,               // 0x8000_FFFF
            VxApoError::RtSafety(_) => E_FAIL,
            VxApoError::State(_) => APOERR_ALREADY_INITIALIZED,    // 0x887D_0001
            VxApoError::DeviceNotFound(_) => E_FAIL,
        }
    }
}

// ── Helper constructors (one per variant) ──────────────────────────────────
impl VxApoError {
    pub fn registry(msg: impl Into<String>) -> Self;
    pub fn config(msg: impl Into<String>) -> Self;
    pub fn format(msg: impl Into<String>) -> Self;
    pub fn io(msg: impl Into<String>) -> Self;
    pub fn internal(msg: impl Into<String>) -> Self;
    pub fn rt_safety(msg: impl Into<String>) -> Self;
    pub fn state(msg: impl Into<String>) -> Self;
    pub fn device_not_found(name: impl Into<String>) -> Self;
}
```

> **No implicit `From<windows_core::Error>` normalization.** Historically such a conversion
> mislabeled service/COM/file errors as "registry error", masking the real failing layer.
> Now: registry semantics are classified as `Registry` inside `sys/registry.rs` (that layer
> is the single source); other Windows API call sites construct `Internal` / `Io` explicitly
> by semantics.

> GUID stringification is centralized in `sys/com/prelude::guid_to_string` (`StringFromGUID2` safe wrapper). `utils/guid` only performs reverse parsing (bytes/string -> GUID) and does not re-implement forward formatting.

## 8.3 `utils/guid.rs`

**Responsibility**: Pure GUID reverse-parsing functions (16-byte little-endian raw data -> GUID, `{XXXXXXXX-...}` string -> GUID). No I/O, no business logic, no APO/registry/COM awareness.

**References**:
- `windows::core::GUID`

**Exports to**: modules that need to parse registry binary/string GUIDs, e.g. `install/device/slots.rs`.

**Public API**:
```rust
/// 16-byte little-endian (data1/data2/data3) + raw data4 -> GUID.
pub fn guid_from_bytes(bytes: &[u8]) -> Option<GUID>;

/// Whether the GUID is all-zero (the Windows "no APO" placeholder).
pub fn is_zero_guid(g: &GUID) -> bool;

/// Parses a `{XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX}` GUID string.
/// Strictly validates the 8-4-4-4-12 hex layout; non-hex characters yield `None`
/// (no panic — a former per-character write overflowed `data4: [u8; 8]` and was fixed).
pub fn parse_guid_string(s: &str) -> Option<GUID>;
```

> Reverse parsing (bytes/string -> GUID) is centralized in `utils/guid`; forward formatting (GUID -> string) remains in `sys/com/prelude::guid_to_string`, avoiding duplicated parsing logic in the install layer.
