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
│   ├── sysfx.rs           # CAPX MSFX 模板定位/接管/恢复（v9.0）
│   ├── identity.rs        # 端点稳定身份（实例 ID / 硬件 ID / 产品名 / 端点历史）
│   ├── stale.rs           # 旧 GUID 残留检测/迁移/清理（分层身份匹配）
│   └── info.rs            # 组合查询层 + 设备枚举唯一入口（只读）
├── selector.rs            # 模块入口（pub mod operation;）
├── audiodg.rs             # DisableProtectedAudioDG 检查与修复
└── selector/
    └── operation.rs       # 安装 + 卸载 + 回滚（install_endpoint / uninstall_endpoint / Transaction）
```

---

### 引用约束总表

> v9.17（单一事实源）：本表不再独立维护——以主规范
> `模块引用规范（无详细模块版）.md` 第十一节「引用约束总表」为唯一基线，
> install 各文件的允许/禁止依赖逐行见主规范。

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

**职责**：管理 FxProperties 下 5 个 APO GUID 槽位，提供安装模式定义与原始 APO GUID 回退查询，以及 **VxAPO 独立安装信息区（子 APO 备份）读取**。只读。

**引用来源**：
- `crate::sys::registry::RegKey`
- `crate::utils::guid::{guid_from_bytes, is_zero_guid, parse_guid_string}`
- `crate::sys::com::prelude::guid_to_string`（GUID 标准字符串格式化，安全收窄边界）

**导出给**：`install/device/info.rs`、`install/selector/operation.rs`、`object/apo.rs`（v8.4：运行期子 APO GUID 读取——端点 GUID → `HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}\{PreMixChild|PostMixChild}`）

**子 APO 安装信息区（v8.4，VxAPO 独立路径）**：

```
HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}
    PreMixChild                     = {前任 PreMix APO GUID}（useOriginalAPOPreMix 时写）
    PostMixChild                    = {前任 PostMix APO GUID}（useOriginalAPOPostMix 时写）
    AllowSilentBufferModification   = "true"/"false"
    DisableAutomaticAdjustment      = "true"（auto_adjust=false 时写）
    Version                         = "2"
```

> **路径隔离（v8.4，指示）**：**禁止**读写 EAPO 的 `HKLM\SOFTWARE\EqualizerAPO\Child APOs`
> （EAPO `childApoPath`，RegistryHelper.h 33 + DeviceAPOInfo.cpp 43）——VxAPO 用独立的
> `HKLM\SOFTWARE\VxAPO` 根。写 EAPO 路径会污染其安装信息区（EAPO 读到 VxAPO 写的
> PreMixChild/PostMixChild，重装/卸载 EAPO 时被覆盖或语义混淆）。
> 对齐的是 EAPO 的**机制**（独立注册表路径存子 APO 备份），非路径本身。

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
    /// Windows 真实注册表槽位属性 ID（PID）——**v8.9 实证（2026-08-04 reg query）**：
    /// `{d04e05a6-...}` 各槽位 PID 为 **0/3/5/6/7**（非连续 0-4）：
    /// LFX=0 / GFX=3 / SFX=5 / MFX=6 / EFX=7（与旧 CLI src/reg.rs 常量一致）。
    pub fn registry_pid(self) -> u8;
    /// 注册表值名称：`{d04e05a6-...},{registry_pid}`
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

/// 从 FxProperties 读取指定槽位（**v8.9 双格式兼容 + 全零归一**）。
///
/// Windows 槽位值同时存在 REG_SZ（EAPO 等第三方写 GUID 字符串，实证）与
/// REG_BINARY（16 字节 LE，部分实现/旧 VxAPO 写）——按真实类型自动识别。
/// **全零 GUID（`{00000000-...}`）是 Windows 的「无 APO」占位，归一为 NoValue**
/// （否则自动探测会把全零槽位误判为已占用）。
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

/// **EAPO 三档安装模式自动探测**（v8.9，执行端落地——EAPO load() 396-413 C41-C44 移植）。
///
/// 纯逻辑 API——调用方（driver info 层 / APP）负责准备输入，driver 只做判定：
///
/// | 优先级 | 条件 | 模式 |
/// |--------|------|------|
/// | 0 | Win < 8.1（不探测） | LfxGfx（Legacy 初始默认） |
/// | 1 | Win8.1+ 且 FxProperties **只有 LFX/GFX 值、SFX/MFX/EFX 全空** | LfxGfx（驱动仅支持 Legacy） |
/// | 2 | 端点 Properties 子键存在蓝牙容器 ID（`{b3f8fa53-...},41`） | SfxMfx（Win11 蓝牙组合，EFX 无效） |
/// | 3 | 否则（现代驱动默认） | SfxEfx |
///
/// # 参数
///
/// - `is_windows_8_1_or_newer`：OS 版本判定（`registry::is_windows_version_at_least(6,3,9600)`）。
/// - `slots`：5 槽位值（来自端点 FxProperties，`read_all_slots`）。
/// - `has_bluetooth_container`：端点 `Properties` 子键下
///   `{b3f8fa53-0004-438e-9003-51a46e139bfc},41`（PKEY_Device_ContainerId，PID 41）值存在。
pub fn detect_install_mode(
    is_windows_8_1_or_newer: bool,
    slots: &[SlotValue; 5],
    has_bluetooth_container: bool,
) -> InstallMode;
```

