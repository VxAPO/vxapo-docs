## 七、`object/` 模块规范（最终版）

**边界**：胶水层。Windows 加载 DLL 时创建的 COM 对象。允许依赖所有模块。

**允许依赖**：所有其他模块（`sys/`、`pipeline/`、`install/`、`config/`、`utils/`、`telemetry/`）

---

### 完整模块树

```
object/
├── apo.rs              # 核心模块
│                        # ├── ApoObject（#[implement]，三接口）
│                        # ├── ApoObjectState（核心状态）
│                        # ├── StateCell（原子状态机 + RAII 回退守卫）
│                        # ├── APOGUID_NOKEY / NOVALUE / GUID_NULL 常量
│                        # ├── is_special_guid / is_valid_apo_guid / guid_matches
│                        # ├── ConnectionFormat / LockConfig / FormatSupportResult
│                        # ├── extract_format_from_media_type
│                        # ├── IAudioProcessingObject_Impl（含完整校验）
│                        # ├── IAudioProcessingObjectRT_Impl（含双链过渡 + catch_unwind）
│                        # ├── IAudioProcessingObjectConfiguration_Impl（含 RAII 回退）
│                        # ├── hot_reload（watcher 回调）
│                        # └── 编译期断言
│
├── child.rs            # 子 APO COM 生命周期管理（类型化接口持有）
│
├── factory.rs          # #[implement(IClassFactory)] + LOCK_COUNT
│
├── ref_count.rs        # INST_COUNT 原子计数
│
├── vx_reg_props.rs     # CLSID×2 + REG_PROPS×2 + 查询辅助 + COM 注册条目
│
└── dll_exports.rs      # DllMain + 四个 COM 导出函数
```

---

### 引用约束总表

| 模块 | 可依赖 | 不可依赖 |
|------|--------|----------|
| `object/apo.rs` | `sys/com/`（全部）、`sys/registry`、`sys/known_folder`、`object/child.rs`、`object/vx_reg_props.rs`、`object/ref_count.rs`、`object/factory.rs`、`pipeline/`、`config/`、`install/audiodg`、`telemetry/logger`、`utils/` | — |
| `object/child.rs` | `sys/com/prelude`、`sys/com/apo_interfaces`、`sys/com/apo_types` | `object/apo.rs`（禁止循环） |
| `object/factory.rs` | `sys/com/prelude`、`sys/com/apo_interfaces`、`object/apo.rs`、`object/vx_reg_props.rs`、`object/ref_count.rs` | — |
| `object/ref_count.rs` | `core` | 所有其他 |
| `object/vx_reg_props.rs` | `sys/com/prelude`、`sys/com/apo_types` | 所有其他 |
| `object/dll_exports.rs` | `object/*`、`sys/com/prelude`、`telemetry` | — |

> 较原规范变更：`dll_exports.rs` 不再依赖 `installation/clsid_entries`，改为依赖 `object/vx_reg_props`（注册条目已合并）。

---

### 7.1 `object/apo.rs` — ApoObject 定义

**职责**：ApoObject 实现三个 APO 接口，持有完整生命周期状态（原子状态机 + 双链过渡 + 互斥保护）。

**引用来源**：
- `crate::sys::com::apo_interfaces::*`（三个 APO trait + IAudioMediaType）
- `crate::sys::com::apo_types::*`（POD 类型）
- `crate::sys::com::prelude::*`（IUnknown、implement、HRESULT 常量）
- `crate::sys::registry::RegKey`
- `crate::object::child::ChildApo`
- `crate::object::vx_reg_props::{CLSID_VXAPO_PRE_MIX, CLSID_VXAPO_POST_MIX, props_for_clsid}`
- `crate::object::ref_count`
- `crate::pipeline::process::{process_audio, process_chain_interleaved, apply_error_policy, ProcessParams, ProcessStatistics, ErrorPolicy}`
- `crate::pipeline::chain::Chain`
- `crate::pipeline::context::PipelineContext`
- `crate::pipeline::format::{extract_format, is_float_format, AudioFormat}`
- `crate::sys::audio_defs::get_channel_names`
- `crate::sys::known_folder::documents_folder`（Initialize per-device 路径解析，v7.2）
- `crate::sys::com::apo_types::APOInitSystemEffects`（Initialize 初始化数据，v7.2）
- `crate::sys::com::prelude::guid_to_string`（设备 GUID 格式化，v7.2）
- `crate::pipeline::dsp::filter::{DspContext, DeviceType, ProcessingStage}`
- `crate::pipeline::dsp::transition::{SmoothingProvider, mix_buffers, default_smoothing_length}`
- `crate::config::parser::ConfigParser`
- `crate::config::watcher::ConfigWatcher`
- `crate::install::audiodg::ensure_can_load`
- `crate::utils::vx_error::*`
- `crate::telemetry::logger::Logger`

---

#### 7.1.1 特殊 GUID 常量

```rust
/// FxProperties 键不存在。
///
/// 设备注册表路径下根本没有 FxProperties 子键。
/// init.rs 在此状态下回退到默认初始化逻辑。
pub const APOGUID_NOKEY: GUID = GUID::from_values(
    0x00000000, 0x0000, 0x0000,
    [0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x01],
);

/// FxProperties 值为空或被其他 APO 占据。
///
/// FxProperties 键存在，但目标 GUID 槽位值为空字符串、
/// 或者已被系统 APO / 第三方 APO 写入了非 VxAPO 的 GUID。
/// apo_conf.rs 在此状态下跳过子 APO 创建。
pub const APOGUID_NOVALUE: GUID = GUID::from_values(
    0x00000000, 0x0000, 0x0000,
    [0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x02],
);

/// 空 GUID（全零），用于初始化和比较。
pub const GUID_NULL: GUID = GUID::from_values(
    0x00000000, 0x0000, 0x0000,
    [0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00],
);
```

**查询辅助**：

```rust
/// 判断给定 GUID 是否为特殊值（NOKEY / NOVALUE）。
pub fn is_special_guid(guid: &GUID) -> bool;

/// 判断给定 GUID 是否为有效的 APO CLSID（非特殊值、非空）。
pub fn is_valid_apo_guid(guid: &GUID) -> bool;

/// 判断两个 GUID 是否代表同一个 APO。特殊值之间不相等（NOKEY ≠ NOVALUE）。
pub fn guid_matches(a: &GUID, b: &GUID) -> bool;
```

---

#### 7.1.2 ApoObjectState

```rust
/// APO 对象的核心状态。
///
/// 由 ApoObject 的 ap_state: Mutex<ApoObjectState> 持有。
#[derive(Debug)]
pub struct ApoObjectState {
    /// 此实例的 CLSID（PreMix 或 PostMix）。
    pub clsid: GUID,

    /// 子 APO 的 CLSID（Initialize 时从 APOInitSystemEffects 解析）。
    /// Some(guid) 表示需要创建子 APO；None 表示无子 APO（降级模式，Note 57）。
    pub child_apo_clsid: Option<GUID>,

    /// 是否已通过 LockForProcess 锁定。
    ///
    /// 锁定前不得调用 APOProcess（Windows 约束）。
    pub is_locked: bool,

    /// 采样率（LockForProcess 时确定）。
    pub sample_rate: u32,

    /// 输入通道数。
    pub input_channel_count: u32,

    /// 输出通道数。
    pub output_channel_count: u32,

    /// 通道掩码。
    pub channel_mask: u32,

    /// 每样本位数。
    pub bits_per_sample: u32,

    /// 是否允许静音缓冲区快速路径（Note 11）。
    pub allow_silent_buffer_modification: bool,
}
```

