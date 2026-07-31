## 五、`install/` 模块规范（最终版）

**边界**：不知道 `pipeline/`、`config/`。只负责设备 APO 的安装/卸载和设备查询。

**允许依赖**：`sys/`、`utils/`、`object/vx_reg_props.rs`

**禁止依赖**：`pipeline/`、`config/`

---

### 完整模块树

```
install/
├── device.rs              # 模块入口
├── device/
│   ├── endpoint.rs        # 端点状态/名称查询（只读）
│   ├── format.rs          # WAVEFORMATEX 解析 + 通道掩码兜底（只读）
│   ├── slots.rs           # 5 槽位读取 + 3 模式 + GUID 回退（只读）
│   └── info.rs            # 组合查询层（只读）
├── selector.rs            # CLI 选择界面
├── audiodg.rs             # DisableProtectedAudioDG 检查与修复
├── install.rs             # 设备级 APO 安装/卸载（写操作）
└── rollback.rs            # 事务回滚 + .reg 备份
```

---

### 引用约束总表

| 模块 | 可依赖 | 不可依赖 |
|------|--------|----------|
| `install/device/endpoint.rs` | `sys/registry`、`utils/error` | `pipeline/`、`config/` |
| `install/device/format.rs` | `sys/registry`、`utils/error`、`sys/audio_defs`（仅 `default_channel_mask`） | `config/` |
| `install/device/slots.rs` | `sys/registry`、`utils/guid` | `pipeline/`、`config/` |
| `install/device/info.rs` | `device/endpoint`、`device/format`、`device/slots`、`sys/registry`、`object/vx_reg_props`、`utils/error` | `pipeline/`、`config/` |
| `install/selector.rs` | `install/device/info` | `pipeline/`、`config/` |
| `install/audiodg.rs` | `sys/registry` | `pipeline/`、`config/` |
| `install/install.rs` | `device/slots`、`device/format`、`rollback`、`sys/registry`、`object/vx_reg_props`、`utils/error`、`utils/guid` | `pipeline/`、`config/` |
| `install/rollback.rs` | `sys/registry`、`utils/error` | `pipeline/`、`config/` |

---

### 5.1 `install/device/endpoint.rs`

**职责**：查询 Windows 音频端点的设备 ID、友好名称与连接状态。只读，不修改系统状态。

**引用来源**：
- `crate::sys::registry::RegKey`
- `crate::utils::error::VxApoError`

**导出给**：`install/device/info.rs`

**公开 API**：

```rust
/// Windows 音频端点状态（对应 IMMDevice::GetState 返回值）。
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum EndpointState {
    Active,              // DEVICE_STATE_ACTIVE = 1
    Disabled,            // DEVICE_STATE_DISABLED = 2
    NotPresent,          // DEVICE_STATE_NOTPRESENT = 4
    Unknown(u32),
}

/// 音频流方向。
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Flow { Render, Capture }

/// 音频端点信息（从注册表查询，只读）。
#[derive(Debug, Clone)]
pub struct EndpointInfo {
    pub device_id: String,         // IMMDevice 的 PKEY_DeviceInstanceId
    pub friendly_name: String,     // PKEY_DeviceInterface_FriendlyName
    pub state: EndpointState,
    pub flow: Flow,
    pub endpoint_guid: String,
}

/// 从已打开的端点注册表键查询端点信息。
///
/// 端点键位于：
/// `HKLM\...\MMDevices\Audio\Render\{guid}` 或 `...\Capture\{guid}`
///
/// 返回 Ok(Some(info)) / Ok(None)（属性缺失）/ Err（I/O 错误）。
pub fn query_endpoint(endpoint_key: &RegKey) -> Result<Option<EndpointInfo>, VxApoError>;

/// 检查端点是否为活跃状态。
pub fn is_endpoint_active(endpoint_key: &RegKey) -> Result<bool, VxApoError>;
```

---

### 5.2 `install/device/format.rs`