**安装信息区存在性检测（v8.5，全量判定依据）**：

```rust
/// VxAPO 独立安装信息区键路径（v8.5）。
pub const CHILD_APO_PATH_ROOT: &str = r"HKLM\SOFTWARE\VxAPO\Child APOs";

/// 判断某设备是否有 VxAPO 安装信息区（全量/非全量判定的唯一依据，intent 七节 v8.5）。
///
/// - 不存在 → 初始安装 / 完全卸载后安装 → `install_endpoint` 走**全量备份路径**；
/// - 存在 → 重装 / 失守重装 → 走**非全量路径**（槽位覆盖或保留旧 childapo）。
///
/// 私有路径保证：只由 install_endpoint 写、uninstall_endpoint 删（卸载必删整个键）；
/// 第三方 APO 软件不会写它（各软件只操作自己的私有路径）——存在性即充分判定。
pub fn child_apo_key_exists(device_guid: &str) -> bool;
```

---

### 5.4 `install/device/info.rs`

**职责**：组合 `endpoint`、`slots`、`format` 三个子模块，提供高层查询接口。**设备枚举唯一入口**（`enumerate_devices`）。只读。

**引用来源**：
- `install/device/endpoint`、`install/device/slots`、`install/device/format`
- `crate::sys::registry::RegKey`
- `crate::object::vx_reg_props::{CLSID_VXAPO_PRE_MIX, CLSID_VXAPO_POST_MIX}`
- `crate::utils::error::VxApoError`

**导出给**：`install/selector/operation.rs`、`object/apo.rs`、CLI（`commands.rs`）

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

    /// 设备是否已禁用（DEVICE_STATE_DISABLED，E3.2/EAPO 借鉴）。
    pub fn is_disabled(&self) -> bool;

    /// 设备是否已拔除（DEVICE_STATE_NOTPRESENT，E3.2/EAPO 借鉴）。
    pub fn is_unplugged(&self) -> bool;

    /// 是否有未应用的更改。
    pub fn has_changes(&self) -> bool;
}

/// 查询设备综合信息（组合 endpoint + slots + format）。
pub fn query_device_info(endpoint_key: &RegKey) -> Result<Option<DeviceInfo>, VxApoError>;

/// 设备枚举——**返回所有合法安装容器**（遍历 MMDevices\Audio\Render 和 Capture
/// 下所有端点键，组合 endpoint/slots/format）。只保证"容器合法"，不判定
/// 默认设备（用户层 COM 职责，见上）。
pub fn enumerate_devices() -> Result<Vec<DeviceInfo>, VxApoError>;
```

> **设备物理状态谓词（E3.2）**：`is_disabled()`/`is_unplugged()` 由
> `DeviceInfo.endpoint.state`（已有 `EndpointState::{Disabled, NotPresent}`）推导，
> 零新增 I/O。用于合法安装容器的进一步筛选。

> **默认设备判定边界（E3.1 澄清）**：driver 层**不判定**"是否默认设备"——
> `enumerate_devices()` 仅返回注册表 MMDevices 下的**合法安装容器**。
> 默认设备/有效设备展示是**用户层（CLI/APP）经 COM `IMMDeviceEnumerator::GetDefaultAudioEndpoint`**
> 取得的有效设备，再以 `device_id`/`endpoint_guid` 与 driver 枚举结果**对应**。
> driver 不引入 COM 设备枚举，保持注册表单栈。

**模式检测逻辑（v8.9 修正——只认 VxAPO CLSID 成对，2026-08-04 实证）**：
1. LFX=VxAPO_PRE **且** GFX=VxAPO_POST → LfxGfx（**非 VxAPO 占槽不算**——EDIFIER 实证：
   旧实现按「任意 GUID 占槽」误判 SfxMfx，实际 SFX/EFX 被 EAPO 占、MFX 被系统占，
   正确表示「未安装 VxAPO」）
2. SFX=VxAPO_PRE **且** MFX=VxAPO_POST → SfxMfx
3. SFX=VxAPO_PRE **且** EFX=VxAPO_POST → SfxEfx
4. 都不成对 → `default_mode()`（SfxEfx，表示未安装 VxAPO）

**EAPO 三档自动探测（v8.9，CLI 缺省 `--mode` / APP 调用入口）**：

```rust
/// 蓝牙组合设备容器 ID 值名（PKEY_Device_ContainerId，WT_DEVICE PID 41）。
/// EAPO DeviceAPOInfo.cpp 51/410-411 实证——端点 `Properties` 子键下存在此值
/// 即 Win11 蓝牙组合设备（EFX 无效），SfxMfx 模式探测判据。
pub const BLUETOOTH_CONTAINER_VALUE: &str
    = "{b3f8fa53-0004-438e-9003-51a46e139bfc},41";