```rust
impl ApoObjectState {
    /// 创建新的 APO 对象状态。由 ClassFactory::CreateInstance 调用。
    pub fn new(clsid: GUID) -> Self;

    /// 判断此实例是 PreMix APO。
    pub fn is_pre_mix(&self) -> bool;   // clsid == CLSID_VXAPO_PRE_MIX

    /// 判断此实例是 PostMix APO。
    pub fn is_post_mix(&self) -> bool;  // clsid == CLSID_VXAPO_POST_MIX

    /// 锁定处理流程（LockForProcess 成功后调用）。
    pub fn lock_for_process(
        &mut self,
        sample_rate: u32,
        input_channels: u32,
        output_channels: u32,
        channel_mask: u32,
        bits_per_sample: u32,
    );

    /// 解锁处理流程（UnlockForProcess 调用）。
    pub fn unlock_for_process(&mut self);
}
```

---

#### 7.1.3 ApoObject 字段

```rust
#[implement(
    IAudioProcessingObject,
    IAudioProcessingObjectRT,
    IAudioProcessingObjectConfiguration
)]
pub struct ApoObject {
    /// APO 自身 CLSID（构造时确定，不可变）。
    pub(crate) clsid: GUID,

    /// 原子状态机（Created → Initialized → Locked）。
    pub(crate) state_cell: StateCell,

    /// 核心运行时状态（Chain、过渡、PipelineContext、临时缓冲区）。
    pub(crate) mutex: Mutex<ApoObjectInner>,

    /// 格式/通道/锁定状态（ApoObjectState）。
    pub(crate) ap_state: Mutex<ApoObjectState>,

    /// 配置加载器。
    pub(crate) config_loader: ConfigLoader,

    /// 配置文件监控器（后台线程）。
    pub(crate) watcher: Option<ConfigWatcher>,

    /// 配置文件路径。
    pub(crate) config_path: String,

    /// 日志器。
    pub(crate) logger: Logger,

    /// 延迟采样数（控制线程写，GetLatency 读）。
    pub(crate) latency_samples: AtomicU32,

    /// 延迟帧数（CalcInputFrames/CalcOutputFrames 用，免锁）。
    pub(crate) latency_frames_atomic: AtomicU32,

    /// 处理统计。
    pub(crate) process_stats: ProcessStatistics,

    /// 子 APO（Initialize 阶段创建）。
    pub(crate) child_apo: Option<ChildApo>,
}

pub struct ApoObjectInner {
    pub current_chain: Box<Chain>,
    pub outgoing_chain: Option<Box<Chain>>,
    /// 退役链（R1/v6.9，EAPO `previousConfig` 借鉴）：过渡完成帧由 RT 线程
    /// 将 outgoing_chain **移入**（`.take()`，零析构），再由控制线程
    /// （hot_reload / UnlockForProcess / Reset）在锁内统一 drop。
    /// 重型滤波器析构绝不留在 RT 线程。
    pub retired_chain: Option<Box<Chain>>,
    pub transition: Option<SmoothingProvider>,
    pub pipeline_context: PipelineContext,
    pub temp_buffers: Vec<Vec<f32>>,
    pub temp_buffer_old: Vec<f32>,
    pub temp_buffer_new: Vec<f32>,
    pub pending_reload: bool,
    /// 加载中标志（R2/v6.9，EAPO loadSemaphore 对齐）：过渡完成后 APOProcess
    /// 触发一次的 hot_reload 会将此置回 false；期间新变更请求被忽略，
    /// 避免 pending_reload 被连续变更覆盖。
    pub reloading: bool,
}
```

> **R1（退役链，v6.9）**：过渡完成帧 RT 线程只做 `inner.retired_chain = inner.outgoing_chain.take()`——
> 移动 Box 所有权（零析构）。`current_chain = next` 后，旧链在 `retired_chain` 挂起，
> 由**控制线程**（下一次 hot_reload / UnlockForProcess / Reset 的锁内）统一 `drop`。
> 这是 EAPO `previousConfig` 槽的本质：把 RT 线程的析构开销与竞态窗口完全消除。

> **R2（阻塞式重载，v6.9，EAPO 信号量对齐）**：hot_reload 检测到
> `transition.is_some()` 时**直接返回不构建**（不再先 parse_file 后排队）——与 EAPO
> `loadSemaphore` 一致：过渡期间不加载，过渡完成后 APOProcess 触发一次重载。
> `reloading` 标志确保同一个过渡周期内至多触发一次；消除了排队式的重复解析、
> pending_reload 覆盖漏洞与额外标志保护。
> 在 10ms 过渡窗口下，"阻塞到过渡完成的解析延迟"完全可忽略（用户决策，R2 理由）。
```

```rust
// SAFETY: 音频引擎保证 LockForProcess / UnlockForProcess / APOProcess 不重叠。
// pipeline 操作通过 mutex 保护。
// latency_samples 和 latency_frames_atomic 是 AtomicU32。
unsafe impl Send for ApoObject {}
unsafe impl Sync for ApoObject {}

impl ApoObject {
    pub fn new(clsid: GUID) -> Self {
        object::ref_count::increment();
        // ...
    }
}

impl Drop for ApoObject {
    fn drop(&mut self) {
        object::ref_count::decrement();
    }
}
```

---

#### 7.1.4 原子状态机

```rust
/// APO 生命周期状态（对应 Windows 引擎驱动顺序）。
#[repr(u8)]
#[derive(Clone, Copy, PartialEq, Eq, Debug)]
pub enum ApoState {
    Created     = 0,   // 对象已创建，Initialize 未调用
    Initialized = 1,   // Initialize 成功；LockForProcess 未调用或已 Unlock
    Locked      = 2,   // LockForProcess 成功；APOProcess 可调用
}

impl ApoState {
    /// `true` 当且仅当 `Locked`（APOProcess 合法）。
    pub const fn allows_process(self) -> bool { matches!(self, ApoState::Locked) }
}

/// 状态转换失败详情（v6.6 补全，tympan-apo 借鉴）。
///
/// 携带 期望态 / 尝试目标态 / 实际观测态 三要素，便于定位状态机问题。
#[derive(Clone, Copy, PartialEq, Eq, Debug)]
pub struct TransitionError {
    pub expected: ApoState,
    pub attempted: ApoState,
    pub actual: ApoState,
}

/// 原子状态机载体。
pub struct StateCell {
    state: AtomicU8,
}

impl StateCell {
    pub fn new() -> Self;   // 初始 Created

    /// 获取当前状态（Acquire 序，RT 路径可安全调用）。
    pub fn current(&self) -> ApoState;

    /// `current() == Locked` 的快捷判断。
    pub fn is_locked(&self) -> bool;

    /// CAS 转换：成功返回 Ok，失败返回 TransitionError{expected, attempted, actual}。
    pub fn transition(&self, from: ApoState, to: ApoState) -> Result<(), TransitionError>;

    /// 无条件复位至 `Created`（`AcqRel` swap）。返回复位前状态。
    ///
    /// 用于 COM `Release` 最终引用释放 / 析构器将单元置于已知终态
    /// （DLL 卸载时的资源回收关键，v6.6 补全）。
    pub fn release(&self) -> ApoState;

