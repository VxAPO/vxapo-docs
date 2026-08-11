## 三、`sys/` 模块规范（最终版）

**边界**：不知道 VxAPO 是什么。不知道音频处理。不知道安装。不知道配置。

**允许依赖**：`windows` crate、`windows-core` crate、`core`、`alloc`

**禁止依赖**：`pipeline/`、`install/`、`config/`、`object/`、`utils/`、`telemetry/`

---

### 完整模块树

```
sys/
├── com.rs                   # COM 子系统模块入口
├── registry.rs              # 注册表模块（单一文件，含 win32_ok + RegKey + 读写删）
├── audio_defs.rs            # Windows 音频基础定义（通道掩码位标志 + 标准布局映射）
├── known_folder.rs          # 已知文件夹路径解析（SHGetKnownFolderPath 安全收窄）
└── com/
    ├── prelude.rs           # COM 基础类型重导出 + HRESULT 常量
    ├── apo_interfaces.rs    # 4 个 APO 接口定义 + 7 个 IID 常量
    └── apo_types.rs         # POD 结构体 + 枚举 + windows-rs 类型重导出 + 常量
```

---

### 引用约束总表

| 模块 | 可依赖 | 不可依赖 |
|------|--------|----------|
| `sys/audio_defs.rs` | `windows` crate | 所有其他 |
| `sys/com/prelude.rs` | `windows` crate | 所有其他 |
| `sys/com/apo_interfaces.rs` | `prelude`、`apo_types`、`windows` crate（3 个系统接口仅取 IID） | 其他 |
| `sys/com/apo_types.rs` | `windows` crate | 其他 |
| `sys/registry.rs` | `windows` crate、`windows-core` crate、`sys/com/prelude`（`guid_to_string`） | 其他 |
| `sys/known_folder.rs` | `windows` crate | 其他 |

---

### 3.1 `sys/com/prelude.rs`

**职责**：重导出 `windows-rs` 的 COM 基础类型、HRESULT 常量，以及 GUID 格式化安全操作辅助

**引用来源**：
- `windows::core::{IUnknown, IUnknown_Vtbl, Interface, interface, GUID, HRESULT, implement}`
- `windows::Win32::System::Com::{IClassFactory, StringFromGUID2}`

**导出给**：`sys/com/` 下所有子模块、以及所有需要 GUID 格式化的模块——`sys/registry.rs`、`install/selector/operation.rs`、`install/device/slots.rs` 等
>
> **允许各层必要依赖**：`guid_to_string` 本质是 `StringFromGUID2`（FFI unsafe 调用）的安全收窄重导出，是重导出性质的规范函数。任何模块在**确实需要 GUID 标准字符串格式化**时均可必要依赖（各使用方模块的约束表需显式列出）。

**公开 API**：

```rust
// ── 类型重导出 ──
pub use windows::core::{
    IUnknown, IUnknown_Vtbl, Interface, interface, GUID, HRESULT, implement,
};
pub use windows::Win32::System::Com::{IClassFactory, StringFromGUID2};
pub use windows::Win32::System::Com::{
    CLSCTX_ALL, CLSCTX_INPROC_SERVER, COINIT_MULTITHREADED, CoCreateInstance, CoInitializeEx,
    CoTaskMemAlloc, CoTaskMemFree, IClassFactory_Impl,
};

// ── GUID 格式化辅助（唯一允许的"自定义函数"特例） ──
/// 将 GUID 格式化为标准字符串 `{XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX}`。
///
/// 理由：`windows::core::GUID` 未实现 `Display`/`ToString`，若直接使用 `Debug`
/// 输出依赖不稳定格式；官方 FFI 绑定 `StringFromGUID2` 需要 `unsafe` 调用，
/// 此函数将该 `unsafe` 收窄为单一安全边界，属系统层职责。
pub fn guid_to_string(g: &GUID) -> String;

// ── HRESULT 常量 ──
pub const S_OK: HRESULT = HRESULT(0);
pub const S_FALSE: HRESULT = HRESULT(1);
pub const E_NOINTERFACE: HRESULT = HRESULT(0x8000_4002u32 as i32);
pub const E_POINTER: HRESULT = HRESULT(0x8000_4003u32 as i32);
pub const E_FAIL: HRESULT = HRESULT(0x8000_4005u32 as i32);
pub const E_UNEXPECTED: HRESULT = HRESULT(0x8000_FFFF_u32 as i32);
pub const E_INVALIDARG: HRESULT = HRESULT(0x8007_0057u32 as i32);
pub const CLASS_E_CLASSNOTAVAILABLE: HRESULT = HRESULT(0x8004_0111u32 as i32);
pub const E_OUTOFMEMORY: HRESULT = HRESULT(0x8007_000Eu32 as i32);
pub const CLASS_E_NOAGGREGATION: HRESULT = HRESULT(0x8004_0110u32 as i32);
pub const SELFREG_E_CLASS: HRESULT = HRESULT(0x8004_0201u32 as i32);

// ── APO 专用 HRESULT 错误码（统一定义在 prelude，apo_types 仅重导出） ──
pub const APOERR_ALREADY_INITIALIZED:          HRESULT = HRESULT(0x887D_0001u32 as i32);
pub const APOERR_NOT_INITIALIZED:              HRESULT = HRESULT(0x887D_0002u32 as i32);
pub const APOERR_FORMAT_NOT_SUPPORTED:         HRESULT = HRESULT(0x887D_0003u32 as i32);
pub const APOERR_INVALID_APO_CLSID:            HRESULT = HRESULT(0x887D_0004u32 as i32);
pub const APOERR_BUFFERS_OVERLAP:              HRESULT = HRESULT(0x887D_0005u32 as i32);
pub const APOERR_ALREADY_UNLOCKED:             HRESULT = HRESULT(0x887D_0006u32 as i32);
pub const APOERR_NUM_CONNECTIONS_INVALID:      HRESULT = HRESULT(0x887D_0007u32 as i32);
pub const APOERR_INVALID_OUTPUT_MAXFRAMECOUNT: HRESULT = HRESULT(0x887D_0008u32 as i32);
pub const APOERR_INVALID_CONNECTION_FORMAT:    HRESULT = HRESULT(0x887D_0009u32 as i32);
pub const APOERR_APO_LOCKED:                   HRESULT = HRESULT(0x887D_000Au32 as i32);
pub const APOERR_INVALID_COEFFCOUNT:           HRESULT = HRESULT(0x887D_000Bu32 as i32);
pub const APOERR_INVALID_COEFFICIENT:          HRESULT = HRESULT(0x887D_000Cu32 as i32);
pub const APOERR_INVALID_CURVE_PARAM:          HRESULT = HRESULT(0x887D_000Du32 as i32);
pub const APOERR_INVALID_INPUTID:              HRESULT = HRESULT(0x887D_000Eu32 as i32);
```