/// 组合 OS 版本 + 5 槽位 + 蓝牙容器 ID 交给 slots::detect_install_mode（自动探测）。
pub fn detect_mode_for_device(endpoint_key: &RegKey) -> InstallMode;

/// 按端点 GUID 自动探测（Render 优先 / Capture 兜底定位端点根键）。
/// 定位失败回退 `detect_install_mode(true, 全空, false)` → SfxEfx。
pub fn detect_mode_for_guid(device_guid: &str) -> InstallMode;
```

---

### 5.5 `install/selector.rs`（模块入口）

**职责**：`selector` 子模块的入口聚合——当前仅 `pub mod operation;` 声明，交互选择流程已由 CLI 自持（v9.19 删除 `select` 死代码）。

**公开 API**：

```rust
pub mod operation;
```

---

### 5.5.1 ~~`install/selector/select.rs`~~（已删除）

> **v9.19 起不再存在**：设备选择交互已上移到 CLI/App，driver 不再包含 `select.rs`。
> 设备枚举仍统一走 `install/device/info::enumerate_devices`。

---

### 5.5.2 `install/selector/operation.rs`

**职责**：设备 APO 安装/卸载执行 + 事务回滚（原 `install.rs` + `rollback.rs` 合并至此）。

**引用来源**：
- `install/device/slots::*`（ApoSlot / InstallMode / SlotValue / read_all_slots / FX_PROPERTIES_KEY / INSTALL_VERSION）
- `install/device/format::*`
- `crate::sys::registry::{RegKey, RegValue}`（备份经 `save_to_file` 委托）
- `crate::object::vx_reg_props::{CLSID_VXAPO_PRE_MIX, CLSID_VXAPO_POST_MIX}`
- `crate::object::dll_exports::register_apo_with_path`（安装前刷新全局 APO 注册，补全 AudioEngine 键）
- `crate::sys::com::prelude::guid_to_string`（GUID 格式化）
- `crate::utils::vx_error::{Result, VxApoError}`

**导出给**：CLI（`vxapo-cli/src/commands.rs`）、`object/apo.rs`（按需）

**公开 API**：

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
    pub use_original_apo_premix: bool,
    /// 是否保留原有 PostMix APO 作为子 APO。
    pub use_original_apo_postmix: bool,
    /// 是否允许静音缓冲区快速路径（Note 11）。
    pub allow_silent_buffer: bool,
    /// 是否启用自动频响校正（E3.3/EAPO `InstallState::autoAdjust` 借鉴）；
    /// 独立于 `allow_silent_buffer`，对应安装 Step 4 写注册表的 `autoAdjust`。
    pub auto_adjust: bool,
}

impl InstallConfig {
    pub fn default_config() -> Self;  // SfxEfx, 双向安装, allow_silent = true, auto_adjust = false
}

pub fn install_endpoint(
    device_guid: &str,
    device_name: &str,
    connection_name: &str,
    config: &InstallConfig,
    verify: bool,   // E3.4（v8.3）：true 时安装末尾激活 IAudioClient 做音频管线自检
) -> Result<()>;

pub fn uninstall_endpoint(device_guid: &str) -> Result<()>;
```

> **`auto_adjust`（E3.3）**：`InstallConfig` 独立字段，Step 4 写入注册表的 `autoAdjust`
> 读取该字段而非硬编码。默认 `false`（EAPO 默认 true，但 VxAPO 无自动校正实现，保守默认关）。

> **安装自检（E3.4，EAPO `testAPOInstallation` 借鉴；v8.3 修正描述）**：`install_endpoint(..., verify)`——
> `verify=true` 时，7 步全部 commit 后执行**音频管线自检**。**EAPO 源码事实**（DeviceAPOInfo.cpp 777-815）：
> `testAPOInstallation` 实际是 `IMMDeviceEnumerator::GetDevice` → `IAudioClient::Activate` →
> `GetMixFormat` → `Initialize(共享模式, 100ms)`——**激活 IAudioClient 验证端点可打开音频管线**，
> **非**「CoCreateInstance 验证 DLL 可实例化」（v8.3 修正：此前描述与源码不符，见 `Equalizer 行为文档.md` S2）。
> VxAPO 对齐：`verify=true` 时按设备激活 IAudioClient 做管线自检（对齐 EAPO）。自检失败：
> - **不自动回滚**（注册表已写入且端点可能瞬时不可用；EAPO 同策略抛异常报告）
> - EAPO 语义：`fail()` 抛 `DeviceException`（DeviceAPOInfo.cpp 817-822）——VxAPO 对齐为返回 `Err` 附明确错误，由 CLI/调用方提示
> - 用户可选择忽略或回滚（`reinstall`/`uninstall` 显式操作）