**职责**：从注册表二进制值解析 WAVEFORMATEX / WAVEFORMATEXTENSIBLE，提取通道数、采样率、位深和通道掩码。只读。

**引用来源**：
- `crate::sys::registry::RegKey`
- `crate::utils::error::VxApoError`
- `crate::sys::audio_defs::default_channel_mask`（兜底）

**导出给**：`install/device/info.rs`

**通道掩码兜底链**（Note 27）：
1. WAVEFORMATEXTENSIBLE 的 `dwChannelMask`（bytes[20..24]）
2. 注册表 `channelMaskValueName`（DWORD）
3. `sys::audio_defs::default_channel_mask`（标准布局映射，1/2/4/6/8 通道）

**公开 API**：

```rust
/// 解析后的音频格式信息。
#[derive(Debug, Clone)]
pub struct AudioFormat {
    pub format_tag: u16,          // WAVE_FORMAT_PCM = 1, EXTENSIBLE = 0xFFFE
    pub channels: u16,
    pub sample_rate: u32,
    pub bits_per_sample: u16,
    pub channel_mask: u32,        // 经兜底链解析后保证非零
    pub is_extensible: bool,
}

/// 从原始字节解析音频格式（纯函数，无 I/O，可独立测试）。
///
/// `channel_mask_override`：外部兜底通道掩码（来自注册表 DWORD 值）。
pub fn parse_audio_format(bytes: &[u8], channel_mask_override: Option<u32>) -> Option<AudioFormat>;

/// 按优先级解析通道掩码（兜底链）。
pub fn resolve_channel_mask(
    extensible_mask: u32,
    registry_mask: Option<u32>,
    channels: u16,
) -> u32;

/// 从注册表键读取并解析音频格式。
///
/// 尝试读取 `format_value_name` 二进制值，解析后返回。
/// 值不存在返回 Ok(None)。
pub fn read_audio_format(
    key: &RegKey,
    format_value_name: &str,
    channel_mask_value_name: Option<&str>,
) -> Result<Option<AudioFormat>, VxApoError>;
```

---

### 5.3 `install/device/slots.rs`

**职责**：管理 FxProperties 下 5 个 APO GUID 槽位，提供安装模式定义与原始 APO GUID 回退查询。只读。

**引用来源**：
- `crate::sys::registry::RegKey`
- `crate::utils::guid::{format_guid, parse_guid_from_bytes}`

**导出给**：`install/device/info.rs`、`install/install.rs`

**5 个槽位**（Note 25）：

| 索引 | 名称 | 角色 |
|------|------|------|
| 0 | LFX | Legacy PreMix（Win8.1+） |
| 1 | GFX | Legacy PostMix（Win8.1+） |
| 2 | SFX | Side-effect PreMix（Win10+） |
| 3 | MFX | Mixed-effect PostMix（Win11 蓝牙） |
| 4 | EFX | Endpoint-effect PostMix（默认） |

**3 种安装模式**（Note 26）：

| 模式 | PreMix 槽位 | PostMix 槽位 | 适用场景 |
|------|------------|-------------|---------|
| LfxGfx | LFX(0) | GFX(1) | Win8.1+ Legacy |
| SfxMfx | SFX(2) | MFX(3) | Win11 蓝牙 |
| SfxEfx | SFX(2) | EFX(4) | **默认** |

**公开 API**：