    // ── 语义化便捷转换（失败即 TransitionError） ──
    pub fn initialize(&self) -> Result<(), TransitionError>; // Created → Initialized
    pub fn lock(&self)       -> Result<(), TransitionError>; // Initialized → Locked
    pub fn unlock(&self)     -> Result<(), TransitionError>; // Locked → Initialized
}
```

**状态转换规则**：
- `Initialize`：要求 `Created` → `Initialized`
- `LockForProcess`：要求 `Initialized` → `Locked`
- `APOProcess`：要求 `Locked`（不改变状态）
- `UnlockForProcess`：要求 `Locked` → `Initialized`
- `release()`：任意状态 → `Created`（仅析构/最终 Release 路径使用）

> **v6.6 设计说明（O2）**：引入 `TransitionError`（expected/attempted/actual）替代
> 原 `VxApoError::State("非法状态转换")` 纯字符串，便于快速定位状态机偏差；
> `release()` 保证 DLL 卸载时无论对象处于何种状态都可安全复位终态。
> 错误转换为 `HRESULT` 时统一 `State(TransitionError)` 映射为 `APOERR_ALREADY_INITIALIZED` 等对应码（实现端确定）。

---

#### 7.1.5 RAII 回退守卫

```rust
struct LockForProcessGuard<'a> {
    apo: &'a ApoObject,
    is_active: bool,
}

impl<'a> LockForProcessGuard<'a> {
    fn new(apo: &'a ApoObject) -> Self {
        Self { apo, is_active: true }
    }
    fn disarm(mut self) {
        self.is_active = false;
    }
}

impl<'a> Drop for LockForProcessGuard<'a> {
    fn drop(&mut self) {
        if self.is_active {
            if let Err(e) = self.apo.state_cell.transition(ApoState::Locked, ApoState::Initialized) {
                self.apo.logger.log(LogLevel::Error, "LockForProcess: state rollback failed");
            }
        }
    }
}
```

---

#### 7.1.6 格式提取辅助

```rust
/// 从 `IAudioMediaType*` 提取 WAVEFORMATEX 信息。
///
/// 返回 `(sample_rate, channels, channel_mask, bits_per_sample)`。
///
/// 通过 `#[interface]` trait 方法 `IAudioMediaType::GetAudioFormat()` 安全调用，
/// 不使用裸 vtable 索引。
unsafe fn extract_format_from_media_type(format_ptr: *mut c_void) -> Option<(u32, u32, u32, u32)>;
```

---

#### 7.1.7 格式协商业务类型

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum FormatSupportResult {
    Supported,
    UnsupportedWithAlternative,
    Unsupported,
}

#[derive(Debug, Clone)]
pub struct FormatNegotiation {
    pub input_channels: u32,
    pub output_channels: u32,
    pub input_sample_rate: u32,
    pub output_sample_rate: u32,
    pub input_bits_per_sample: u32,
    pub output_bits_per_sample: u32,
    pub input_channel_mask: u32,
    pub output_channel_mask: u32,
}

pub fn check_format_support(neg: &FormatNegotiation) -> FormatSupportResult;

/// 连接格式描述（LockForProcess 参数的简化表示）。
#[derive(Debug, Clone)]
pub struct ConnectionFormat {
    pub channel_count: u32,
    pub sample_rate: u32,
    pub bits_per_sample: u32,
    pub channel_mask: u32,
}

/// LockForProcess 的输入输出描述。
#[derive(Debug, Clone)]
pub struct LockConfig {
    pub inputs: Vec<ConnectionFormat>,
    pub outputs: Vec<ConnectionFormat>,
}
```

---

#### 7.1.8 `Initialize`（含 per-device 配置路径解析，v7.2）

**职责**：初始化。解析 `APOInitSystemEffects`（提取子 APO CLSID + 设备 GUID），
确定 per-device 配置路径并确保 `Documents\VxAPO\{GUID}\config.txt` 存在。

```rust
fn Initialize(&self, cb_data_size: u32, pby_data: *mut u8) -> HRESULT {
    // 1. 参数校验：pby_data 非空、cb_data_size >= size_of::<APOInitSystemEffects>()
    //    非法 → 不返回错误，**降级默认配置**（log::warn + resolve_config_path(None)），
    //    仍返回 Ok（v7.6 修订，P0-3 实现对齐②）：Initialize 失败会使 APO 无法加载、
    //    音频流停摆；降级到默认 passthrough 比硬失败更稳健（SDK 容错惯例）。
    // 2. state_cell.transition(Created, Initialized)；失败 → 对应 HRESULT
    // 3. 解析 APOInitSystemEffects（强制类型转换 pby_data）：
    //    - pAPOSystemEffectsProperties（IPropertyStore）→ GetValue(PKEY_AudioEndpoint_GUID)
    //      → PROPVARIANT（VT_CLSID）→ puuid → 设备 endpoint GUID（v7.6 修订，P0-3 现实反馈①）
    //    - 提取子 APO CLSID（无子 APO → None，降级模式 Note 57）
    // 4. 如有子 APO CLSID，创建 ChildApo（失败降级为无子 APO，不阻塞初始化）
    // 5. 确定 per-device 配置路径（load_device_config）：
    //    a. sys::known_folder::documents_folder() → Documents 路径
    //       （SHGetKnownFolderPath(FOLDERID_Documents) 安全收窄）
    //    b. 设备 GUID → 大写格式字符串（guid_to_string：{XXXXXXXX-...}）
    //    c. 拼接：{Documents}\VxAPO\{GUID}\  → config_path
    //    d. 目录不存在 → std::fs::create_dir_all 创建
    //    e. config.txt 不存在 → 写入默认 passthrough（空文件或仅注释行）
    // 6. 启动 watcher（v7.3，P0-4）：
    //    watch_dir = config_path 父目录（Documents\VxAPO\{GUID}）
    //    经 config/watcher.rs::ConfigWatcher::new(watch_dir, poll_interval_ms, dedup_window_ms)
    //    轮询检测变更 → WatchEvent::ConfigFileChanged → hot_reload（7.1.18）
    // 7. self.config_path = path；返回 S_OK
}
```

**watcher 启动约定（v7.3，P0-4）**：
> - 监控目录 = `config_path` 父目录（`Documents\VxAPO\{GUID}`），非 config.txt 文件本身
> - 轮询间隔默认 2000ms、去重窗口默认 500ms（`config 6.2`）
> - `ConfigFileChanged` / `ConfigFileDeleted` 事件经 `hot_reload`（7.1.18）处理：
>   R2 阻塞式（过渡在途/加载中直接返回）、过渡完成触发、退役链 R1 控制线程析构
> - `ConfigWatcher` 生命周期与 APO 实例一致（`ApoObject.watcher` 字段持有），
>   `UnlockForProcess` / `Reset` 不停止 watcher（配置热重载跨锁定周期持续生效）

**config_path 确定规则（v7.2，P0-3）**：

```
Documents\VxAPO\{GUID}\config.txt
```

- `{GUID}`：从 `APOInitSystemEffects.pAPOSystemEffectsProperties`（`IPropertyStore`）取
  `PKEY_AudioEndpoint_GUID`（PROPVARIANT VT_CLSID 的 `puuid`）获得端点 GUID（v7.6 修订），
  经 `sys/com/prelude::guid_to_string` 格式化为大写 `{XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX}`
- **目录自动创建**：`VxAPO\{GUID}` 目录不存在时 `std::fs::create_dir_all` 创建
- **config 默认写入**：config.txt 不存在时写入默认 passthrough（空文件或仅注释行，
  DLL 不创建预设——预设由 App/CLI 管理，dll 只兜底 passthrough）