**Note 47 安装流程**（7 步，v8.5 补全量/非全量判定；v8.9 实现对齐——最小写权限/独立信息区/self-preserve/REG_SZ/EAPO 对齐）：

**0. 全量判定（v8.5，产品意图「全量备份发生条件判定」）**：`install_endpoint` 开头检查
`HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}` 键是否存在（`install/device/slots` 提供存在性检测）：
- **不存在 → 全量备份路径**（初始安装 / 完全卸载后安装——卸载已删键故等价）：5 槽位完整快照
  （含 NOKEY/NOVALUE 占位）+ 按 useOriginal 写 PreMix/PostMixChild + AllowSilent/DisableAutoAdjust/Version；
- **存在 → 非全量路径**（重装/失守重装）：失守时（应用层检测到所需安装槽位非 VxAPO CLSID）仅**覆盖**
  `PreMixChild`/`PostMixChild` 为当前被夺占槽位值（最新前任）；无失守保留旧 childapo；
  并**全量重新快照**维持回退基线最新。
- **v8.9 补充：被覆盖槽位名/原值无条件备份**——无论全量/非全量，
  覆盖 PreMix/PostMix 槽位前把「槽位名 + 原值」备份到信息区
  （`PreMixSlot`/`PostMixSlot` + `PreMixSlotValue`/`PostMixSlotValue`）；
  uninstall 按此恢复被覆盖的第三方 APO（EAPO 等），否则快照恢复后槽位变 NoValue
  （2026-08-04 实测：VxAPO 把 EAPO 弄成子 APO 后另一软件又覆盖父槽位 → 卸载需恢复 EAPO）。

**0a. 前置（v8.10，driver 全流程收口）**：
- `DisableProtectedAudioDG=1`（`install/audiodg::disable`）——audiodg 创建 APO 实例前检查，缺值拒载；
- `refresh_global_registration()`——读已有 CLSID→DLL 绑定并调 `object/dll_exports::register_apo_with_path`
  补全/覆盖 `AudioEngine\AudioProcessingObjects` 完整字段（旧注册缺 MaxInstances 等导致父槽位不加载）。

1. 创建 **VxAPO 独立安装信息区** `HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}` 键
   （v8.4 路径隔离修正：**对齐 EAPO `HKLM\SOFTWARE\EqualizerAPO\Child APOs`（childApoPath，
   RegistryHelper.h 33 + DeviceAPOInfo.cpp 43）的机制，但用 VxAPO 自己的 SOFTWARE\VxAPO 根**；
   **禁止写入 EAPO 安装信息区**——污染 EAPO 会使其读到 VxAPO 写的 PreMixChild/PostMixChild，
   重装/卸载 EAPO 时被覆盖或语义混淆）
2. FxProperties 打开/创建（v8.9 最小写权限）：
   - **已存在 → `RegKey::open_for_write`（KEY_SET_VALUE|KEY_QUERY_VALUE）**——MMDevices 端点键
     ACL 只给管理员 SetValue/ReadKey（无 CreateSubKey），请求 KEY_ALL_ACCESS 被拒（0x80070005）；
     is_new 判定读 `version` 值存在性
   - 不存在 → `RegKey::create`（SAM_ALL，新建键可全权限）并事务记录删除回滚
3. 已存在则备份 FxProperties 至 `.reg`（`sys::registry::save_to_file`，best-effort）+ 事务记录槽位回滚
4. 写入子 APO 配置到 **`HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}`**：
   `PreMixChild` / `PostMixChild`（**v8.9 self-preserve 过滤：槽位值 == VxAPO 自身 CLSID → 不保留
   （否则重装把 VxAPO 自己当子 APO，快照 diff 实测 childPreMix=41C34613 自占）**；
   其余按 useOriginalAPOPreMix/PostMix 决定是否写，对齐 EAPO DeviceAPOInfo.cpp 558-563；
   非全量路径失守时覆盖为当前被夺占槽位值）/ `PreMixSlot`+`PreMixSlotValue` 等备份值
   （**无条件写**——uninstall 恢复槽位必须用它把第三方 APO 写回；child GUID 只在保留时写，
   原值备份始终写）/ `AllowSilentBufferModification` / `DisableAutomaticAdjustment`（autoAdjust）/ `Version`
