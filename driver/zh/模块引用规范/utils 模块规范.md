## 八、`utils/` 模块规范

**边界**：不依赖任何其他模块

**允许依赖**：`windows-core`（`vx_error.rs` 中的 `HRESULT` 转换与 `E_FAIL` / `E_UNEXPECTED` 常量）、`windows`（`guid.rs` 中的 `windows::core::GUID`）

**禁止依赖**：`sys/`、`pipeline/`、`install/`、`config/`、`object/`、`telemetry/`

### 完整模块树

```
utils/
├── guid.rs        # GUID 纯解析工具（字节/字符串 → GUID）
├── ring.rs        # SPSC 无锁环形缓冲
└── vx_error.rs    # VxApoError 业务错误
```

### 8.1 （原 `utils/align.rs`：已删除）

> **2026-10-02 删除。** 原 `align.rs` 提供 `AlignedBuffer` / `SIMD_ALIGN` 的自定义
> 对齐分配。删除理由（实测）：
>
> 1. **生产路径零消费者**——`AlignedBuffer` / `SIMD_ALIGN` 仅出现在其自身 `mod tests`；
>    `pipeline/**` 对它的引用数为 0。
> 2. **真正的 SIMD 路径不需要它**——`pipeline/dsp/fir.rs` 的 AVX2 FMA 点积
>    （`dot_avx2_fma`）使用的是 **`_mm256_loadu_ps`（非对齐加载）**，全仓对齐加载
>    （`_mm256_load_ps` / `_mm_load_ps`）出现 **0 次**。即对齐并非本项目的性能前提。
> 3. **本节原先描述的 API 与代码早已不符**（`SIMD_ALIGN = 16` 对实际的 32；
>    `align_offset` / `is_aligned` 两个函数在全仓**不存在**）——说明该模块长期无人维护。
>
> 本删除是一个独立提交，可用 `git revert` 单独恢复。

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

// ⚠ 以下三个函数已全部移除（全仓搜索 0 处）：succeeded / failed / check_hresult
// 现行：HRESULT 判定直接使用 windows-rs 方法；错误反向映射见下方 `impl From<VxApoError> for HRESULT`
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
