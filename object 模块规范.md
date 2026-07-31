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
| `object/apo.rs` | `sys/com/`（全部）、`object/child.rs`、`object/vx_reg_props.rs`、`object/ref_count.rs`、`object/factory.rs`、`pipeline/`、`config/`、`install/audiodg`、`telemetry/logger`、`utils/` | — |
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
    pub transition: Option<SmoothingProvider>,
    pub pipeline_context: PipelineContext,
    pub temp_buffers: Vec<Vec<f32>>,
    pub temp_buffer_old: Vec<f32>,
    pub temp_buffer_new: Vec<f32>,
    pub pending_reload: bool,
}
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
#[repr(u8)]
pub enum ApoState {
    Created     = 0,
    Initialized = 1,
    Locked      = 2,
}

pub struct StateCell {
    state: AtomicU8,
}

impl StateCell {
    pub fn transition(&self, from: ApoState, to: ApoState) -> Result<()> {
        self.state.compare_exchange(
            from as u8, to as u8,
            Ordering::AcqRel, Ordering::Acquire,
        ).map(|_| ())
         .map_err(|_| VxApoError::State("非法状态转换".into()))
    }
    pub fn current(&self) -> ApoState;
}
```

**状态转换规则**：
- `Initialize`：要求 `Created` → `Initialized`
- `LockForProcess`：要求 `Initialized` → `Locked`
- `APOProcess`：要求 `Locked`（不改变状态）
- `UnlockForProcess`：要求 `Locked` → `Initialized`

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

#### 7.1.8 `Initialize`

```rust
fn Initialize(&self, cb_data_size: u32, pby_data: *mut u8) -> HRESULT {
    // 1. 获取 mutex.lock()
    // 2. state_cell.transition(Created, Initialized)
    // 3. 解析 APOInitSystemEffects（pby_data），提取子 APO CLSID
    // 4. 如有子 APO CLSID，创建 ChildApo
    // 5. 返回 S_OK
}
```

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
    inner.transition = None;
    inner.pipeline_context = ctx;
    inner.temp_buffers = vec![vec![0.0f32; ctx.max_frame_count]; max_ch];
    inner.temp_buffer_old = temp_buffer_old;
    inner.temp_buffer_new = temp_buffer_new;
    inner.pending_reload = false;

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

        // 过渡完成
        if factor >= 1.0 {
            inner.outgoing_chain = None;
            inner.transition = None;
            // 过渡完成时检查是否有排队的热重载请求
            if inner.pending_reload {
                inner.pending_reload = false;
                drop(inner); // 释放锁，避免递归死锁
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
    // 1. 锁外解析配置（不持有 mutex）
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

    // 2. 短锁内交换（仅交换指针）
    let mut inner = self.mutex.lock().unwrap();

    if inner.transition.is_some() {
        inner.pending_reload = true;
        self.logger.log(LogLevel::Info, "hot_reload: transition in progress, queued");
        return;
    }

    let old = std::mem::replace(&mut inner.current_chain, Box::new(new_chain));
    inner.outgoing_chain = Some(old);
    inner.transition = Some(SmoothingProvider::new(
        default_smoothing_length(inner.pipeline_context.sample_rate),
    ));
    // pending_reload 在过渡完成后由 APOProcess（RT 线程）触发。
    // 此设计确保过渡完全结束后才执行重载，避免多个过渡混合。
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

#### CLSID 常量（2 个）

```rust
pub const CLSID_VXAPO_PRE_MIX: GUID = GUID::from_values(
    0xA1B2C3D4, 0x1234, 0x5678,
    [0x9A, 0xBC, 0xDE, 0xF0, 0x12, 0x34, 0x56, 0x78],
);

pub const CLSID_VXAPO_POST_MIX: GUID = GUID::from_values(
    0xD4C3B2A1, 0x4321, 0x8765,
    [0x9A, 0xBC, 0xDE, 0xF0, 0x12, 0x34, 0x56, 0x79],
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

**流程**：
1. 获取 DLL 路径（`GetModuleFileNameW` + `MODULE_HANDLE`）
2. 按注册顺序（PostMix → PreMix）逐个注册 CLSID
3. 每个 CLSID 写入 `HKCR\CLSID\{GUID}\InprocServer32`（路径 + `ThreadingModel = "Both"`）
4. 注册失败时回滚已注册条目

---

#### `DllUnregisterServer`（Note 30）

```rust
#[no_mangle]
pub extern "system" fn DllUnregisterServer() -> HRESULT;
```

**流程**：
1. 按注销顺序（PreMix → PostMix，与注册相反）
2. 先删 `InprocServer32` 子键，再删 `CLSID` 父键
3. 尽力清理，即使某条目注销失败也继续

---

#### 内部辅助

```rust
/// 获取当前 DLL 文件路径。使用 GetModuleFileNameW + MODULE_HANDLE。
fn get_dll_path() -> Option<String>;

/// 注册单个 CLSID 的 COM 类。
fn register_com_class(entry: &vx_reg_props::ClsidEntry, dll_path: &str) -> Result<(), HRESULT>;

/// 注销单个 CLSID 的 COM 类。
fn unregister_com_class(entry: &vx_reg_props::ClsidEntry) -> Result<(), HRESULT>;

/// 将字符串转为注册表所需的 null-terminated UTF-16 字节数组。
fn to_registry_bytes(s: &str) -> Vec<u8>;
```