5. 按模式写入 APO GUID（v8.9 EAPO 互斥语义 + REG_SZ 强制）：
   - **槽位写 REG_SZ（GUID 字符串）**——EAPO 生态（RegistryHelper.h）与 Windows 音频枚举器读
     槽位期望 REG_SZ；写 REG_BINARY 报「wrong type」导致 EAPO 无法枚举设备（2026-08-04 实证）
   - **删除非当前模式旧槽位，保留 EAPO「不动槽位」**：LfxGfx 删 SFX/MFX/EFX（Legacy 独占）；
     SfxMfx 删 LFX/GFX **保 EFX**；SfxEfx 删 LFX/GFX **保 MFX**（旧实现一律删非当前模式槽位 →
     SfxEfx 误删 MFX，蓝牙场景 MFX 与 EFX 冲突）
6. 写入默认处理模式 GUID（`KSDATAFORMAT_SUBTYPE_DEFAULT_PROCESSMODE` =
   AUDIO_SIGNALPROCESSINGMODE_DEFAULT，REG_SZ）
7. 删除 DisableEnhancements
   - **v9.0 对齐 EAPO**：同时删除 `{1da5d803-d492-4edd-8c23-e0c0ffee7f0e},5`
     （PKEY_AudioEndpoint_Disable_SysFx），强制启用增强链
   - **capture 特例（v8.9，EAPO DeviceAPOInfo.cpp 583/607/632 对齐）**：采集端点（路径含
     Capture）只装 PreMix、不装 PostMix（VxAPO 不做采集端增强）；`PreMixChild` 备份保留、
     PostMixChild 强制 None
8. **接管 CAPX「设备默认效果」**（v9.0，`install/device/sysfx.rs`）：
   - 按设备实例 ID + JackSubType 定位 `HKLM\SYSTEM\...\DeviceClasses\...\Device Parameters\MSFX\N`
   - 微软 StreamEffectClsid（`,5`）→ VxAPO PreMix；删除 ModeEffectClsid（`,6`，避免微软 MFX 与 VxAPO PostMix 重复处理）
   - **运行期自愈（v9.4）**：安装时接管只覆盖当时存在的模板；Windows 重新枚举/
     重启后可能把微软 CAPX 重新灌回 `MSFX\N`。DLL `Initialize` 时对已装 VxAPO 的
     端点再次调用 `ensure_takeover_for_endpoint`——只动微软 CLSID、不覆盖第三方、
     幂等（无微软条目零写入）、失败仅降级日志；同时删除 DisableEnhancements /
     Disable_SysFx 强制启用增强链。
   - 端点 FxProperties 残留的微软 `,6` 同样删除；原始值写入 VxAPO 安装信息区（`SysFxBackups`）
   - 卸载时恢复微软默认效果（优先用备份，无备份按 CAPX 默认值兜底）
9. **重启 AudioSrv**（`install/audiodg::restart_audio_service`，EAPO 安装对齐）——
   使新槽位拓扑/注册生效；CLI 不再重复重启

整个安装在 `Transaction`（Drop 时逆序回滚）保护下执行，任何步骤失败时自动回滚。

**卸载流程**（v8.5 补删键；v8.9 实现对齐——只删 VxAPO CLSID/删空才恢复/接管者不覆盖）：

**卸载语义（2026-08-04 明确+实测）**：只卸载「能确定属于 VxAPO 的部分」，绝不碰其他 APO：
0. **先停 AudioSrv**（`install/audiodg::stop_audio_service`）——作用**不是**解锁：
   写/删 `FxProperties` 值只需句柄具备 `KEY_SET_VALUE`（`open_for_write` 即是），
   2026-09-16 实测：音频播放中、DLL 已被 audiodg 加载、audiodg 持有点端时删槽位值
   同样成功。真正的理由是 ① **释放 `vxapo_driver.dll` 模块映像**（audiodg 不退出则
   紧随其后的重装/换 DLL 会因文件被占用而覆盖失败；NSIS `installer-hooks.nsh`
   同样为此在安装/卸载前停服务）② 让本流程末尾的端点重启
   （`pnputil /restart-device`）**立刻生效**（引擎会缓存端点 APO 链）
1. 定位端点 FxProperties（不存在 → 视为未安装，返回成功）；**用 `open_for_write` 打开**
   （删除值需写权限；SAM_ALL 超权限在 MMDevices 端点键被拒 0x80070005）
2. **只删 VxAPO 的 CLSID**：遍历 5 槽位，仅当槽位值 == VxAPO PRE/POST CLSID 才删除
   （别的 APO 槽位不动——EAPO 重装占回时不受影响）；
   **注意**：不能用 `read_all_slots`（期望端点根键、内部再 open FxProperties）；当前句柄已是
   FxProperties 键，需用 `read_slot_value` 直接在 fx_key 上读（2026-08-04 实测：uninstall 后
   slot 残留 VxAPO CLSID 即此因）
