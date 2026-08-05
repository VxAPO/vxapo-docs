## 八、`utils/` 模块规范

**边界**：不依赖任何其他模块

**允许依赖**：`windows-core`（`vx_error.rs` 中的 `check_hresult`、`guid.rs` 中的 `GUID`）

**禁止依赖**：`sys/`、`pipeline/`、`install/`、`config/`、`object/`、`telemetry/`

### 完整模块树

```
utils/
├── align.rs       # SIMD 对齐工具
├── guid.rs        # GUID 纯解析工具（字节/字符串 → GUID）
└── vx_error.rs    # VxApoError 业务错误
```

### 8.1 `utils/align.rs`

```rust
pub const SIMD_ALIGN: usize = 16;
pub const SIMD_ALIGN_AVX: usize = 32;
pub fn align_offset(ptr: usize, align: usize) -> usize;
pub fn is_aligned(ptr: *const f32, align: usize) -> bool;
```

### 8.2 `utils/vx_error.rs`

```rust
#[derive(Debug, thiserror::Error)]
pub enum VxApoError {
    #[error("注册表错误: {0}")] Registry(String),
    #[error("配置解析错误: {0}")] Config(String),
    #[error("格式不支持: {0}")] Format(String),
    #[error("I/O 错误: {0}")] Io(String),
    #[error("内部错误: {0}")] Internal(String),
    #[error("实时安全违规: {0}")] RtSafety(String),
    #[error("状态转换错误: {0}")] State(String),
    #[error("设备未找到: {0}")] DeviceNotFound(String),
}

pub type Result<T> = core::result::Result<T, VxApoError>;

pub fn succeeded(hr: HRESULT) -> bool {
    hr.0 >= 0
}

pub fn failed(hr: HRESULT) -> bool {
    hr.0 < 0
}

pub fn check_hresult(hr: HRESULT) -> Result<()> {
    if succeeded(hr) {
        Ok(())
    } else {
        Err(VxApoError::Internal(format!(
            "HRESULT 错误: 0x{:08X}",
            hr.0
        )))
    }
}

// VxApoError → HRESULT 映射（供 COM 方法返回）
impl From<VxApoError> for HRESULT {
    fn from(err: VxApoError) -> Self {
        match err {
            VxApoError::Registry(_) => E_FAIL,
            VxApoError::Config(_) => E_FAIL,
            VxApoError::Format(_) => APOERR_FORMAT_NOT_SUPPORTED,
            VxApoError::Io(_) => E_FAIL,
            VxApoError::Internal(_) => E_UNEXPECTED,
            VxApoError::RtSafety(_) => E_FAIL,
            VxApoError::State(_) => APOERR_ALREADY_INITIALIZED,
            VxApoError::DeviceNotFound(_) => E_FAIL,
        }
    }
}
```

> GUID 字符串化统一走 `sys/com/prelude::guid_to_string`（`StringFromGUID2` 安全收窄）；`utils/guid` 只做反向解析（字节/字符串 → GUID），不重复实现正向格式化。

### 8.3 `utils/guid.rs`

**职责**：GUID 反向解析纯函数（16 字节小端原始数据 → GUID、`{XXXXXXXX-...}` 字符串 → GUID）。不包含 I/O、不包含业务逻辑、不感知 APO/注册表/COM。

**引用来源**：
- `windows::core::GUID`

**导出给**：`install/device/slots.rs` 等需要把注册表二进制/字符串解析为 GUID 的模块。

**公开 API**：
```rust
/// 16 字节小端（data1/data2/data3）+ data4 原始 → GUID。
pub fn guid_from_bytes(bytes: &[u8]) -> Option<GUID>;

/// GUID 是否为全零（Windows「无 APO」占位）。
pub fn is_zero_guid(g: &GUID) -> bool;

/// 解析 `{XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX}` 格式 GUID 字符串。
pub fn parse_guid_string(s: &str) -> Option<GUID>;
```

> 反向解析（字节/字符串 → GUID）统一收口到 `utils/guid`；正向格式化（GUID → 字符串）仍由 `sys/com/prelude::guid_to_string` 负责，避免在安装层重复实现解析逻辑。