**禁止**：不包含任何自定义类型或业务逻辑。**唯一例外**：`guid_to_string` 函数——
GUID 未实现 `Display` 的必要安全操作扩展，属系统层职责；其余不得新增函数。
>
> **消费边界**：`guid_to_string` 允许**任意模块必要依赖**（本质是 `StringFromGUID2` 的安全收窄重导出）；使用方模块的约束表需显式列出该依赖。

---

### 3.2 `sys/com/apo_interfaces.rs`

**职责**：re-export windows-rs 0.62.2 已提供的 4 个 APO 接口结构体 + 3 个对应 `*_Impl` traits（`#[implement]` 实现对象必需）；导出全部 7 个接口 IID 常量

**引用来源**：
- `crate::sys::com::prelude::*`
- `crate::sys::com::apo_types::*`（`REFERENCE_TIME`、`APO_REG_PROPERTIES`、`APO_CONNECTION_DESCRIPTOR`、`APO_CONNECTION_PROPERTY`、`UNCOMPRESSED_AUDIO_FORMAT`）
- `windows::Win32::Media::Audio::Apo`（4 个 APO 接口结构体直接 re-export：`IAudioMediaType`、`IAudioProcessingObject`、`IAudioProcessingObjectRT`、`IAudioProcessingObjectConfiguration`）
- `windows::Win32::Media::Audio::Apo::{IAudioProcessingObjectNotifications, IAudioSystemEffects, IAudioSystemEffects2}`（3 个系统接口结构体 re-export，仅取 IID）
- `windows::core::{Interface, IUnknown}`（取 `::IID`、作为接口根基）

**导出给**：`object/apo.rs`、`object/child.rs`、`object/factory.rs`

---

#### `*_Impl` traits re-export（`#[implement]` 实现 COM 对象必需）

```rust
pub use windows::Win32::Media::Audio::Apo::{
    IAudioProcessingObject_Impl,
    IAudioProcessingObjectRT_Impl,
    IAudioProcessingObjectConfiguration_Impl,
};
```

> 这三个 `*_Impl` trait 由 windows-rs `define_interface!` 宏生成，`#[implement(...)]` 派生时需要；
> 自定义 APO 对象（`object/apo.rs`）通过实现这些 trait 提供方法。
>
> **措辞澄清**：允许 re-export `*_Impl` traits（仅方法签名，供实现方实现/派生），
> 禁止的是**业务函数体实现**（即本文件不得写任何 COM 方法的具体实现逻辑）。

#### 接口 re-export（windows-rs 结构体，4 个）

**`IAudioMediaType`**（IID: `4e997f73-b71f-4798-873b-ed7dfcf15b4d`）：

| 方法 | 签名 | 说明 |
|------|------|------|
| `IsCompressedFormat` | `fn IsCompressedFormat(&self, pf_compressed: *mut u32) -> HRESULT` | 判断是否为压缩格式 |
| `IsEqual` | `fn IsEqual(&self, p_type: *mut IAudioMediaType, pdw_flags: *mut u32) -> HRESULT` | 比较两个媒体类型 |
| `GetAudioFormat` | `fn GetAudioFormat(&self) -> *const c_void` | 返回只读 `WAVEFORMATEX*` |
| `GetUncompressedAudioFormat` | `fn GetUncompressedAudioFormat(&self, p_format: *mut UNCOMPRESSED_AUDIO_FORMAT) -> HRESULT` | 获取未压缩格式描述 |