3. **不写回第三方 APO**（v8.10，2026-08-05 纠正：**卸载 ≠ 快照恢复**）——
   install 时备份的 EAPO 等前任 APO 只在 `snapshot restore` 时恢复；卸载只清 VxAPO 自己的 CLSID
4. **恢复 CAPX MSFX 模板**（v9.0）：先于删除信息区执行，按 `SysFxBackups` 恢复微软
   StreamEffect/ModeEffect；无备份时按 CAPX 默认 CLSID 兜底
5. 删除信息区 `HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}` 整个键（v8.5 产品意图：
   卸载必删键 → 再次安装回「全量路径」判定）
6. 清理 FxProperties 上的旧遗留 VxAPO 配置值（childGuid/allowSilentBuffer/autoAdjust/version，
   best-effort）+ DisableEnhancements
7. **重启 AudioSrv**（`install/audiodg::restart_audio_service`）恢复输出

**事务回滚**（原 rollback.rs 职责，合并至此）：

```rust
enum RollbackAction {
    DeleteKey(String),
    RestoreValue { key_path: String, name: String, backup: Vec<u8> },
}

struct Transaction { ... }
// Drop 时未 commit 则逆序执行全部回滚动作
```

**禁止**：不依赖 `pipeline/`、`config/`

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

/// 停止 Windows 音频服务（只停不启，卸载前置）。
pub fn stop_audio_service() -> Result<()>;

/// 重启 Windows 音频服务（安装/卸载收尾，EAPO 安装对齐）。
pub fn restart_audio_service() -> Result<()>;

/// 确保音频服务处于运行状态（不重启，仅在停止时启动）。
pub fn ensure_audio_service_running() -> Result<()>;

/// 依赖服务感知地停止音频服务（先停 AudioEndpointBuilder 等依赖，轮询 STOPPED）。
pub fn stop_audio_service_with_dependents(stop_timeout_secs: u32) -> Result<()>;

/// 依赖服务感知地启动音频服务（轮询 RUNNING）。
pub fn start_audio_service_with_dependents(start_timeout_secs: u32) -> Result<()>;

/// 停服 + 启服并等待结果（--verify 安装流程用）。
pub fn restart_audio_service_wait(stop_timeout_secs: u32, start_timeout_secs: u32) -> Result<()>;

/// 定向重启单个端点设备（render / capture），不整服重启。
pub fn restart_endpoint_device(device_guid: &str, is_capture: bool) -> Result<()>;