```rust
pub const FX_PROPERTIES_KEY: &str = "FxProperties";
pub const INSTALL_VERSION: &str = "2";
pub const INSTALL_VERSION_LEGACY: &str = "1";

/// APO 槽位索引。
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
#[repr(u8)]
pub enum ApoSlot { Lfx = 0, Gfx = 1, Sfx = 2, Mfx = 3, Efx = 4 }

impl ApoSlot {
    pub const ALL: [ApoSlot; 5];
    pub fn index(self) -> u8;
    /// 注册表值名称：`{d04e05a6-...},{index}`
    pub fn value_name(self) -> String;
    pub fn is_premix(self) -> bool;
    pub fn is_postmix(self) -> bool;
}

/// 安装模式。
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum InstallMode { LfxGfx, SfxMfx, SfxEfx }

impl InstallMode {
    pub fn premix_slot(self) -> ApoSlot;
    pub fn postmix_slot(self) -> ApoSlot;
    pub fn default_mode() -> InstallMode;  // SfxEfx
}

/// 槽位值状态（三态）。
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SlotValue {
    NoKey,                       // FxProperties 键不存在
    NoValue,                     // 值为空或被其他 APO 占据
    Guid(windows::core::GUID),   // 具体的 APO CLSID
}

impl SlotValue {
    pub fn is_guid(&self) -> bool;
    pub fn is_empty(&self) -> bool;
    pub fn as_guid(&self) -> Option<windows::core::GUID>;
}

/// 从 FxProperties 读取指定槽位。
pub fn read_slot_value(fx_key: &RegKey, slot: ApoSlot) -> SlotValue;

/// 从端点根键读取所有 5 个槽位。
/// FxProperties 不存在时所有槽位返回 NoKey。
pub fn read_all_slots(endpoint_key: &RegKey) -> [SlotValue; 5];

/// 获取原始 PreMix APO GUID（带回退，Note 46）。
///
/// 回退规则：主槽位 NoValue 时回退到另一模式的 PreMix 槽位。
/// NoKey 或无回退时返回空字符串。
pub fn get_original_pre_mix(slots: &[SlotValue; 5], mode: InstallMode) -> String;

/// 获取原始 PostMix APO GUID（带回退，Note 46）。
///
/// 回退规则：主槽位 NoValue 时按模式优先级尝试其他 PostMix 槽位：
/// - SfxEfx：EFX → MFX → GFX
/// - SfxMfx：MFX → EFX → GFX
/// - LfxGfx：GFX → EFX → MFX
pub fn get_original_post_mix(slots: &[SlotValue; 5], mode: InstallMode) -> String;
```

---

### 5.4 `install/device/info.rs`

**职责**：组合 `endpoint`、`slots`、`format` 三个子模块，提供高层查询接口。只读。

**引用来源**：
- `install/device/endpoint`、`install/device/slots`、`install/device/format`
- `crate::sys::registry::RegKey`
- `crate::object::vx_reg_props::{CLSID_VXAPO_PRE_MIX, CLSID_VXAPO_POST_MIX}`
- `crate::utils::error::VxApoError`

**导出给**：`install/selector.rs`、`object/apo.rs`

**公开 API**：

```rust
/// 设备综合信息（查询结果，只读）。
#[derive(Debug, Clone)]
pub struct DeviceInfo {
    pub endpoint: EndpointInfo,
    pub install_mode: InstallMode,
    pub slots: [SlotValue; 5],
    pub format: Option<AudioFormat>,
    pub installed_version: String,     // "2"、"1" 或 ""（未安装）
}

impl DeviceInfo {
    /// VxAPO 是否已安装。
    /// 判定：版本非空，且 PreMix 或 PostMix 槽位含 VxAPO CLSID。
    pub fn is_installed(&self) -> bool;

    /// 是否可升级（已安装且版本低于 INSTALL_VERSION）。
    pub fn can_be_upgraded(&self) -> bool;

    /// 是否为实验性安装（LfxGfx 模式）。
    pub fn is_experimental(&self) -> bool;

    /// 音频增强是否被禁用。
    pub fn is_enhancements_disabled(&self, endpoint_key: &RegKey) -> bool;

    /// 是否有未应用的更改。
    pub fn has_changes(&self) -> bool;
}

/// 查询设备综合信息（组合 endpoint + slots + format）。
pub fn query_device_info(endpoint_key: &RegKey) -> Result<Option<DeviceInfo>, VxApoError>;

/// 设备枚举（遍历 MMDevices\Audio\Render 和 Capture 下所有端点）。
pub fn enumerate_devices() -> Result<Vec<DeviceInfo>, VxApoError>;
```