- **无设备 GUID 兜底**：`PKEY_AudioEndpoint_GUID` 提取失败 / 值为空（无端点属性）时，
  回退单实例共用路径 `Documents\VxAPO\_default\config.txt`（日志告警，不阻断初始化）
- **Documents 路径获取失败兜底（v7.6 修订，P0-3 实现对齐③）**：`documents_folder()` 失败时
  回退**固定单实例路径** `C:\ProgramData\VxAPO\config.txt`（`DEFAULT_CONFIG_PATH` 常量，
  非 `_default` 子目录——Documents 本身不可用时库路径也无意义，取系统级固定位置）

**引用来源**（追加）：
- `crate::sys::known_folder::documents_folder`
- `crate::sys::com::apo_types::APOInitSystemEffects`
- `crate::sys::com::prelude::guid_to_string`
- `std::fs`（create_dir_all / 默认 config 写入）

---

#### 7.1.9 `LockForProcess`

**完整流程**：

```rust
fn LockForProcess(&self, num_input, pp_inputs, num_output, pp_outputs) -> HRESULT {
    let mut inner = self.mutex.lock().unwrap();

    // 状态转换
    if let Err(e) = self.state_cell.transition(ApoState::Initialized, ApoState::Locked) {
        return e.into();
    }

    // RAII 回退守卫：所有 ? 失败时自动回退状态
    let guard = LockForProcessGuard::new(self);

    // 提取格式（通过 #[interface] trait 方法，非裸 vtable）
    let input_desc = unsafe { &**pp_inputs };
    let output_desc = unsafe { &**pp_outputs };
    let input_format = extract_format(input_desc.format)?;
    let output_format = extract_format(output_desc.format)?;
    if input_format.sample_rate != output_format.sample_rate {
        return E_INVALIDARG;
    }

    // 强制要求输入输出通道数相同
    if input_format.channels != output_format.channels {
        return APOERR_INVALID_CONNECTION_FORMAT;
    }

    // 构建 PipelineContext（不含 latency_frames）
    let ctx = PipelineContext {
        sample_rate: input_format.sample_rate,
        input_channels: input_format.channels,
        output_channels: output_format.channels,
        channel_mask: input_format.channel_mask,
        max_frame_count: input_desc.max_frame_count as usize,
    };

    // 加载配置 → 构建 Chain
    let dsp_ctx = DspContext {
        sample_rate: input_format.sample_rate,
        channel_count: input_format.channels,
        channel_mask: input_format.channel_mask,
        max_frame_count: input_desc.max_frame_count,
        channel_names: get_channel_names(input_format.channel_mask),
        bits_per_sample: input_format.bits_per_sample,
        device_type: if is_capture { DeviceType::Capture } else { DeviceType::Render },
        stage: ProcessingStage::None,
        variables: HashMap::new(),
    };
    let filters = self.config_parser.parse_file(&self.config_path, &dsp_ctx)?;
    let mut chain = Chain::new();
    for f in filters { chain.add_filter(f)?; }
    let total_latency = chain.total_latency();

    // 预分配过渡缓冲区
    let max_ch = ctx.input_channels.max(ctx.output_channels) as usize;
    let max_samples = ctx.max_frame_count * ctx.input_channels.max(ctx.output_channels) as usize;
    let temp_buffer_old = vec![0.0f32; max_samples];
    let temp_buffer_new = vec![0.0f32; max_samples];

    // 检查 DisableProtectedAudioDG
    ensure_can_load()?;

    // 写入 inner
    inner.current_chain = Box::new(chain);
    inner.outgoing_chain = None;
    inner.retired_chain = None;   // R1：新锁定周期无退役链
    inner.transition = None;
    inner.pipeline_context = ctx;
    inner.temp_buffers = vec![vec![0.0f32; ctx.max_frame_count]; max_ch];
    inner.temp_buffer_old = temp_buffer_old;
    inner.temp_buffer_new = temp_buffer_new;
    inner.pending_reload = false;
    inner.reloading = false;      // R2：新锁定周期无加载在途

    // 写入两个原子延迟值
    self.latency_samples.store(total_latency, Ordering::SeqCst);
    self.latency_frames_atomic.store(total_latency, Ordering::SeqCst);

    // 子 APO LockForProcess 委托
    if let Some(ref child) = self.child_apo {
        // 构造最小描述符委托子 APO 锁定
        // 子 APO 拒绝锁定时降级（不阻塞父 APO，Note 57）
    }

    // 成功：disarm guard，不回退
    guard.disarm();
    S_OK
}
```

---

#### 7.1.10 `UnlockForProcess`

```rust
fn UnlockForProcess(&self) -> HRESULT {
    // 子 APO UnlockForProcess 委托。
    // 子解锁失败不阻塞父解锁（与 Note 57 降级立场一致）：
    // UnlockForProcess 无重试语义，子 APO 可能已部分解锁，
    // 父 APO 继续执行自身解锁流程，失败仅记录日志用于诊断。
    if let Some(ref child) = self.child_apo {
        let hr = child.unlock_for_process();
        if hr.0 != 0 {
            self.logger.log(LogLevel::Warn, "child APO UnlockForProcess failed");
        }
    }

    let mut inner = self.mutex.lock().unwrap();
    if let Err(e) = self.state_cell.transition(ApoState::Locked, ApoState::Initialized) {
        return e.into();
    }
    inner.current_chain = Box::new(Chain::new(0));
    inner.outgoing_chain = None;
    // R1（v6.9）：退役链由控制线程在锁内统一析构。
    inner.retired_chain = None;
    inner.transition = None;
    inner.pipeline_context = PipelineContext::new();
    inner.temp_buffers.clear();
    inner.temp_buffer_old.clear();
    inner.temp_buffer_new.clear();
    self.latency_samples.store(0, Ordering::SeqCst);
    self.latency_frames_atomic.store(0, Ordering::SeqCst);
    S_OK
}
```

---

#### 7.1.11 `APOProcess`

**完整伪代码**：