**`IAudioProcessingObject`**（IID: `fd7f2b29-24d0-4b5c-b177-592c39f9ca10`）：

| 方法 | 签名 | 说明 |
|------|------|------|
| `Reset` | `fn Reset(&self) -> HRESULT` | 重置 APO 内部状态 |
| `GetLatency` | `fn GetLatency(&self, p_latency: *mut REFERENCE_TIME) -> HRESULT` | 获取延迟（100ns 单位） |
| `GetRegistrationProperties` | `fn GetRegistrationProperties(&self, pp_props: *mut *mut APO_REG_PROPERTIES) -> HRESULT` | 获取注册属性，调用方负责 `CoTaskMemFree` |
| `Initialize` | `fn Initialize(&self, cb_data_size: u32, pby_data: *mut u8) -> HRESULT` | 初始化 |
| `IsInputFormatSupported` | `fn IsInputFormatSupported(&self, p_opposite: *mut IAudioMediaType, p_requested: *mut IAudioMediaType, pp_supported: *mut *mut IAudioMediaType) -> HRESULT` | 查询输入格式支持，`S_FALSE` 表示返回替代格式 |
| `IsOutputFormatSupported` | `fn IsOutputFormatSupported(&self, p_opposite: *mut IAudioMediaType, p_requested: *mut IAudioMediaType, pp_supported: *mut *mut IAudioMediaType) -> HRESULT` | 查询输出格式支持 |
| `GetInputChannelCount` | `fn GetInputChannelCount(&self, p_count: *mut u32) -> HRESULT` | 获取输入通道数 |

**`IAudioProcessingObjectRT`**（IID: `9e1d6a6d-ddbc-4e95-a4c7-ad64ba37846c`）：

| 方法 | 签名 | 说明 |
|------|------|------|
| `APOProcess` | `fn APOProcess(&self, num_input: u32, pp_inputs: *mut *mut APO_CONNECTION_PROPERTY, num_output: u32, pp_outputs: *mut *mut APO_CONNECTION_PROPERTY)` | 返回 void（实时路径不允许错误传播） |
| `CalcInputFrames` | `fn CalcInputFrames(&self, output_frames: u32) -> u32` | 给定输出帧数计算需要的输入帧数 |
| `CalcOutputFrames` | `fn CalcOutputFrames(&self, input_frames: u32) -> u32` | 给定输入帧数计算可产生的输出帧数 |

**`IAudioProcessingObjectConfiguration`**（IID: `0e5ed805-aba6-49c3-8f9a-2b8c889c4fa8`）：

| 方法 | 签名 | 说明 |
|------|------|------|
| `LockForProcess` | `fn LockForProcess(&self, num_input: u32, pp_inputs: *mut *mut APO_CONNECTION_DESCRIPTOR, num_output: u32, pp_outputs: *mut *mut APO_CONNECTION_DESCRIPTOR) -> HRESULT` | 锁定处理流程 |
| `UnlockForProcess` | `fn UnlockForProcess(&self) -> HRESULT` | 解锁处理流程 |

---

#### IID 导出常量（7 个）

**APO 接口 IID（4 个）**：

```rust
pub const IID_IAPO: GUID = IAudioProcessingObject::IID;
pub const IID_IAPO_RT: GUID = IAudioProcessingObjectRT::IID;
pub const IID_IAPO_CONFIG: GUID = IAudioProcessingObjectConfiguration::IID;
pub const IID_IAUDIO_MEDIA_TYPE: GUID = IAudioMediaType::IID;
```

**系统接口 IID（3 个，从 `windows-rs` 直接引用，不自行定义接口 trait）**：

```rust
use windows::Win32::Media::Audio::Apo::{
    IAudioProcessingObjectNotifications, IAudioSystemEffects, IAudioSystemEffects2,
};

pub const IID_IAUDIO_SYSTEM_EFFECTS: GUID = IAudioSystemEffects::IID;
pub const IID_IAUDIO_SYSTEM_EFFECTS2: GUID = IAudioSystemEffects2::IID;
pub const IID_IAUDIO_PROCESSING_OBJECT_NOTIFICATIONS: GUID =
    IAudioProcessingObjectNotifications::IID;
```

> 这 3 个接口由 Windows 实现（非 APO 实现），APO 侧不需要 trait 定义，仅需 IID 用于 `QueryInterface` 查询。

#### AEC 接口预留（O4，v6.6）

Windows 11 AEC（声学回声消除）APO 未来若支持，需在此预留以下接口 re-export 与 IID 常量
（当前**不实现**，仅声明与门控位置，避免未来大改）：