**模式检测逻辑**：
1. SFX 有 GUID + MFX 有 GUID → SfxMfx
2. SFX 有 GUID + MFX 无 GUID → SfxEfx
3. LFX 有 GUID + SFX 无 GUID → LfxGfx
4. 都没有 → SfxEfx（默认）

---

### 5.5 `install/selector.rs`

**职责**：CLI 选择界面。枚举设备、列出名称、让用户选择，调用 `install.rs` 执行操作。

**引用来源**：
- `install/device/info::DeviceInfo`
- `install/device/info::enumerate_devices`
- `install/install::{install_endpoint, uninstall_endpoint, InstallConfig}`

**导出给**：外部 CLI

**公开 API**：

```rust
pub fn list_devices() -> Result<Vec<DeviceInfo>>;
pub fn select_device() -> Result<Option<DeviceInfo>>;
pub fn run_install_flow() -> Result<()>;
pub fn run_uninstall_flow() -> Result<()>;
pub fn print_device_list(devices: &[DeviceInfo]);
pub fn prompt_user(devices: &[DeviceInfo]) -> Result<usize>;
```

**禁止**：不直接操作注册表（通过 `install.rs`）

---

### 5.6 `install/audiodg.rs`

**职责**：DisableProtectedAudioDG 检查与修复。

**引用来源**：
- `crate::sys::registry::{RegKey, RegValue}`

**导出给**：`object/apo.rs`

**公开 API**：

```rust
/// 检查保护是否已禁用。
///
/// 返回 true：DisableProtectedAudioDG 值存在且 == 1（允许第三方 APO 加载）。
/// 返回 false：值不存在或 != 1（Windows 阻止第三方 APO 加载）。
pub fn is_disabled() -> Result<bool>;

/// 检查是否允许第三方 APO 加载。
///
/// is_disabled() 的语义别名——返回 true 表示可以加载。
pub fn is_third_party_allowed() -> Result<bool>;

/// 设置 DisableProtectedAudioDG = 1（禁用保护，允许第三方加载）。
///
/// 需要管理员权限（写入 HKLM）。
pub fn disable() -> Result<()>;

/// 删除 DisableProtectedAudioDG 值（恢复 Windows 默认保护行为）。
///
/// 值不存在不算错误。
pub fn restore() -> Result<()>;

/// 检查并确保允许加载，不允许时尝试修复。
///
/// 由 object/apo.rs LockForProcess 调用。
pub fn ensure_can_load() -> Result<()>;
```

**注册表路径**：`HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Audio`

**值名**：`DisableProtectedAudioDG`（`REG_DWORD`）

**语义**：值不存在或 != 1 → 阻止；值存在且 == 1 → 允许。

**实现要点**：
- `disable()` 使用 `RegKey::create()` 获取可写句柄，`write_dword()` 写入 1，`handle()` 获取底层 HKEY
- `restore()` 使用 `RegKey::open()` 获取句柄，`handle()` 获取底层 HKEY，`delete_value()` 删除值（值不存在静默返回）
- 不使用 unsafe 指针转换（pre 的 `RegKey::handle()` 已暴露底层句柄）

---

### 5.7 `install/rollback.rs`

**职责**：安装事务管理与 .reg 备份。RAII Drop 保证安装失败时自动回滚。

**引用来源**：
- `crate::sys::registry::{read, write}`
- `crate::utils::error::Result`

**导出给**：`install/install.rs`

**公开 API**：

```rust
/// 安装操作的回滚项。
#[derive(Debug)]
pub enum RollbackAction {
    DeleteKey(String),
    RestoreValue { root: HKEY, key: String, name: String, backup: Vec<u8> },
    DeleteClsidKey(String),
}

/// 分层事务管理器。
///
/// 安装过程中记录所有操作。失败时 Drop 自动按逆序回滚。
/// 成功时调用 `commit()` 禁用回滚。
#[derive(Debug)]
pub struct Transaction;

impl Transaction {
    pub fn new() -> Self;
    pub fn record(&mut self, action: RollbackAction);
    pub fn commit(&mut self);
    pub fn rollback(&mut self);
    pub fn action_count(&self) -> usize;
    pub fn is_committed(&self) -> bool;
}
// Drop 时未 commit 则自动 rollback

/// 备份设备 FxProperties 到 .reg 文件（Note 32）。
/// 文件名格式：`backup_{设备名}_{连接名}.reg`
pub fn backup_fx_properties(
    device_name: &str,
    connection_name: &str,
    fx_properties_path: &str,
    output_dir: &str,
) -> Result<String>;
```