```rust
fn APOProcess(&self, input_props, output_props, params: &ProcessParams) {
    // 1. 状态检查
    if self.state_cell.current() != ApoState::Locked { return; }

    // 2. 提取输入切片（系统级 APO 硬性要求单输入）
    let input_slice = BufferInfo::from_prop(&input_props[0], params.input_channels).as_slice();

    // 3. 获取锁
    let mut inner = self.mutex.lock().unwrap();

    // 4. 过渡模式检查
    if let Some(outgoing) = &mut inner.outgoing_chain {
        let channels = params.input_channels as usize;
        let frames = params.valid_frame_count;
        let input_info = BufferInfo::from_prop(&input_props[0], channels);
        let input_slice = input_info.as_slice();

        // 旧链 → temp_buffer_old
        let result_old = process_chain_interleaved(
            inner.outgoing_chain.as_mut().unwrap(),
            input_slice, &mut inner.temp_buffer_old,
            channels, frames, &mut inner.temp_buffers,
        );
        apply_error_policy(result_old, input_slice, &mut inner.temp_buffer_old,
            params.error_policy, &self.process_stats);

        // 新链 → temp_buffer_new
        let result_new = process_chain_interleaved(
            &mut inner.current_chain,
            input_slice, &mut inner.temp_buffer_new,
            channels, frames, &mut inner.temp_buffers,
        );
        apply_error_policy(result_new, input_slice, &mut inner.temp_buffer_new,
            params.error_policy, &self.process_stats);

        // 混合
        let factor = inner.transition.as_mut().unwrap().advance().unwrap_or(1.0);
        let output_info = BufferInfo::from_prop_mut(&mut output_props[0], params.output_channels as usize);
        unsafe {
            mix_buffers(
                inner.temp_buffer_old.as_ptr(),
                inner.temp_buffer_new.as_ptr(),
                output_info.as_slice_mut().as_mut_ptr(),
                frames * channels, factor,
            );
        }
        output_props[0].buffer_flags = APO_BUFFER_FLAGS::Valid;

        // 过渡完成（v7.6 修订，P0-3 实现对齐①）：
        // - 旧链移入退役槽（R1，仅移动所有权零析构，控制线程锁内统一 drop）
        // - 触发重载时**不**先置 `reloading=true`——`reloading` 表示"正在解析中"
        //   （hot_reload 自己会置位）；若先置 true 再调 hot_reload，短锁检查
        //   `reloading==true` 会直接 return → 延迟重载被自己拦截。
        // - 防御 pending 残留但 transition 已被清空（无混合器）→ 直接复制输入到输出
        //   （bypass，APO 契约：每帧必须写输出）+ 旧链退役 + 可重载则补重载。
        // - advance() 返回 None（过渡已达上限）→ 按 factor=1.0（纯新链）输出。
        if finished {
            // R1：旧链移入退役槽（零析构），控制线程锁内统一析构。
            inner.retired_chain = outgoing.take();
            inner.transition = None;
            // R2 修正：保持 pending 语义，直接触发（不置 reloading）。
            if pending && !inner.reloading {
                inner.pending_reload = false;
                drop(inner);
                self.hot_reload();
                return;
            }
        }
    } else {
        // 正常模式
        let result = process_audio(input_props, output_props, params, &mut inner.current_chain, &self.process_stats, &mut inner.temp_buffers)
        if let Err(e) = result {
            self.logger.log(LogLevel::Error, "APOProcess: process_audio failed");
        }
    }
}

/// 统一处理链错误。
fn apply_error_policy(
    result: Result<()>,
    input_slice: &[f32],
    output_buffer: &mut [f32],
    policy: ErrorPolicy,
    stats: &ProcessStatistics,
) {
    match result {
        Ok(()) => {}
        Err(_) => {
            stats.error_count.fetch_add(1, Ordering::Relaxed);
            match policy {
                ErrorPolicy::Bypass => {
                    debug_assert!(input_slice.len() <= output_buffer.len());
                    output_buffer[..input_slice.len()].copy_from_slice(input_slice);
                    if output_buffer.len() > input_slice.len() {
                        output_buffer[input_slice.len()..].fill(0.0);
                    }
                }
                ErrorPolicy::Silence => {
                    output_buffer.fill(0.0);
                }
            }
        }
    }
}
```

---

#### 7.1.12 `CalcInputFrames` / `CalcOutputFrames`（无锁设计）

```rust
fn CalcInputFrames(&self, output_frames: u32) -> u32 {
    let latency = self.latency_frames_atomic.load(Ordering::Acquire);
    output_frames + latency
}

fn CalcOutputFrames(&self, input_frames: u32) -> u32 {
    let latency = self.latency_frames_atomic.load(Ordering::Acquire);
    if input_frames >= latency { input_frames - latency } else { 0 }
}
```

> 通过 `latency_frames_atomic`（`AtomicU32`）实现免锁帧数计算，RT 路径无需获取 mutex。

---

#### 7.1.13 `Reset`

```rust
fn Reset(&self) -> HRESULT {
    let mut inner = self.mutex.lock().unwrap();
    inner.current_chain = Box::new(Chain::new(0));
    inner.outgoing_chain = None;
    // R1（v6.9）：退役链由控制线程在锁内统一析构。
    inner.retired_chain = None;
    inner.transition = None;
    inner.pipeline_context = PipelineContext::new();
    inner.temp_buffers.clear();
    inner.temp_buffer_old.clear();
    inner.temp_buffer_new.clear();
    self.latency_samples.store(0, Ordering::SeqCst);
    self.latency_frames_atomic.store(0, Ordering::SeqCst);
    S_OK
}
```

---

#### 7.1.14 `GetLatency`

```rust
fn GetLatency(&self, p_latency: *mut REFERENCE_TIME) -> HRESULT {
    if p_latency.is_null() { return E_POINTER; }
    let inner = self.mutex.lock().unwrap();
    let latency_hns = self.latency_samples.load(Ordering::Acquire) as i64 * 10_000_000
        / inner.pipeline_context.sample_rate as i64;
    unsafe { *p_latency = latency_hns };
    S_OK
}
```

---

#### 7.1.15 `GetRegistrationProperties`

```rust
fn GetRegistrationProperties(&self, pp_props: *mut *mut APO_REG_PROPERTIES) -> HRESULT {
    if pp_props.is_null() { return E_POINTER; }
    let clsid = match self.ap_state.lock() {
        Ok(s) => s.clsid,
        Err(_) => return E_FAIL,
    };
    let props = match props_for_clsid(&clsid) {
        Some(p) => p,
        None => return E_FAIL,
    };
    // 使用 CoTaskMemAlloc 分配副本（调用方负责 CoTaskMemFree 释放）
    let size = std::mem::size_of::<APO_REG_PROPERTIES>();
    unsafe {
        let mem = windows::Win32::System::Com::CoTaskMemAlloc(size);
        if mem.is_null() { return E_OUTOFMEMORY; }
        std::ptr::copy_nonoverlapping(props as *const _ as *const u8, mem as *mut u8, size);
        *pp_props = mem as *mut APO_REG_PROPERTIES;
    }
    S_OK
}
```

> **内存管理约定**：返回的 `APO_REG_PROPERTIES` 由 `CoTaskMemAlloc` 分配，调用方必须通过 `CoTaskMemFree` 释放。与 Windows COM 标准惯例一致。

---

#### 7.1.16 `IsInputFormatSupported` / `IsOutputFormatSupported`

> **关键时序约束（v7.7 修订，P0-3 实现缺陷反馈）**：`IsInputFormatSupported`/`IsOutputFormatSupported`
> 由 Windows 引擎在**格式协商阶段**调用，**早于 `LockForProcess`**——此时 `pipeline_context` 仍为
> `PipelineContext::new()`（全零默认值）。因此本方法**禁止依赖 `pipeline_context` 做等值比较**
> （如 `fmt.channels == ctx.input_channels`——请求的真实格式 vs 全零永远不等 → 拒绝所有格式，
> APO 无法协商）。正确做法是对请求格式做**独立属性检查**（下述浮点格式 + 采样率范围 + 通道数范围），
> 而这些属性在协商时即已确定、与锁定后上下文无关。

```rust
fn IsInputFormatSupported(&self, p_opposite, p_requested, pp_supported) -> HRESULT {
    if pp_supported.is_null() { return E_POINTER; }
    unsafe { *pp_supported = std::ptr::null_mut(); }
    if p_requested.is_null() { return E_INVALIDARG; }

    // 浮点格式检查
    if !is_float_format(p_requested) {
        return APOERR_FORMAT_NOT_SUPPORTED;
    }

    let format = match extract_format(p_requested) {
        Ok(f) => f,
        Err(_) => return APOERR_FORMAT_NOT_SUPPORTED,
    };

    // 采样率范围：44.1kHz ~ 192kHz
    if format.sample_rate < 44100 || format.sample_rate > 192000 {
        return APOERR_FORMAT_NOT_SUPPORTED;
    }

    // 通道数范围：1 ~ 8
    if format.channels == 0 || format.channels > 8 {
        return APOERR_FORMAT_NOT_SUPPORTED;
    }

    S_OK
}

fn IsOutputFormatSupported(&self, p_opposite, p_requested, pp_supported) -> HRESULT {
    // 输出格式与输入格式使用相同的检查逻辑
    self.IsInputFormatSupported(p_opposite, p_requested, pp_supported)
}
```