```rust
// feature-gated：`#[cfg(feature = "aec")]`（非默认 feature）
pub use windows::Win32::Media::Audio::Apo::{
    IApoAcousticEchoCancellation,
    IApoAuxiliaryInputConfiguration,
    IApoAuxiliaryInputRT,
};

pub const IID_IAPO_ACOUSTIC_ECHO_CANCELLATION: GUID = IApoAcousticEchoCancellation::IID;
pub const IID_IAPO_AUXILIARY_INPUT_CONFIGURATION: GUID = IApoAuxiliaryInputConfiguration::IID;
pub const IID_IAPO_AUXILIARY_INPUT_RT: GUID = IApoAuxiliaryInputRT::IID;
```

> **门控设计**：`feature = "aec"` 非默认——非 AEC 构建不得引入 Windows 11 SDK 接口面。
> 实现时 `object/apo.rs` 三接口 + 3 个 AEC 接口 = **六接口**承载（对应 tympan-apo 的
> `AecApoInstanceCom` 九接口：六 SISO + 三 AEC）。
> 当前仅声明位置，不实现逻辑（与 object 目标态一致）。

**禁止**：不包含任何实现逻辑

---

### 3.3 `sys/com/apo_types.rs`

**职责**：**re-export windows-rs 0.62.2 已提供的 APO 类型（POD 结构体、APO_FLAG、APO_BUFFER_FLAGS 及其常量）**；仅自定义 windows-rs 缺失项（UNCOMPRESSED_AUDIO_FORMAT、AUDIO_FLOW_TYPE、REFERENCE_TIME、签名/比较常量、APOERR 错误码）

**引用来源**：
- `windows::core::{GUID, HRESULT}`
- `windows::Win32::Media::Audio::Apo::{APO_FLAG, APO_BUFFER_FLAGS, APO_REG_PROPERTIES, APO_CONNECTION_DESCRIPTOR, APO_CONNECTION_PROPERTY}`（及全部关联常量，直接 re-export）
- `windows::Win32::Media::Audio::Apo::APOInitSystemEffects`（Initialize 初始化结构体，含 `pAPOSystemEffectsProperties`（IPropertyStore）、设备 GUID 提取）

**导出给**：`sys/com/apo_interfaces.rs`、`pipeline/`、`install/`、`object/`、`config/`

---

#### 3.3.1 从 `windows-rs` 重导出的类型

**`APO_FLAG`**（直接使用 SDK 类型，不自定义）：

```rust
pub use windows::Win32::Media::Audio::Apo::{
    APO_FLAG,
    APO_FLAG_NONE,
    APO_FLAG_INPLACE,
    APO_FLAG_SAMPLESPERFRAME_MUST_MATCH,
    APO_FLAG_FRAMESPERSECOND_MUST_MATCH,
    APO_FLAG_BITSPERSAMPLE_MUST_MATCH,
    APO_FLAG_MIXER,
    APO_FLAG_DEFAULT,
};
```

> `windows-rs` 的 `APO_FLAG` 是 `repr(transparent)` 包装 `i32`，关联常量完整覆盖 SDK 枚举全部值，支持位运算。无需自定义。
>
> **流向**：单向输出。APO 构建 `APO_REG_PROPERTIES` 时设置 `Flags` 字段 → Windows 引擎读取。Windows 不会向 APO 回写 `APO_FLAG`。

**`APO_BUFFER_FLAGS` 及关联常量**（windows-rs 类型，直接使用，不自定义枚举）：

```rust
pub use windows::Win32::Media::Audio::Apo::{
    APO_BUFFER_FLAGS,
    BUFFER_INVALID,   // 0
    BUFFER_VALID,     // 1
    BUFFER_SILENT,    // 2
};
// 供内部语义引用：BUFFER_INVALID / BUFFER_VALID / BUFFER_SILENT
```

**`WinAPO_BUFFER_FLAGS`**（windows-rs 类型别名，供对外交互时使用）：

```rust
pub use windows::Win32::Media::Audio::Apo::APO_BUFFER_FLAGS as WinAPO_BUFFER_FLAGS;
```

**POD 结构体**（windows-rs 0.62.2 已提供，直接 re-export，不自定义）：

```rust
pub use windows::Win32::Media::Audio::Apo::{
    APO_REG_PROPERTIES,          // 注册属性（1092 字节 conformant array）
    APO_CONNECTION_DESCRIPTOR,   // 连接描述符（x64=40 / x86=20 字节）
    APO_CONNECTION_PROPERTY,     // 连接属性（x64=24 / x86=16 字节）
};
pub use windows::Win32::Media::Audio::{WAVEFORMATEX, WAVEFORMATEXTENSIBLE};
```

> 结构体大小/偏移由 windows-rs 绑定保证，**不再需要** 3.3.8 中针对这些结构体的编译期断言。

---

#### 3.3.1b `APOInitSystemEffects`（Initialize 初始化数据）

```rust
pub use windows::Win32::Media::Audio::Apo::APOInitSystemEffects;
```

> 由 windows-rs 0.62.2 提供，直接 re-export 不自定义。Initialize（`object 7.1.8`）接收
> `pby_data` 指向 `APOInitSystemEffects`，从 **`pAPOEndpointProperties`**（`IPropertyStore`）
> 取 `PKEY_AudioEndpoint_GUID`（Windows 11 实测为 VT_LPWSTR；兼容 VT_CLSID）提取设备 GUID
> （v7.2 定义，v7.6 修订为实测路径，v8.10 修正属性存储字段）。

---

#### 3.3.2 自定义枚举（windows-rs 缺失项）

**`AUDIO_FLOW_TYPE`**：

```rust
#[repr(u32)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum AUDIO_FLOW_TYPE {
    PULL = 0,
    PUSH = 1,
}
```

**`APO_CONNECTION_BUFFER_TYPE`**：

```rust
#[repr(i32)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum APO_CONNECTION_BUFFER_TYPE {
    ALLOCATED = 0,
    EXTERNAL = 1,
    DEPENDANT = 2,
}
```

> **注意**：此枚举使用 `#[repr(i32)]`（有符号），与 C 的 `int` 底层语义一致。需编译期断言验证。

