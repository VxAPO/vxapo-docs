## 八、`utils/` 模块规范

**边界**：不依赖任何其他模块

**允许依赖**：`windows-core`（仅 `vx_error.rs` 中的 `check_hresult`）

**禁止依赖**：`sys/`、`pipeline/`、`install/`、`config/`、`object/`、`telemetry/`

### 完整模块树

```
utils/
├── align.rs       # SIMD 对齐工具
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

> GUID 转换直接使用 `windows::core::GUID` 的 `Display` trait（`format!("{guid}")` 输出标准格式）和 `GUID::from_values` 构造，无需自定义工具函数。`sys/registry.rs` 中的 `get_guid_string` 内部通过 `GUID::from_values` 和 `Display` 完成二进制到字符串的转换。