/// 事件驱动等待 `audiodg.exe` 全部退出（Toolhelp 快照取 PID →
/// OpenProcess(SYNCHRONIZE) → WaitForSingleObject，进程一退出立即返回；上限两轮扫描）。
///
/// 用途：停服/`taskkill` 之后确认 **模块映像**已释放（audiodg 不退出则
/// `vxapo_driver.dll` 仍被占用，随后的重装/换 DLL 会覆盖失败）。
/// 槽位值的写/删不依赖本函数（只需 KEY_SET_VALUE 句柄）。
/// 返回 true = 已无 audiodg；false = 超时仍有残留（调用方 best-effort 继续）。
pub fn wait_for_audiodg_exit(timeout_ms: u32) -> bool;
```

**注册表路径**：`HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Audio`

**值名**：`DisableProtectedAudioDG`（`REG_DWORD`）

**语义**：值不存在或 != 1 → 阻止；值存在且 == 1 → 允许。

**实现要点**：
- `disable()` 使用 `RegKey::create()` 获取可写句柄，`write_dword()` 写入 1，`handle()` 获取底层 HKEY
- `restore()` 使用 `RegKey::open()` 获取句柄，`handle()` 获取底层 HKEY，`delete_value()` 删除值（值不存在静默返回）
- 不使用 unsafe 指针转换（pre 的 `RegKey::handle()` 已暴露底层句柄）

**禁止**：不知道 `pipeline/` 的存在

---

### 5.7 `install/device/stale.rs`

**职责**：Windows 重新枚举端点后，旧 GUID 的槽位键消失但 VxAPO 记录残留
（`HKLM\SOFTWARE\VxAPO\Child APOs\{旧GUID}`、`C:\ProgramData\VxAPO\{旧GUID}`、
`snapshots\{旧GUID}.json`）。本模块以**设备稳定身份**（端点历史 / 设备实例 ID / 硬件 ID）
把旧记录匹配到当前活跃端点，并支持迁移/清理。

**分层匹配（v9.24，2026-09-16 修复）**：Windows 大版本更新会重排端点 GUID，并可能把
老端点键 `...\MMDevices\Audio\{Render|Capture}\{oldGuid}` **整体删除**——此时原本
「读老端点键拿实例 ID」的配对路径失效，记录退化为 `unmatched`（实测 App 只剩清理出口，
恢复逻辑不触发）。现按优先级分层匹配，命中来源写入 `matched_by`：

1. `endpoint_history`：老 GUID 出现在活跃端点的端点历史属性
   `{4b416b7d-8501-40c1-acfd-97aa9bdc17c8},1`（REG_MULTI_SZ，元素形如
   `{0.0.0.00000000}.{guid}`）里；或活跃 GUID 出现在记录已落盘的 `EndpointHistory` 值里；
2. `device_instance_id`：老端点键仍可读时的实例 ID（旧路径保留）；
3. `stored_identity`：记录键里安装/迁移时落盘的 `DeviceInstanceId`；
4. `hardware_id`：硬件 ID 相交且产品名一致，且**唯一**命中（USB 换口兜底）。

多候选（歧义）或全部未命中 → `unmatched`：只提供清理，不猜、不自动迁移。

**引用来源**：
- `crate::install::device::{endpoint::query_endpoint, info::{enumerate_devices, detect_mode_for_guid}}`
- `crate::install::device::slots::{child_apo_key_exists, ChildApoKind, InstallMode, CHILD_APO_PATH_ROOT, FX_PROPERTIES_KEY, INSTALL_VERSION}`
- `crate::install::device::sysfx::{decode_backups, SYSFX_BACKUP_VALUE}`
- `crate::install::selector::operation::{find_endpoint_path, write_install_config, InstallConfig}`
- `crate::sys::registry::{delete_tree, split_key, RegKey, RegValue}`
- `crate::utils::guid::parse_guid_string`、`crate::utils::vx_error::{Result, VxApoError}`

**导出给**：`vxapo-cli`（`stale list/migrate/cleanup/fix-acl`）、App 后端命令
`list_stale_installs` / `migrate_stale_install` / `cleanup_stale_install` / `repair_stale_acl`；
`selector/operation.rs::uninstall_endpoint` 的残留兜底路径。

**公开 API**：

```rust
#[derive(Serialize)] #[serde(rename_all = "snake_case")]
pub struct StaleInstall {
    pub guid: String,
    pub device_instance_id: String,
    pub display_name: String,
    pub matched_by: Option<String>,    // endpoint_history | device_instance_id | stored_identity | hardware_id
    pub config_path: Option<String>,
    pub config_mtime_ms: Option<u64>,
    pub snapshot_path: Option<String>,
    pub snapshot_mtime_ms: Option<u64>,
    pub premix_slot: Option<String>,
    pub postmix_slot: Option<String>,
    pub inferred_mode: String,
    pub has_child_backup: bool,
    pub has_sysfx_backup: bool,
    pub target_guid: Option<String>,
    pub target_name: Option<String>,
    pub target_state: String,          // matched_healthy | matched_partial | unmatched
}

#[derive(Serialize)] #[serde(rename_all = "snake_case")]
pub struct MigrationReport {
    pub success: bool,
    pub target_guid: String,
    pub config_from: Option<String>,
    pub snapshot_from: Option<String>,
    pub config_migrated: bool,
    pub snapshot_migrated: bool,
    pub install_repaired: bool,
    pub removed_guids: Vec<String>,
    pub warnings: Vec<String>,
}

/// 扫描旧 GUID 安装记录，并匹配当前活跃端点（只读）。
pub fn list_stale_installs() -> Result<Vec<StaleInstall>>;

/// 清理一个无法匹配到活跃端点的旧 GUID 记录。
pub fn cleanup_orphan(guid: &str) -> Result<()>;

/// 修复迁移后 config/snapshot 的 ACL：给交互用户授予 Modify。
pub fn fix_config_acl(guid: &str) -> Result<()>;