---

#### 3.3.3 类型别名

```rust
pub type REFERENCE_TIME = i64;
```

---

#### 3.3.4 自定义 POD 结构体（windows-rs 缺失项）

> `APO_REG_PROPERTIES`（1092 字节）、`APO_CONNECTION_DESCRIPTOR`（x64=40/x86=20 字节）、`APO_CONNECTION_PROPERTY`（x64=24/x86=16 字节）已由 windows-rs 0.62.2 提供并在 3.3.1 re-export，此处仅保留自定义项。

**`UNCOMPRESSED_AUDIO_FORMAT`**（36 字节，无指针，跨平台一致）：

```rust
#[repr(C)]
#[derive(Clone, Copy)]
pub struct UNCOMPRESSED_AUDIO_FORMAT {
    pub guid_format_type: GUID,
    pub dw_samples_per_frame: u32,
    pub dw_bytes_per_sample_container: u32,
    pub dw_valid_bits_per_sample: u32,
    pub f_frames_per_second: f32,
    pub dw_channel_mask: u32,
}
```

---

#### 3.3.5 签名常量

```rust
pub const APO_CONNECTION_DESCRIPTOR_SIGNATURE: u32 = u32::from_le_bytes(*b"ACDS");
pub const APO_CONNECTION_PROPERTY_SIGNATURE: u32 = u32::from_le_bytes(*b"ACPS");
pub const APO_CONNECTION_PROPERTY_V2_SIGNATURE: u32 = u32::from_le_bytes(*b"ACP2");
```

#### 3.3.6 比较标志常量

```rust
pub const AUDIOMEDIATYPE_EQUAL_FORMAT_TYPES: u32 = 0x0000_0002;
pub const AUDIOMEDIATYPE_EQUAL_FORMAT_DATA: u32 = 0x0000_0004;
pub const AUDIOMEDIATYPE_EQUAL_FORMAT_USER_DATA: u32 = 0x0000_0008;
```

#### 3.3.7 APO 专用 HRESULT 错误码

> APOERR 常量**统一定义在 `sys/com/prelude.rs`**（3.1），`apo_types` 仅做 re-export，
> 避免 HRESULT 常量分散在多个系统子模块：

```rust
pub use crate::sys::com::prelude::{
    APOERR_ALREADY_INITIALIZED,
    APOERR_ALREADY_UNLOCKED,
    APOERR_APO_LOCKED,
    APOERR_BUFFERS_OVERLAP,
    APOERR_FORMAT_NOT_SUPPORTED,
    APOERR_INVALID_APO_CLSID,
    APOERR_INVALID_COEFFCOUNT,
    APOERR_INVALID_COEFFICIENT,
    APOERR_INVALID_CONNECTION_FORMAT,
    APOERR_INVALID_CURVE_PARAM,
    APOERR_INVALID_INPUTID,
    APOERR_INVALID_OUTPUT_MAXFRAMECOUNT,
    APOERR_NOT_INITIALIZED,
    APOERR_NUM_CONNECTIONS_INVALID,
};
```

#### 3.3.8 编译期断言

使用 `const _: () = { ... };` 块，仅针对**自定义项**（windows-rs 提供的结构体大小/偏移由绑定保证，不再重复断言）：

**自定义枚举大小**：

| 断言 | 预期值 |
|------|--------|
| `size_of::<AUDIO_FLOW_TYPE>()` | 4 |
| `size_of::<APO_CONNECTION_BUFFER_TYPE>()` | 4 |

**自定义枚举 repr 语义**：

| 断言 | 预期值 |
|------|--------|
| `APO_CONNECTION_BUFFER_TYPE::ALLOCATED as i32` | 0 |
| `APO_CONNECTION_BUFFER_TYPE::EXTERNAL as i32` | 1 |
| `APO_CONNECTION_BUFFER_TYPE::DEPENDANT as i32` | 2 |

**自定义无指针结构体（跨平台一致）**：