---

#### 7.1.17 `GetInputChannelCount`

```rust
fn GetInputChannelCount(&self, p_count: *mut u32) -> HRESULT {
    if p_count.is_null() { return E_POINTER; }
    if self.state_cell.current() != ApoState::Locked {
        return APOERR_NOT_INITIALIZED;
    }
    let inner = self.mutex.lock().unwrap();
    unsafe { *p_count = inner.pipeline_context.input_channels };
    S_OK
}
```

---

#### 7.1.18 `hot_reload`（由 watcher 回调，后台线程）

```rust
fn hot_reload(&self) {
    // 1. 若已有过渡在途或正在加载——阻塞式退出（R2/v6.9，EAPO 信号量对齐）。
    //    过渡期间不构建新链；过渡完成后由 APOProcess 触发下一次 hot_reload。
    {
        let lock = self.mutex.lock().unwrap();
        if lock.transition.is_some() || lock.reloading {
            return;
        }
    }

    // 2. 锁外解析配置（不持有 mutex）。此刻保证无过渡在途。
    let current_ctx = { self.mutex.lock().unwrap().pipeline_context.clone() };
    let filters = match self.config_loader.parse_file(&self.config_path, &mut ctx) {
        Ok(f) => f,
        Err(_) => {
            self.logger.log(LogLevel::Error, "hot_reload: config parse failed");
            return;
        }
    };
    let mut new_chain = Chain::new(current_ctx.max_frame_count);
    for f in filters {
        if let Err(_) = new_chain.add_filter(f) {
            self.logger.log(LogLevel::Error, "hot_reload: add_filter failed");
            return;
        }
    }

    // 3. 短锁内交换（仅交换指针）
    let mut inner = self.mutex.lock().unwrap();
    if inner.transition.is_some() {
        // 竞态兜底：解析期间可能已有新过渡启动，退回阻塞。
        inner.pending_reload = true;
        return;
    }

    let old = std::mem::replace(&mut inner.current_chain, Box::new(new_chain));
    // 旧链进 outgoing（过渡窗口内由 RT 双处理）；
    // 过渡完成后 APOProcess 将其移入 retired_chain（R1），控制线程统一析构。
    inner.outgoing_chain = Some(old);
    inner.transition = Some(SmoothingProvider::new(
        default_smoothing_length(inner.pipeline_context.sample_rate),
    ));
    inner.pending_reload = false;
    inner.reloading = false;
    // 当前过渡在途；下次变更由 APOProcess 过渡完成后触发（reloading 防覆盖）。
}
```

---

#### 7.1.19 编译期断言

```rust
const _: () = {
    // 三个特殊 GUID 互不相同
    assert!(APOGUID_NOKEY.data1 != APOGUID_NOVALUE.data1
        || APOGUID_NOKEY.data4[7] != APOGUID_NOVALUE.data4[7]);
    assert!(GUID_NULL.data1 != APOGUID_NOKEY.data1
        || GUID_NULL.data4[7] != APOGUID_NOKEY.data4[7]);
    assert!(GUID_NULL.data1 != APOGUID_NOVALUE.data1
        || GUID_NULL.data4[7] != APOGUID_NOVALUE.data4[7]);
};
```

---

### 7.2 `object/child.rs` — 子 APO COM 生命周期管理

**职责**：管理子 APO 的 COM 接口持有和方法委托。

**引用来源**：
- `crate::sys::com::prelude::*`
- `crate::sys::com::apo_interfaces::*`
- `crate::sys::com::apo_types::*`
- `windows::core::{GUID, IUnknown}`
- `windows::Win32::System::Com::{CoCreateInstance, CLSCTX_ALL}`

**结构体**：

```rust
/// 子 APO COM 对象持有者。
///
/// 持有类型化的 COM 接口引用，通过 #[interface] trait 方法调用，
/// 无需手动维护 vtable 索引。Drop 时自动释放三个 COM 接口引用。
pub struct ChildApo {
    iapo: IAudioProcessingObject,
    iapo_rt: IAudioProcessingObjectRT,
    iapo_cfg: IAudioProcessingObjectConfiguration,
}

unsafe impl Send for ChildApo {}
unsafe impl Sync for ChildApo {}
```

**方法**：

```rust
impl ChildApo {
    /// 创建子 APO 实例。
    ///
    /// CoCreateInstance → IUnknown → QI 三个接口 → drop IUnknown。
    ///
    /// # Safety
    /// - COM 必须已初始化（CoInitializeEx）
    /// - clsid 必须指向有效的 APO CLSID
    pub unsafe fn create(clsid: &GUID) -> Result<Self, HRESULT>;

    /// 是否有效（三个接口均非 null）。
    pub fn is_valid(&self) -> bool;

    // ── IAudioProcessingObject 委托 ──────────────────────────────────
    pub fn get_latency(&self) -> REFERENCE_TIME;
    pub fn reset(&self) -> HRESULT;
    pub unsafe fn get_registration_properties(&self, pp_props: *mut *mut APO_REG_PROPERTIES) -> HRESULT;
    pub unsafe fn initialize(&self, cb_data_size: u32, pby_data: *mut u8) -> HRESULT;
    pub unsafe fn is_input_format_supported(&self, p_opposite: *mut IAudioMediaType, p_requested: *mut IAudioMediaType, pp_supported: *mut *mut IAudioMediaType) -> HRESULT;
    pub unsafe fn is_output_format_supported(&self, p_opposite: *mut IAudioMediaType, p_requested: *mut IAudioMediaType, pp_supported: *mut *mut IAudioMediaType) -> HRESULT;
    pub fn get_input_channel_count(&self, p_count: *mut u32) -> HRESULT;

    // ── IAudioProcessingObjectRT 委托 ────────────────────────────────
    pub fn calc_input_frames(&self, output_frames: u32) -> u32;
    pub fn calc_output_frames(&self, input_frames: u32) -> u32;

    // ── IAudioProcessingObjectConfiguration 委托 ─────────────────────
    pub unsafe fn lock_for_process(&self, num_input: u32, pp_inputs: *mut *mut APO_CONNECTION_DESCRIPTOR, num_output: u32, pp_outputs: *mut *mut APO_CONNECTION_DESCRIPTOR) -> HRESULT;
    pub fn unlock_for_process(&self) -> HRESULT;
}

// Drop 自动管理：三个 COM 接口引用各自 Release，无需手动管理。
```

---

### 7.3 `object/factory.rs` — ApoFactory

**职责**：`IClassFactory` 实现 + `LOCK_COUNT` 管理。

**引用来源**：
- `crate::sys::com::prelude::*`
- `crate::sys::com::apo_interfaces::*`
- `crate::object::apo::ApoObject`
- `crate::object::vx_reg_props::is_vxapo_clsid`

---

#### LOCK_COUNT

```rust
static LOCK_COUNT: AtomicU32 = AtomicU32::new(0);

pub fn lock_increment() -> u32;
pub fn lock_decrement() -> u32;
pub fn lock_count() -> u32;
pub fn lock_is_zero() -> bool;

#[cfg(test)]
pub fn lock_reset_for_test();
```

---

#### ClassFactory