/// 把旧 GUID 安装迁移到新 GUID。
///
/// `config_from` / `snapshot_from` 为显式来源；缺省按「最新 config、最早 snapshot」
/// 在同设备实例的旧记录与目标现有文件之间选择。
pub fn migrate_install(
    from: &str,
    to: &str,
    config_from: Option<&str>,
    snapshot_from: Option<&str>,
) -> Result<MigrationReport>;
```

**关键路径**：
- 配置根：`C:\ProgramData\VxAPO`；快照：`C:\ProgramData\VxAPO\snapshots`；
  迁移备份：`C:\ProgramData\VxAPO\_migration_backup`。
- 子 APO 根：`HKLM\SOFTWARE\VxAPO\Child APOs`。

**实现要点**：
- `target_state` 由「分层匹配结果 + `target_health`」决定；`matched_by` 记录命中来源
  （未命中为 `null`），供 CLI `stale list` 与 App 诊断展示。
- 记录键（`Child APOs\{guid}`）新增 4 个**值**（不是子键——迁移的 `copy_values`
  只搬值，写成子键会在迁移时静默丢失）：`DeviceInstanceId`（REG_SZ）、
  `DeviceHardwareIds`（REG_MULTI_SZ）、`DeviceProductName`（REG_SZ）、
  `EndpointHistory`（REG_MULTI_SZ，小写带花括号）。它们**不参与** APO 加载——
  加载只看 `FxProperties` 槽位 `{d04e05a6-…},0/3/5/6/7`；该键仍只由 install 建、
  uninstall 整键删，语义不变。
- 安装期由 `selector/operation.rs::write_child_apo_config` 写入身份值；迁移成功后
  用活跃端点身份刷新，并把 `EndpointHistory` 写回并集（已落盘历史 ∪ 活跃端点历史 ∪
  各旧 GUID ∪ 新 GUID，去重排序），使后续再次刷新 GUID 仍能认回同一设备。
- 迁移前把会被覆盖的目标文件备份到 `_migration_backup\{target_guid}\`，
  `removed_guids` 记录被删除的旧 GUID，`warnings` 收集非致命问题（如 ACL 修复失败）。
- `fix_config_acl` 用 `icacls` 给交互用户 SID `*S-1-5-4` 授予 Modify（目录递归 `(OI)(CI)M /T`）。
- `stale migrate` **全程不停服**（2026-09-16 实测）：写/删端点 `FxProperties` 值只需
  句柄具备 `KEY_SET_VALUE`（`RegKey::open_for_write` 即是），与 audiodg 是否持有点端
  无关——在活动音频流上删除槽位值同样成功；历史 0x80070005 来自旧实现用 `SAM_ALL`
  （含 CreateSubKey 位，ACL 未授予）或只读句柄打开该键。健康路径只写 ProgramData
  文件与 `HKLM\SOFTWARE\VxAPO\Child APOs\*` 值，`config.toml` 也不会被"占用"
  （DLL 一次性 `fs::read` + 目录通知热重载，App 一直用 tmp+rename 改写）。
  修复分支（目标安装状态不健康 → `write_install_config` 改写槽位）在写完后**重启该端点**
  （`audiodg::restart_endpoint_device`）让变更生效——引擎会缓存端点 APO 链，只改注册表
  不会立刻重载（实测：删掉槽位值后新起的流仍加载旧 APO）。收尾统一调
  `ensure_audio_service_running()`（已运行则幂等返回）。
- `copy_file` 的 `tmp → 目标` 替换带有限重试（5 次 × 50 ms，仅对
  ACCESS_DENIED/SHARING_VIOLATION/LOCK_VIOLATION 重试）——覆盖"不停服时 rename
  恰好撞上 DLL 读取"的微秒级窗口，避免一次瞬时共享冲突让整次迁移硬失败。
- 需管理员权限的调用由 CLI / App 提权路径完成；本模块自身不弹 UAC。

**禁止**：依赖 `pipeline/`、`config/` 的运行时语义。

---

### 5.8 `install/device/identity.rs`

**职责**（v9.24 新增）：跨端点 GUID 刷新的**稳定身份**读取与落盘，供 `stale.rs` 配对、
`selector/operation.rs` 安装期写入身份使用。不依赖老端点键，因此端点键被 Windows
整体删除后仍可工作。

**身份来源**：
- 活跃端点 `Properties` 子键：`PKEY_DeviceInstanceId`（`{b3f8fa53-…},2`）、
  `PKEY_Device_ProductName`（`{b3f8fa53-…},6`）、
  `PKEY_DeviceInterface_FriendlyName`（`{a45c254e-…},2`）、
  设备节点硬件 ID（`{9dad2fed-3c19-4cde-b3c9-1bd56be25698},0`，REG_MULTI_SZ）、
  端点历史（`{4b416b7d-8501-40c1-acfd-97aa9bdc17c8},1`，REG_MULTI_SZ）。
- VxAPO 记录键值：`DeviceInstanceId` / `DeviceHardwareIds` / `DeviceProductName` /
  `EndpointHistory`（与 `operation.rs` 写入的值名一致）。

**纯函数（可单测）**：
- `normalize_device_id`：`\\?\`、`{1}.`、`{2}.` 前缀**循环**剥离（实证
  `{2}.\\?\usb#vid_…` 叠加形态）、`#`→`\`、大写、去首尾 `\`；
- `normalize_endpoint_guid`：`{0.0.0.00000000}.{guid}` / `{0.0.1.00000000}.{guid}` /
  接口路径 / 裸 GUID → 小写 `{guid}`（非法输入返回 `None`）；
- `normalize_endpoint_history` / `merge_endpoint_history`：解析、去重、排序。

**实现要点**：
- `read_endpoint_identity` 缺失值一律降级为空值（不返回错误），保证扫描不因单台设备
  属性异常而整体失败。
- 身份值一律写成**值**而非子键：`stale.rs::copy_values` 迁移时只搬值。
- 新增/读取身份**不改变** APO 加载：加载只看 `FxProperties` 槽位。

**禁止**：依赖 `pipeline/`、`config/`；写入 `MMDevices`（端点键只读）。