| 结构体 | 大小 | offset 断言 |
|--------|------|------------|
| `UNCOMPRESSED_AUDIO_FORMAT` | 36 字节 | `guid_format_type`=0, `dw_samples_per_frame`=16, `dw_bytes_per_sample_container`=20, `dw_valid_bits_per_sample`=24, `f_frames_per_second`=28, `dw_channel_mask`=32 |

**禁止**：不包含业务逻辑

---

### 3.4 `sys/registry.rs`

**边界**：纯注册表操作工具层。不知道 APO。不知道音频处理。不知道安装业务。不知道配置文件。

**允许依赖**：`windows` crate、`windows-core` crate、`sys/com/prelude`（`guid_to_string`）

**禁止依赖**：`pipeline/`、`install/`、`config/`、`object/`、`utils/`、`telemetry/`

> 由原 `read.rs`、`write.rs`、`delete.rs` 合并为单一模块。GUID 格式化统一走 `sys/com/prelude::guid_to_string`（StringFromGUID2 的 unsafe 已收窄至该单一安全边界；依赖 `prelude` 的 `guid_to_string` 属允许例外）。错误类型统一为 `windows::core::Error`。权限提升函数（`make_writable`、`take_ownership`、`PrivilegeGuard`、`enable_take_ownership_privilege`、`create_administrators_sid`）迁移至 `install/permission.rs`，不属于工具层职责。

---

#### 3.4.1 私有辅助函数

| 函数 | 说明 |
|------|------|
| `win32_ok(err: WIN32_ERROR) -> Result<()>` | Win32 错误码转 `windows::core::Error`（HRESULT 格式：`0x8007_0000 | (code & 0xFFFF)`） |
| `close_key(handle: HKEY)` | RAII Drop 用，null/已关闭时静默返回 |
| `utf16_bytes_to_string(buf: &[u8]) -> String` | LE UTF-16 字节流 → String，遇 null 停止 |
| `parse_multi_sz(buf: &[u8]) -> Vec<String>` | LE UTF-16 字节流 → `Vec<String>`，双 null 结束 |
| `is_not_found(err: WIN32_ERROR) -> bool` | 判断错误码是否为 2（FILE_NOT_FOUND）或 3（PATH_NOT_FOUND） |
| `to_registry_bytes(s: &str) -> Vec<u8>` | `&str` → UTF-16 字节（含 null），用于 `RegSetValueExW` |
| `hkey_to_name(hkey: HKEY) -> Result<&'static str>` | 根键 → 名称字符串（`save_to_file` 用） |
| `dump_key_recursive(root, sub_key, display_path, content)` | 递归导出键及子键（`save_to_file` 内部） |

---

#### 3.4.2 类型定义