```rust
#[implement(IClassFactory)]
pub struct ClassFactory {
    target_clsid: GUID,
}

impl IClassFactory_Impl for ClassFactory_Impl {
    fn CreateInstance(
        &self,
        punkouter: Ref<'_, IUnknown>,
        riid: *const GUID,
        ppvobject: *mut *mut c_void,
    ) -> windows_core::Result<()> {
        // 1. 输出指针初始化为 null
        // 2. 参数校验（riid / ppvobject null 检查）
        // 3. 聚合检查（punkouter 非空 → CLASS_E_NOAGGREGATION）
        // 4. 创建 ApoObject（传入 self.target_clsid）
        // 5. 转为 IUnknown 并 QI 获取请求的接口
        // 6. 释放临时引用
    }

    fn LockServer(&self, flock: BOOL) -> windows_core::Result<()> {
        if flock.as_bool() { lock_increment(); }
        else { lock_decrement(); }
        Ok(())
    }
}
```

---

#### 工厂创建辅助

```rust
/// 根据 CLSID 创建 ClassFactory。不匹配则返回 None。
pub fn create_factory(clsid: &GUID) -> Option<IClassFactory> {
    if is_vxapo_clsid(clsid) {
        Some(ClassFactory { target_clsid: *clsid }.into())
    } else {
        None
    }
}
```

---

### 7.4 `object/ref_count.rs`

**职责**：`INST_COUNT` 原子计数，追踪当前存活的 APO COM 对象实例数。

**引用来源**：`core`（`AtomicU32`、`Ordering`）

**公开 API**：

```rust
static INST_COUNT: AtomicU32 = AtomicU32::new(0);

/// 创建实例时递增。调用时机：ClassFactory::CreateInstance / ApoObject::new。
/// 返回递增后的值（用于调试日志）。
pub fn increment() -> u32;

/// 实例析构时递减。调用时机：ApoObject Drop。
/// 返回递减后的值（用于调试日志）。
pub fn decrement() -> u32;

/// 读取当前活跃实例数。
pub fn get() -> u32;

/// 检查是否没有活跃实例。
pub fn is_zero() -> bool;
```

**`DllCanUnloadNow` 判定**：`inst_count::is_zero() && factory::lock_is_zero()` → `S_OK`

---

### 7.5 `object/vx_reg_props.rs`

**职责**：VxAPO 自身的 CLSID 定义、APO 注册属性、COM 注册条目与路径生成。

**引用来源**：
- `crate::sys::com::prelude::GUID`
- `crate::sys::com::apo_types::{APO_FLAG, APO_REG_PROPERTIES, IID_IAPO}`

**导出给**：`object/apo.rs`（`props_for_clsid`）、`object/factory.rs`（`is_vxapo_clsid`）、`object/dll_exports.rs`（CLSID 路由 + 注册条目）

---

#### CLSID 常量（2 个，v7.5 正式 GUID）

> **v7.5 变更（用户确定正式 GUID）**：CLSID 由占位值替换为正式生成值——
> `PRE_MIX = 41C34613-D391-459D-A039-72B2B15A1A1D`、`POST_MIX = B4A97313-ABC0-45ED-9C33-428B20D39428`。
> 变更涉及面：`vxapo.def` 导出、`sys/com/apo_types` 的 APOInitSystemEffects 路由、
> `install/device/slots` 设备绑定、`object/vx_reg_props`（本文件）——同步更新。

```rust
pub const CLSID_VXAPO_PRE_MIX: GUID = GUID::from_values(
    0x41C34613, 0xD391, 0x459D,
    [0xA0, 0x39, 0x72, 0xB2, 0xB1, 0x5A, 0x1A, 0x1D],
);

pub const CLSID_VXAPO_POST_MIX: GUID = GUID::from_values(
    0xB4A97313, 0xABC0, 0x45ED,
    [0x9C, 0x33, 0x42, 0x8B, 0x20, 0xD3, 0x94, 0x28],
);
```

---

#### APO 名称、版权、标志位

```rust
const APO_NAME: &str = "VxAPO";
const APO_COPYRIGHT: &str = "VxAPO Project";

/// APO 标志位：采样率必须匹配 | 位深必须匹配 | 支持 inplace
const APO_FLAGS: APO_FLAG = APO_FLAG(
    APO_FLAG::FRAMESPERSECOND_MUST_MATCH.0
    | APO_FLAG::BITSPERSAMPLE_MUST_MATCH.0
    | APO_FLAG::INPLACE.0,
);
```

---

#### 注册属性实例（2 个）

```rust
pub static REG_PROPS_PRE_MIX: APO_REG_PROPERTIES = APO_REG_PROPERTIES {
    clsid: CLSID_VXAPO_PRE_MIX,
    flags: APO_FLAGS,
    sz_friendly_name: str_to_u16_256(APO_NAME),
    sz_copyright_info: str_to_u16_256(APO_COPYRIGHT),
    major_version: 1,
    minor_version: 0,
    min_input_connections: 1,
    max_input_connections: 1,
    min_output_connections: 1,
    max_output_connections: 1,
    max_instances: 1,
    num_apo_interfaces: 3,
    iid_apo_interface_list: [IID_IAPO],
};

/// PostMix（与 PreMix 共享名称和标志，仅 CLSID 不同）。
pub static REG_PROPS_POST_MIX: APO_REG_PROPERTIES = APO_REG_PROPERTIES {
    clsid: CLSID_VXAPO_POST_MIX,
    ..REG_PROPS_PRE_MIX
};
```

---

#### UTF-16 编码辅助（编译期）

```rust
/// 编译期将 &str 转为 [u16; 256]（UTF-16，null 填充）。
const fn str_to_u16_256(s: &str) -> [u16; 256];
```

---

#### 查询辅助

```rust
/// 根据 CLSID 查找对应的注册属性。
pub fn props_for_clsid(clsid: &GUID) -> Option<&'static APO_REG_PROPERTIES>;

/// 判断是否为 VxAPO 的 CLSID（PreMix 或 PostMix）。
pub fn is_vxapo_clsid(clsid: &GUID) -> bool;

/// 返回所有支持的 CLSID 列表。
pub fn supported_clsids() -> &'static [GUID];
```

---

#### COM 注册条目

**注册表路径常量**：

```rust
const CLSID_ROOT: &str = r"CLSID";
const INPROC_SERVER: &str = "InprocServer32";
const THREADING_MODEL_VALUE: &str = "ThreadingModel";
const THREADING_MODEL_BOTH: &str = "Both";
```

**注册表结构**：

```text
HKCR\CLSID\{GUID}\InprocServer32
    (Default) = "C:\...\vxapo.dll"
    ThreadingModel = "Both"
```

**`ClsidEntry` 结构体**：

```rust
/// 单个 CLSID 的注册信息。
#[derive(Debug)]
pub struct ClsidEntry {
    /// CLSID GUID
    pub clsid: GUID,
    /// 格式化后的 CLSID 字符串 {xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}
    pub clsid_str: String,
}

impl ClsidEntry {
    pub fn new(clsid: GUID) -> Self {
        Self {
            clsid,
            clsid_str: format!("{clsid}"),  // GUID 实现了 Display
        }
    }

    /// HKCR\CLSID\{GUID}
    pub fn clsid_key_path(&self) -> String {
        format!("{CLSID_ROOT}\\{}", self.clsid_str)
    }

    /// HKCR\CLSID\{GUID}\InprocServer32
    pub fn inproc_server_path(&self) -> String {
        format!("{CLSID_ROOT}\\{}\\{INPROC_SERVER}", self.clsid_str)
    }

    /// 注册所需的值列表。
    pub fn registration_entries(&self, dll_path: &str) -> Vec<(&'static str, String, String)> {
        vec![
            ("Default", String::new(), dll_path.to_owned()),
            ("ThreadingModel", THREADING_MODEL_VALUE.to_owned(), THREADING_MODEL_BOTH.to_owned()),
        ]
    }
}
```