---

### 5.8 `install/install.rs`

**职责**：设备级 APO 安装与卸载。实现 Note 47 定义的完整 7 步安装流程。

**引用来源**：
- `install/device/slots::*`
- `install/device/format::*`
- `install/rollback::{Transaction, RollbackAction}`
- `crate::sys::registry::{read, write}`
- `crate::object::vx_reg_props::{CLSID_VXAPO_PRE_MIX, CLSID_VXAPO_POST_MIX}`
- `crate::utils::error::*`
- `crate::utils::guid::*`

**导出给**：`install/selector.rs`

---

#### InstallConfig

```rust
/// 安装参数。
pub struct InstallConfig {
    /// 是否安装 PreMix APO。
    pub install_premix: bool,
    /// 是否安装 PostMix APO。
    pub install_postmix: bool,
    /// 安装模式（决定使用哪两个槽位）。
    pub install_mode: InstallMode,
    /// 是否保留原有 PreMix APO 作为子 APO。
    /// true 时读取当前 PreMix 槽位 GUID 写入 childGuid。
    pub use_original_apo_premix: bool,
    /// 是否保留原有 PostMix APO 作为子 APO。
    pub use_original_apo_postmix: bool,
    /// 是否允许静音缓冲区快速路径（Note 11）。
    pub allow_silent_buffer: bool,
}

impl InstallConfig {
    pub fn default_config() -> Self;  // SfxEfx, 双向安装, allow_silent = true
}
```

---

#### 7 步安装流程（Note 47）

```rust
/// 安装 VxAPO 到指定音频端点。
///
/// Note 47 完整流程：
/// 1. 创建 Child APOs 键
/// 2. FxProperties 不存在则创建（失败则权限提升重试，Note 31）
/// 3. 已存在则备份原始 GUID 到 .reg（Note 32）+ 记录回滚
/// 4. 写入子 APO 配置（childGuid / allowSilentBuffer / autoAdjust / version）
/// 5. 按模式写入 APO GUID（删除非当前模式的旧槽位）
/// 6. 写入默认处理模式 GUID（AUDIO_SIGNALPROCESSINGMODE_DEFAULT）
/// 7. 删除 DisableEnhancements
///
/// 整个安装在 Transaction 保护下执行，任何步骤失败时自动逆序回滚。
pub fn install_endpoint(
    device_guid: &str,
    device_name: &str,
    connection_name: &str,
    config: &InstallConfig,
) -> Result<()>;
```

**注册表路径**：
- `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\MMDevices\Audio\Render\{device}\FxProperties`
- `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\MMDevices\Audio\Capture\{device}\FxProperties`

**安装写入**（与 pre 规范一致）：
- `LfxGfx`：PreMix → LFX(0)，PostMix → GFX(1)，删除 SFX/MFX/EFX
- `SfxMfx`：PreMix → SFX(2)，PostMix → MFX(3)，删除 LFX/GFX/EFX
- `SfxEfx`：PreMix → SFX(2)，PostMix → EFX(4)，删除 LFX/GFX/MFX

---

#### 卸载流程

```rust
/// 从指定音频端点卸载 VxAPO。
///
/// 1. 定位端点 FxProperties
/// 2. 读取当前槽位，删除 VxAPO 的 CLSID
/// 3. 删除子 APO 配置值（childGuid / allowSilentBuffer / autoAdjust / version）
/// 4. 删除 DisableEnhancements
pub fn uninstall_endpoint(device_guid: &str) -> Result<()>;
```

**禁止**：不知道 `pipeline/` 的存在