**`RegValue`**（注册表值的类型化表示）：

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum RegValue {
    Sz(String),           // REG_SZ 或 REG_EXPAND_SZ
    Dword(u32),           // REG_DWORD
    Qword(u64),           // REG_QWORD
    Binary(Vec<u8>),      // REG_BINARY
    MultiSz(Vec<String>), // REG_MULTI_SZ
}
```

**`RegKey`**（注册表键句柄，RAII）：

```rust
#[derive(Debug)]
pub struct RegKey {
    handle: HKEY,
}
// Drop 时调用 close_key
```

---

#### 3.4.3 `RegKey` 方法

**打开/创建**：

| 方法 | 签名 | 说明 |
|------|------|------|
| `open` | `fn open(root: HKEY, sub_key: &str) -> Result<Self>` | 以 `KEY_READ` 权限打开子键 |
| `create` | `fn create(root: HKEY, sub_key: &str) -> Result<Self>` | 创建或打开（`KEY_ALL_ACCESS`），写入/删除操作的入口 |
| `open_sub_key` | `fn open_sub_key(&self, sub_key: &str) -> Result<Self>` | 委托 `Self::open(self.handle, sub_key)` |
| `handle` | `fn handle(&self) -> HKEY` | 返回底层句柄。调用方不可自行 `RegCloseKey` |

**只读操作**：

| 方法 | 签名 | 说明 |
|------|------|------|
| `read_value` | `fn read_value(&self, name: &str) -> Result<RegValue>` | 自动识别类型读取。空字符串 `""` 读取默认值 |
| `read_sz_value` | `fn read_sz_value(&self, name: &str) -> Result<String>` | 读取 REG_SZ |
| `read_sz` | `fn read_sz(&self, name: &str) -> Option<String>` | `read_sz_value` 的 Option 包装 |
| `read_dword_value` | `fn read_dword_value(&self, name: &str) -> Result<u32>` | 读取 REG_DWORD |
| `read_binary_value` | `fn read_binary_value(&self, name: &str) -> Result<Vec<u8>>` | 读取 REG_BINARY |
| `read_multi_value` | `fn read_multi_value(&self, name: &str) -> Result<Vec<String>>` | 读取 REG_MULTI_SZ |
| `value_exists` | `fn value_exists(&self, name: &str) -> Result<bool>` | 检查当前键下指定值是否存在 |
| `key_exists_child` | `fn key_exists_child(&self, sub_key: &str) -> Result<bool>` | 检查当前键下指定子键是否存在 |
| `enum_sub_keys` | `fn enum_sub_keys(&self) -> Result<Vec<String>>` | 枚举所有子键名称。**设备枚举的底层能力源**（供 `install/device/info::enumerate_devices` 遍历 MMDevices 子键；本模块不认识设备） |
| `enum_values` | `fn enum_values(&self) -> Result<Vec<String>>` | 枚举所有值名称（含默认值 `""`） |
| `get_guid_string` | `fn get_guid_string(&self, name: &str) -> Result<String>` | 读取 GUID，支持 REG_BINARY（16 字节 LE）和 REG_SZ；统一经 `sys/com/prelude::guid_to_string` 格式化 |

**写入操作**（需要通过 `create` 获得的句柄）：

| 方法 | 签名 | 说明 |
|------|------|------|
| `write_sz` | `fn write_sz(&self, name: &str, value: &str) -> Result<()>` | 写入 REG_SZ |
| `write_dword` | `fn write_dword(&self, name: &str, value: u32) -> Result<()>` | 写入 REG_DWORD |
| `write_binary` | `fn write_binary(&self, name: &str, data: &[u8]) -> Result<()>` | 写入 REG_BINARY |

**删除操作**（需要通过 `create` 获得的句柄）：

| 方法 | 签名 | 说明 |
|------|------|------|
| `delete_value` | `fn delete_value(&self, name: &str) -> Result<()>` | 删除值（幂等） |
| `delete_sub_key` | `fn delete_sub_key(&self, name: &str) -> Result<()>` | 递归删除子键（幂等） |

---

#### 3.4.4 顶层独立函数

| 函数 | 签名 | 说明 |
|------|------|------|
| `split_key` | `fn split_key(path: &str) -> Result<(HKEY, &str)>` | 拆分路径为根键 + 子键。根键名大小写不敏感，支持 5 个标准根键及缩写（HKLM/HKCU/HKCR/HKU/HKCC） |
| `key_exists` | `fn key_exists(root: HKEY, sub_key: &str) -> Result<bool>` | 检查注册表键是否存在 |
| `value_exists` | `fn value_exists(root: HKEY, sub_key: &str, name: &str) -> Result<bool>` | 检查注册表值是否存在 |
| `delete_tree` | `fn delete_tree(root: HKEY, sub_key: &str) -> Result<()>` | 递归删除子树（幂等，不需要已打开的句柄） |
| `is_windows_version_at_least` | `fn is_windows_version_at_least(major: u32, minor: u32, build: u32) -> Result<bool>` | 读取 `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion` 检查版本 |
| `save_to_file` | `fn save_to_file(root: HKEY, sub_key: &str, path: &str) -> Result<()>` | 递归导出注册表键为 `.reg` 文件（UTF-16LE with BOM），用于安装前备份 |

**禁止**：不包含任何 APO 专用路径或安装业务逻辑

---

### 3.5 `sys/audio_defs.rs`

**职责**：Windows 音频基础定义。提供通道掩码位标志常量、标准布局常量，以及通道掩码兜底/通道名映射函数。

**引用来源**：
- `windows::Win32::Media::KernelStreaming::SPEAKER_*`（re-export 优先，缺失位自定义补齐）

**导出给**：`install/device/format.rs`（`default_channel_mask` 兜底）、`object/apo.rs`（`get_channel_names`，构造 `DspContext.channel_names` 后注入 config）

**公开 API**：

```rust
// ── 1. 通道掩码位标志（re-export 优先）──
pub use windows::Win32::Media::KernelStreaming::{
    SPEAKER_FRONT_LEFT,            // 0x1
    SPEAKER_FRONT_RIGHT,           // 0x2
    SPEAKER_FRONT_CENTER,          // 0x4
    SPEAKER_LOW_FREQUENCY,         // 0x8
    SPEAKER_BACK_LEFT,             // 0x10
    SPEAKER_BACK_RIGHT,            // 0x20
    SPEAKER_FRONT_LEFT_OF_CENTER,  // 0x40
    SPEAKER_FRONT_RIGHT_OF_CENTER, // 0x80
    SPEAKER_BACK_CENTER,           // 0x100
    SPEAKER_SIDE_LEFT,             // 0x200
    SPEAKER_SIDE_RIGHT,            // 0x400
    SPEAKER_TOP_CENTER,            // 0x800
    SPEAKER_TOP_FRONT_LEFT,        // 0x1000
    SPEAKER_TOP_FRONT_CENTER,      // 0x2000
    SPEAKER_TOP_FRONT_RIGHT,       // 0x4000
    SPEAKER_TOP_BACK_LEFT,         // 0x8000
    SPEAKER_TOP_BACK_CENTER,       // 0x10000
    SPEAKER_TOP_BACK_RIGHT,        // 0x20000
};
// windows-rs 缺失的位使用 `pub const SPEAKER_xxx: u32 = 0x...;` 自定义补齐