**注册/注销顺序**：

```rust
/// 注册顺序：PostMix → PreMix（Note 29：先注册 PostMix，失败时回滚）。
pub fn registration_order() -> Vec<ClsidEntry>;

/// 注销顺序：PreMix → PostMix（与注册相反，Note 30）。
pub fn unregistration_order() -> Vec<ClsidEntry>;
```

---

#### 编译期断言

```rust
const _: () = {
    // APO_FLAGS 组合值：FRAMESPERSECOND(0x04) | BITSPERSAMPLE(0x08) | INPLACE(0x01) = 0x0D
    assert!(APO_FLAGS.0 == 0x0000_000D);
    assert!(REG_PROPS_PRE_MIX.sz_friendly_name[0] != 0);
    assert!(REG_PROPS_PRE_MIX.flags.0 == REG_PROPS_POST_MIX.flags.0);
    assert!(REG_PROPS_PRE_MIX.max_input_connections == REG_PROPS_POST_MIX.max_input_connections);
    // CLSID 互不相同
    assert!(CLSID_VXAPO_PRE_MIX.data1 != CLSID_VXAPO_POST_MIX.data1);
};
```

---

### 7.6 `object/dll_exports.rs`

**职责**：DLL 导出函数。

**引用来源**：
- `crate::object::factory`
- `crate::object::ref_count`
- `crate::object::vx_reg_props`
- `crate::sys::com::prelude::*`
- `crate::sys::registry`（CLSID 键写入，HKCR 根键）
- `crate::telemetry`

---

#### 全局状态

```rust
/// DLL 模块句柄（DllMain 中保存，Note 59）。
/// 用于 GetModuleFileNameW 获取 DLL 路径，供注册/注销使用。
static MODULE_HANDLE: AtomicPtr<std::ffi::c_void> = AtomicPtr::new(std::ptr::null_mut());
```

---

#### `DllMain`（Note 59）

```rust
#[no_mangle]
pub unsafe extern "system" fn DllMain(
    h_module: HMODULE,
    ul_reason_for_call: u32,
    _lp_reserved: *mut c_void,
) -> BOOL;
```

**DllMain 约束**（Note 59）：
- `DLL_PROCESS_ATTACH`：仅保存 `MODULE_HANDLE`（`AtomicPtr` 原子写入），不做任何其他操作
- `DLL_PROCESS_DETACH`：当前无需特殊清理
- `DLL_THREAD_ATTACH` / `DLL_THREAD_DETACH`：忽略
- 始终返回 `TRUE`

**禁止在 `DLL_PROCESS_ATTACH` 中**：
- 初始化 COM（`CoInitializeEx`）
- 创建线程（`std::thread::spawn`）
- 触发全局 `static` 的复杂初始化（`once_cell::Lazy::force()` 等）
- 使用 `#[ctor]` 宏标注的初始化函数
- 调用任何 `windows-rs` 的 COM 初始化宏

所有 COM 初始化与 APO 对象构造延迟到 `DllGetClassObject` 或 `CreateInstance` 被首次调用时执行。

---

#### `DllGetClassObject`（Note 4）

```rust
#[no_mangle]
pub unsafe extern "system" fn DllGetClassObject(
    rclsid: *const GUID,
    riid: *const GUID,
    ppv: *mut *mut c_void,
) -> HRESULT;
```

**流程**：
1. 参数校验（null 检查）
2. 输出指针初始化为 null
3. 读取 `rclsid`，调用 `factory::create_factory(&clsid)`
4. 不匹配返回 `CLASS_E_CLASSNOTAVAILABLE`
5. QI 获取 `riid` 请求的接口
6. 释放工厂临时引用

---

#### `DllCanUnloadNow`（Note 2）

```rust
#[no_mangle]
pub extern "system" fn DllCanUnloadNow() -> HRESULT {
    if inst_count::is_zero() && factory::lock_is_zero() {
        S_OK
    } else {
        S_FALSE
    }
}
```

---

#### `DllRegisterServer`（Note 29）

```rust
#[no_mangle]
pub extern "system" fn DllRegisterServer() -> HRESULT;
```

**职责边界（v7.1 澄清）**：`regsvr32` 调用本函数时**无设备参数**，因此本函数**只能**完成
**全局 COM 类注册**——使 DLL 可被 `CoCreateInstance` 实例化（`object/factory.rs`）。
**不包含** APO 设备挂载 / FxProperties 绑定——那属于 `install_endpoint`（`install 5.5.2`），
由 `vxapo-cli install -d <device>` 触发。二者分层，`regsvr32` 不绑定设备。

**流程**：
1. 获取 DLL 路径（`GetModuleFileNameW` + `MODULE_HANDLE`；失败 → `SELFREG_E_CLASS`）
2. 按 `vx_reg_props::registration_order()`（PostMix → PreMix）逐个注册 CLSID
3. 每个 CLSID：
   - 创建/打开 `HKCR\CLSID\{GUID}\InprocServer32`（经 `sys/registry`）
   - 写 `(Default)` = DLL 路径，`ThreadingModel` = `"Both"`
4. **幂等性**：键已存在时覆盖写入（重复 `regsvr32` 安全）
5. 任一步失败 → 按已注册条目的逆序回滚 → 返回 `SELFREG_E_CLASS`
6. 全部成功 → `S_OK`

**禁止**：
- 触碰 `MMDevices` / `FxProperties`（设备绑定属 `install_endpoint`）
- 在函数内创建线程 / 初始化 COM（`CoInitializeEx` 由 regsvr32 宿主进程负责）

---

#### `DllUnregisterServer`（Note 30）

```rust
#[no_mangle]
pub extern "system" fn DllUnregisterServer() -> HRESULT;
```

**职责边界**：仅删除 `DllRegisterServer` 创建的全局 COM 类键。**不触碰设备关联**。

**流程**：
1. 按 `vx_reg_props::unregistration_order()`（PreMix → PostMix，与注册相反）
2. 每个 CLSID：先删 `InprocServer32` 子键，再删 `CLSID\{GUID}` 父键
3. **幂等性**：键不存在视为成功（重复 `regsvr32 /u` 安全）
4. 尽力清理：某条目失败也继续后续
5. 全部完成 → `S_OK`

---

#### 内部辅助

```rust
/// 获取当前 DLL 文件路径。使用 GetModuleFileNameW + MODULE_HANDLE。
fn get_dll_path() -> Option<String>;

/// 注册单个 CLSID 的 COM 类。
///
/// 写入 `HKCR\CLSID\{GUID}\InprocServer32`（创建/打开 + 写值，经 `sys/registry`）。
/// 失败返回具体 HRESULT，由 `DllRegisterServer` 统一回滚。
fn register_com_class(entry: &vx_reg_props::ClsidEntry, dll_path: &str) -> Result<(), HRESULT>;

/// 注销单个 CLSID 的 COM 类。
///
/// 先删 `InprocServer32` 子键，再删 `CLSID\{GUID}` 父键；键不存在视为成功（幂等）。
fn unregister_com_class(entry: &vx_reg_props::ClsidEntry) -> Result<(), HRESULT>;

/// 将字符串转为注册表所需的 null-terminated UTF-16 字节数组。
fn to_registry_bytes(s: &str) -> Vec<u8>;
```