// ── 2. 标准布局常量 ──
// windows-rs 提供 KSAUDIO_SPEAKER_* 时 re-export，否则按位组合自定义
pub const KSAUDIO_SPEAKER_MONO:    u32 = SPEAKER_FRONT_CENTER;
pub const KSAUDIO_SPEAKER_STEREO:  u32 = SPEAKER_FRONT_LEFT | SPEAKER_FRONT_RIGHT;
pub const KSAUDIO_SPEAKER_QUAD:    u32 = SPEAKER_FRONT_LEFT | SPEAKER_FRONT_RIGHT
                                       | SPEAKER_BACK_LEFT | SPEAKER_BACK_RIGHT;
pub const KSAUDIO_SPEAKER_5POINT1: u32 = SPEAKER_FRONT_LEFT | SPEAKER_FRONT_RIGHT
                                       | SPEAKER_FRONT_CENTER | SPEAKER_LOW_FREQUENCY
                                       | SPEAKER_BACK_LEFT | SPEAKER_BACK_RIGHT;
pub const KSAUDIO_SPEAKER_7POINT1: u32 = SPEAKER_FRONT_LEFT | SPEAKER_FRONT_RIGHT
                                       | SPEAKER_FRONT_CENTER | SPEAKER_LOW_FREQUENCY
                                       | SPEAKER_BACK_LEFT | SPEAKER_BACK_RIGHT
                                       | SPEAKER_SIDE_LEFT | SPEAKER_SIDE_RIGHT;

// ── 3. 函数 ──
/// 按通道数返回标准布局掩码（兜底用）。
/// 1→MONO、2→STEREO、4→QUAD、6→5POINT1、8→7POINT1；其余返回 0。
pub fn default_channel_mask(channels: u32) -> u32;

/// 掩码 → 短名通道列表（L/R/C/LFE/BL/BR/SL/SR，按标准位顺序）。
/// 严格按掩码位映射，未命中位不补。
/// 掩码为 0 时按通道数兜底到 default_channel_mask 后重试。
pub fn get_channel_names(mask: u32) -> Vec<String>;
```

**通道名 → 掩码位映射表**：

| 短名 | 位标志 | 标准位置 |
|------|--------|---------|
| `L` | `SPEAKER_FRONT_LEFT` | 0x1 |
| `R` | `SPEAKER_FRONT_RIGHT` | 0x2 |
| `C` | `SPEAKER_FRONT_CENTER` | 0x4 |
| `LFE` | `SPEAKER_LOW_FREQUENCY` | 0x8 |
| `BL` | `SPEAKER_BACK_LEFT` | 0x10 |
| `BR` | `SPEAKER_BACK_RIGHT` | 0x20 |
| `SL` | `SPEAKER_SIDE_LEFT` | 0x200 |
| `SR` | `SPEAKER_SIDE_RIGHT` | 0x400 |

**职责边界**：只包含 Windows SDK 语义的静态定义与纯函数。不包含任何 VxAPO 业务逻辑，不引用 `pipeline/`、`config/`、`install/`、`object/`、`utils/`。

**禁止**：不包含任何业务逻辑，不感知运行时状态

---

### 3.6 `sys/known_folder.rs`

**职责**：Windows 已知文件夹路径解析（`SHGetKnownFolderPath` 安全收窄）。供 per-device 配置路径
（`C:\ProgramData\VxAPO\{GUID}\config.toml`）定位配置目录（v7.2，P0-3；v9.11 起扩展名/根路径随 driver 对齐）。

**引用来源**：
- `windows::Win32::UI::Shell::{SHGetKnownFolderPath, FOLDERID_Documents}`
- `windows::Win32::System::Com::CoTaskMemFree`（释放 `PWSTR`）
- `windows::core::{Error, PWSTR}`

**导出给**：`object/apo.rs`（Initialize 路径解析）

**边界**：只做"已知文件夹 → 字符串路径"的 FFI 收窄。**不知道** VxAPO、config.toml、
设备 GUID 拼接——拼接规则属 `object` 层（`object 7.1.8`）。不引用任何其他模块。

**公开 API**：

```rust
/// 返回文档（Documents）文件夹的绝对路径（不含结尾反斜杠）。
///
/// 封装 `SHGetKnownFolderPath(FOLDERID_Documents, KF_FLAG_DEFAULT, ...)`：
/// 1. 调用返回 `PWSTR`（CoTaskMemAlloc 分配）
/// 2. 转 UTF-16 → String
/// 3. `CoTaskMemFree` 释放（RAII 守卫，所有路径包括错误路径均释放）
///
/// # Errors
/// - `Error`：SHGetKnownFolderPath 返回非 S_OK
pub fn documents_folder() -> Result<String, windows::core::Error>;
```

**禁止**：
- 不拼接任何子路径（`\VxAPO\{GUID}\config.toml` 由 `object/apo.rs` 负责）
- 不创建目录 / 文件（I/O 属 object 层配置加载逻辑）
- 不包含业务逻辑，不感知 VxAPO
