## 四、`pipeline/` 模块规范（最终版）

**边界**：不知道 `install/`、`config/`、`object/`。只处理音频数据，不知道谁在调用。

**允许依赖**：`sys/`、`utils/`

**禁止依赖**：`install/`、`config/`、`object/`

---

### 数据流架构

**正常模式**（APOProcess → process_audio）：

```
Windows 交织缓冲区（&[f32]，来自 APO_CONNECTION_PROPERTY.p_buffer）
  │
  ▼
pipeline/process.rs — evaluate_buffer → 提取输入/输出切片
  │
  ▼
pipeline/channel.rs — deinterleave_into（交织 → 去交织，零分配）
  │
  ▼
pipeline/chain.rs — 逐 Filter 调用（去交织空间）
  │   └── filter.process(&mut [Vec<f32>], frame_count)
  │
  ▼
pipeline/channel.rs — interleave_from（去交织 → 交织，零分配）
  │
  ▼
Windows 交织缓冲区（写回 APO_CONNECTION_PROPERTY.p_buffer）
```

**过渡模式**（APOProcess → process_chain_interleaved × 2 → mix_buffers）：

```
Windows 交织输入（共享）
  │
  ├──▶ deinterleave_into ──▶ old_chain.process ──▶ interleave_from ──▶ temp_buffer_old
  │
  └──▶ deinterleave_into ──▶ new_chain.process ──▶ interleave_from ──▶ temp_buffer_new
                                                                        │
                                                                        ▼
                              mix_buffers(temp_old, temp_new, factor) ──▶ Windows 输出
```

> 过渡模式下两次 `process_chain_interleaved` 串行执行，共享同一组去交织工作缓冲区（`temp_buffers`），各自写入不同的交织输出缓冲区。`transition.rs` 的 `SmoothingProvider` 仅提供混合因子，不管理链状态。链状态由 `object/apo.rs` 的 `ApoObjectInner` 管理（对象切换）。

---

### 完整模块树

```
pipeline/
├── context.rs          # PipelineContext（静态格式信息）
├── buffer.rs           # BufferInfo + 缓冲区状态判定
├── format.rs           # 从 IAudioMediaType 提取 WAVEFORMATEX
├── chain.rs            # Filter 链执行 + 延迟累计
├── process.rs          # APOProcess 调度入口 + 桥接函数 + 错误策略
├── channel.rs          # 通道映射 + 去交织/交织 + 默认通道掩码
├── realtime.rs         # 实时安全基础设施模块入口
├── realtime/
│   ├── ring.rs         # SPSC 无锁环形缓冲区
│   └── contract.rs     # RT 上下文断言（RtGuard / assert_in_rt）
├── dsp.rs              # DSP 算法模块入口
└── dsp/
    ├── filter.rs       # Filter trait + FilterCreateResult + FilterFactory + DspContext + ConfigLoader
    ├── factory.rs      # FilterRegistry + 工厂注册 + 工厂遍历匹配
    ├── transition.rs   # SmoothingProvider + raised_cosine + mix_buffers
    ├── biquad.rs       # 双二阶滤波器
    ├── peq.rs          # 参量均衡器（级联 biquad）
    ├── hp_lp.rs        # 高通/低通
    ├── gain.rs         # 增益（含内部平滑插值）
    ├── delay.rs        # 延迟线（环形缓冲实现）
    ├── copy.rs         # 通道复制/混音
    ├── graphic_eq.rs   # 图形均衡器（多段）
    ├── convolution.rs  # 卷积（FFT 骨架）
    ├── vst.rs          # ⚠️ feature gate = ["vst"]
    └── loudness.rs     # ISO 226 等响曲线
```

---

### 引用约束总表

| 模块 | 可依赖 | 不可依赖 |
|------|--------|----------|
| `pipeline/context.rs` | 无 | `install/`、`config/`、`object/` |
| `pipeline/buffer.rs` | `sys/com/apo_types` | `install/`、`config/`、`object/` |
| `pipeline/format.rs` | `sys/com/apo_interfaces`、`sys/com/apo_types` | `install/`、`config/`、`object/` |
| `pipeline/channel.rs` | 无 | `install/`、`config/`、`object/` |
| `pipeline/chain.rs` | `dsp/filter`、`utils/` | `install/`、`config/`、`object/`、`dsp/transition` |
| `pipeline/process.rs` | `context`、`chain`、`buffer`、`channel`、`dsp/filter`、`dsp/transition`、`sys/com/apo_types`、`utils/` | `install/`、`config/`、`object/` |
| `pipeline/realtime/contract.rs` | `core` | 其他 |
| `pipeline/realtime/ring.rs` | `core` | 其他 |
| `pipeline/dsp/filter.rs` | `utils/` | `install/`、`config/`、`object/` |
| `pipeline/dsp/factory.rs` | `dsp/filter`、`utils/` | `install/`、`config/`、`object/` |
| `pipeline/dsp/transition.rs` | 无 | `install/`、`config/`、`object/` |
| `pipeline/dsp/*.rs`（具体 Filter） | `dsp/filter`、`dsp/biquad`（如需要）、`utils/` | `install/`、`config/`、`object/` |

> **关键设计决策**：Chain 不拥有缓冲区。缓冲区由 `object/apo.rs` 的 `ApoObjectInner` 预分配并持有，Chain 仅接受外部传入的去交织缓冲区引用执行处理。此设计保证：
> 1. **缓冲区稳定性**——预分配于 `LockForProcess`，生命周期由 `ApoObjectInner` 管理
> 2. **process 无阻塞**——Chain.process 为纯计算（逐 Filter 调用），无锁、无分配、无 I/O
> 3. **过渡模式简洁**——两个 Chain 共享同一组去交织工作缓冲区（串行使用），各自产出独立的交织输出后由 `transition.rs` 混合
> 4. **热重载原子性**——Chain 替换仅涉及 `Box<Chain>` 指针交换（在 mutex 内，亚微秒级），缓冲区不受影响

---

### 4.1 `pipeline/context.rs`

**职责**：运行时上下文。存放 `LockForProcess` 时确定的静态格式信息，`APOProcess` 期间只读。

**引用来源**：无外部依赖

**导出给**：`pipeline/process.rs`、`pipeline/chain.rs`、`object/apo.rs`

**字段**：

```rust
pub struct PipelineContext {
    pub sample_rate: u32,
    pub input_channels: u32,
    pub output_channels: u32,
    pub channel_mask: u32,
    pub max_frame_count: usize,
}

impl PipelineContext {
    pub fn new() -> Self;
    pub fn bytes_per_frame(&self) -> usize;
}
```

**禁止**：不包含任何 Filter 引用，不包含 DspContext 构造逻辑

---

### 4.2 `pipeline/buffer.rs`

**职责**：缓冲区描述与状态判定。封装 `APO_CONNECTION_PROPERTY`，提供结构化缓冲区访问；提供缓冲区状态判定和操作工具。

**引用来源**：
- `crate::sys::com::apo_types::APO_BUFFER_FLAGS`

**导出给**：`pipeline/process.rs`、`object/apo.rs`

---

#### BufferInfo（APO_CONNECTION_PROPERTY 安全封装）

```rust
/// 缓冲区信息——封装 APO_CONNECTION_PROPERTY 的字段。
///
/// 提供安全的切片访问。内部数据为交织格式（&[f32]）。
pub struct BufferInfo {
    pub ptr: *mut f32,
    pub valid_frames: usize,
    pub flags: APO_BUFFER_FLAGS,
    pub channels: usize,
}

impl BufferInfo {
    pub fn new(ptr: *mut f32, valid_frames: usize, flags: APO_BUFFER_FLAGS, channels: usize) -> Self;
    pub fn from_prop(prop: &APO_CONNECTION_PROPERTY, channels: usize) -> Self;
    pub fn from_prop_mut(prop: &mut APO_CONNECTION_PROPERTY, channels: usize) -> Self;

    pub fn is_valid(&self) -> bool;
    pub fn is_silent(&self) -> bool;
    pub fn total_samples(&self) -> usize;   // valid_frames * channels
    pub fn bytes(&self) -> usize;           // total_samples * size_of::<f32>()
    pub fn as_slice(&self) -> &[f32];       // 交织格式连续切片
    pub fn as_slice_mut(&mut self) -> &mut [f32];
    pub fn zero(&mut self);
}
```

---

#### 缓冲区状态判定（Note 11）

```rust
const SILENCE_THRESHOLD: f32 = 1e-10;  // -200 dBFS 以下

/// 缓冲区处理动作。
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum BufferAction {
    /// 正常处理——缓冲区包含有效音频数据，或静音但允许处理。
    Process,
    /// 跳过——标志不合法（Invalid），不处理，输出清零。
    Skip,
    /// 静音——标志为 Silent 且不允许静音缓冲区修改，输出清零。
    Silent,
}

/// 根据输入标志和 allowSilentBuffer 确定处理动作。
/// 返回 (BufferAction, output_flags)。
pub fn evaluate_buffer(flags: APO_BUFFER_FLAGS, allow_silent_buffer: bool) -> (BufferAction, APO_BUFFER_FLAGS);
```

**evaluate_buffer 决策逻辑**：

| 输入 flags | allow_silent_buffer | BufferAction | 备注 |
|-----------|--------------------|--------------|----|
| Invalid | 任意 | Skip | 不处理 |
| Silent | false | Silent | 强制静音输出 |
| Silent | true | Process | 传递给 DSP（可能确实是静音） |
| Valid | 任意 | Process | 正常处理 |

---

#### 静音检测（去交织空间，实时路径用）

```rust
/// 检查去交织平面缓冲区中所有采样是否为静音。
/// 通道数 1/2/6/8 时编译期特化展开。
pub fn is_silent(samples: &[Vec<f32>], frame_count: usize) -> bool;
```

---

#### 缓冲区操作工具（去交织空间，实时路径用）

```rust
/// 将去交织平面缓冲区所有通道清零。
pub fn zero_buffers(buffers: &mut [Vec<f32>], frame_count: usize);

/// 将单个通道清零。
pub fn zero_channel(channel: &mut [f32], frame_count: usize);

/// 将源缓冲区内容复制到目标缓冲区（替代 Vec::clone，避免隐式堆分配）。
pub fn copy_buffers(src: &[Vec<f32>], dst: &mut [Vec<f32>], frame_count: usize);
```

---

#### 调试摘要（非实时路径）

```rust
#[derive(Debug, Clone)]
pub struct BufferSummary {
    pub channels: usize,
    pub frame_count: usize,
    pub is_silent: bool,
    pub peak_level: f32,
    pub rms_level: f32,
}

pub fn summarize(buffers: &[Vec<f32>], frame_count: usize) -> BufferSummary;
```

---

### 4.3 `pipeline/format.rs`

**职责**：从 `IAudioMediaType` 提取 WAVEFORMATEX 信息。

**引用来源**：
- `crate::sys::com::apo_interfaces::IAudioMediaType`
- `crate::sys::com::apo_types::*`

**导出给**：`object/apo.rs`

**公开 API**：

```rust
pub struct AudioFormat {
    pub sample_rate: u32,
    pub channels: u32,
    pub bits_per_sample: u32,
    pub channel_mask: u32,
}

/// 从 IAudioMediaType 提取格式信息。
/// 通过 #[interface] trait 方法安全调用 GetAudioFormat()，不使用裸 vtable 索引。
pub fn extract_format(media_type: *mut IAudioMediaType) -> Result<AudioFormat>;

/// 检查是否为浮点格式（WAVE_FORMAT_IEEE_FLOAT 或 SubFormat = KSDATAFORMAT_SUBTYPE_IEEE_FLOAT）。
pub fn is_float_format(media_type: *mut IAudioMediaType) -> bool;
```

**禁止**：不包含 Filter 相关逻辑

---

### 4.4 `pipeline/channel.rs`

**职责**：通道映射、去交织/交织、默认通道掩码。

**引用来源**：无外部依赖

**导出给**：`pipeline/process.rs`、`pipeline/chain.rs`、`config/`

**公开 API**：

```rust
pub fn default_channel_mask(channels: u32) -> u32;
pub fn get_channel_names(mask: u32) -> Vec<String>;

// ── 去交织/交织 ──

/// 交织格式 → 去交织平面缓冲区（分配新 Vec<Vec<f32>>）。非实时路径用。
pub fn deinterleave(input: &[f32], channels: usize, frames: usize) -> Vec<Vec<f32>>;

/// 去交织平面缓冲区 → 交织格式（分配新 Vec<f32>）。非实时路径用。
pub fn interleave(channels: &[Vec<f32>], frames: usize) -> Vec<f32>;

/// 交织格式 → 已分配的去交织缓冲区（零分配，写入 output）。
///
/// `output` 为预分配的 `Vec<Vec<f32>>`，每通道长度 ≥ frames。
/// 内部通过 `output[ch].as_mut_slice()` 提取切片，无堆分配。
pub fn deinterleave_into(input: &[f32], output: &mut [Vec<f32>], channels: usize, frames: usize);

/// 已分配的去交织缓冲区 → 交织格式（零分配，写入 output）。
///
/// 内部通过 `input[ch].as_slice()` 提取切片，无堆分配。
pub fn interleave_from(input: &[Vec<f32>], output: &mut [f32], channels: usize, frames: usize);
```

**性能约束**：去交织/交织使用编译期展开特化（channels 为 2/6/8 时）。实时路径使用 `deinterleave_into` / `interleave_from`（零分配）。

---

### 4.5 `pipeline/chain.rs`

**职责**：Filter 链执行与延迟累计。工作在去交织空间。不拥有缓冲区，接受外部传入。

**引用来源**：
- `crate::pipeline::dsp::filter::Filter`
- `crate::utils::vx_error::VxApoError`

**导出给**：`pipeline/process.rs`

**字段**：

```rust
pub struct Chain {
    filters: Vec<Box<dyn Filter>>,
    total_latency: u32,
}
```

> **设计理由**：Chain 不持有 `max_frame_count`（缓冲区大小由调用方管理），不持有缓冲区（由 `ApoObjectInner` 预分配）。Chain 的唯一职责是按序执行 Filter 链。这保证了：
> - Chain 替换（`hot_reload`）不涉及缓冲区重分配
> - 过渡模式下两个 Chain 共享同一组工作缓冲区（串行使用）
> - `process()` 为纯计算操作，RT 安全

**公开 API**：

```rust
impl Chain {
    pub fn new() -> Self;
    pub fn add_filter(&mut self, filter: Box<dyn Filter>) -> Result<()>;
    pub fn total_latency(&self) -> u32;
    pub fn filter_count(&self) -> usize;

    /// 在去交织空间执行 Filter 链。
    ///
    /// `samples` 为预分配的去交织平面缓冲区（`samples[channel][frame]`）。
    /// 逐 Filter 调用 `filter.process(samples, frame_count)`。
    /// 纯计算操作：无锁、无分配、无 I/O。
    ///
    /// # Safety 不变量
    ///
    /// - `samples.len()` >= 所有 Filter 期望的通道数
    /// - `samples[ch].len()` >= `frame_count` 对所有通道
    /// - `frame_count` <= `LockForProcess` 时确定的 `max_frame_count`
    ///
    /// 违反上述条件返回 `Err`（防御性检查），调用方按 `ErrorPolicy` 处理。
    pub fn process(&mut self, samples: &mut [Vec<f32>], frame_count: usize) -> Result<()>;

    /// 重置所有 Filter 状态。
    pub fn reset(&mut self);
}
```

**`process()` 实现逻辑**：

```rust
pub fn process(&mut self, samples: &mut [Vec<f32>], frame_count: usize) -> Result<()> {
    for filter in self.filters.iter_mut() {
        filter.process(samples, frame_count);
    }
    Ok(())
}
```

> 当前实现中 `process()` 始终返回 `Ok(())`。保留 `Result` 返回类型用于未来扩展（如帧数越界检查、`catch_unwind` 集成）。

**禁止**：不知道 `config/`、`object/`、缓冲区生命周期

---

### 4.6 `pipeline/process.rs`

**职责**：APOProcess 调度入口。负责交织/去交织边界转换、BufferFlags 处理、错误策略。提供桥接函数供过渡模式使用。

**引用来源**：
- `crate::pipeline::context::PipelineContext`
- `crate::pipeline::chain::Chain`
- `crate::pipeline::buffer::{BufferInfo, evaluate_buffer, BufferAction, is_silent, zero_buffers, copy_buffers}`
- `crate::pipeline::channel::{deinterleave_into, interleave_from}`
- `crate::pipeline::dsp::transition::mix_buffers`
- `crate::sys::com::apo_types::{APO_CONNECTION_PROPERTY, APO_BUFFER_FLAGS}`
- `crate::utils::vx_error::*`

**导出给**：`object/apo.rs`

---

#### 类型定义

```rust
#[derive(Copy, Clone)]
pub enum ErrorPolicy {
    Bypass,   // 链处理失败时直通（仅当输入有效时安全）
    Silence,  // 链处理失败时静音
}

pub struct ProcessParams {
    pub input_channels: u32,
    pub output_channels: u32,
    pub sample_rate: u32,
    pub max_frame_count: usize,
    pub valid_frame_count: usize,
    pub error_policy: ErrorPolicy,
    pub allow_silent_buffer: bool,         // 来自 ApoObjectState.allow_silent_buffer_modification
}

pub struct ProcessStatistics {
    pub error_count: AtomicU32,
}

impl ProcessStatistics {
    pub fn new() -> Self;
}
```

---

#### `apply_error_policy` 函数

```rust
/// 统一错误恢复：链处理失败时按策略恢复输出。
///
/// 用于 process_audio（正常模式）和 object/apo.rs（过渡模式）。
/// 实时安全：无分配、无锁、无 I/O。
pub fn apply_error_policy(
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
                    let copy_len = input_slice.len().min(output_buffer.len());
                    output_buffer[..copy_len].copy_from_slice(&input_slice[..copy_len]);
                    if output_buffer.len() > copy_len {
                        output_buffer[copy_len..].fill(0.0);
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

#### `process_chain_interleaved` 函数

```rust
/// 对单个 Chain 执行完整的交织→去交织→处理→交织流程。
///
/// 用途：
/// 1. 过渡模式——object/apo.rs 对新旧两个 Chain 各调用一次，再用 mix_buffers 混合
/// 2. 可作为独立的链执行桥接函数
///
/// 数据流：
///   input（交织）→ deinterleave_into → temp（去交织）→ chain.process → interleave_from → output（交织）
///
/// # 参数
///
/// - `chain`：要执行的 Filter 链
/// - `input`：交织格式输入切片（长度 = frame_count × channels）
/// - `output`：交织格式输出切片（长度 = frame_count × channels）
/// - `channels`：通道数
/// - `frame_count`：帧数
/// - `temp`：预分配的去交织工作缓冲区（通道数 ≥ channels，每通道长度 ≥ frame_count）
///
/// # 实时安全
///
/// 所有缓冲区预分配。无堆分配、无锁、无 I/O。
pub fn process_chain_interleaved(
    chain: &mut Chain,
    input: &[f32],
    output: &mut [f32],
    channels: usize,
    frame_count: usize,
    temp: &mut [Vec<f32>],
) -> Result<()> {
    deinterleave_into(input, temp, channels, frame_count);
    chain.process(temp, frame_count)?;
    interleave_from(temp, output, channels, frame_count);
    Ok(())
}
```

> **过渡模式调用示例**（`object/apo.rs` APOProcess）：
> ```rust
> let channels = params.input_channels as usize;
> let frames = params.valid_frame_count;
>
> // 旧链 → temp_buffer_old
> let r1 = process_chain_interleaved(
>     inner.outgoing_chain.as_mut().unwrap(),
>     input_slice, &mut inner.temp_buffer_old,
>     channels, frames, &mut inner.temp_buffers,
> );
> apply_error_policy(r1, input_slice, &mut inner.temp_buffer_old, params.error_policy, &self.process_stats);
>
> // 新链 → temp_buffer_new
> let r2 = process_chain_interleaved(
>     &mut inner.current_chain,
>     input_slice, &mut inner.temp_buffer_new,
>     channels, frames, &mut inner.temp_buffers,
> );
> apply_error_policy(r2, input_slice, &mut inner.temp_buffer_new, params.error_policy, &self.process_stats);
>
> // 混合
> let factor = inner.transition.as_mut().unwrap().advance().unwrap_or(1.0);
> unsafe { mix_buffers(
>     inner.temp_buffer_old.as_ptr(), inner.temp_buffer_new.as_ptr(),
>     output_ptr, frames * channels, factor,
> ); }
> ```
>
> 两次调用共享 `temp_buffers`（去交织工作区），串行执行安全。各自产出独立的交织输出到 `temp_buffer_old` / `temp_buffer_new`。

---

#### `process_audio` 函数

```rust
/// APOProcess 正常模式的完整处理流程。
///
/// 数据流：
/// 1. evaluate_buffer → 判断处理动作
/// 2. 从 APO_CONNECTION_PROPERTY 提取交织输入切片
/// 3. deinterleave_into → 去交织平面缓冲区（零分配）
/// 4. is_silent 优化检测（去交织空间）
/// 5. mono 上混（单声道输入且输出 ≥2 时）
/// 6. chain.process（去交织空间）
/// 7. interleave_from → 交织输出切片（零分配）
/// 8. apply_error_policy（chain.process 失败时）
/// 9. 设置输出 APO_CONNECTION_PROPERTY 的 buffer_flags
///
/// 预分配要求：temp_buffers 由调用方（object/apo.rs）在 LockForProcess 时预分配，
/// 通道数 = max(input_channels, output_channels)，每通道帧数 = max_frame_count。
pub fn process_audio(
    input_props: &[APO_CONNECTION_PROPERTY],
    output_props: &mut [APO_CONNECTION_PROPERTY],
    params: &ProcessParams,
    chain: &mut Chain,
    stats: &ProcessStatistics,
    temp_buffers: &mut [Vec<f32>],
) -> Result<()>;
```

**完整处理流程（伪代码）**：

```rust
pub fn process_audio(
    input_props: &[APO_CONNECTION_PROPERTY],
    output_props: &mut [APO_CONNECTION_PROPERTY],
    params: &ProcessParams,
    chain: &mut Chain,
    stats: &ProcessStatistics,
    temp_buffers: &mut [Vec<f32>],
) -> Result<()> {
    let in_ch = params.input_channels as usize;
    let out_ch = params.output_channels as usize;
    let frames = params.valid_frame_count;

    for (i, (input_prop, output_prop)) in input_props.iter().zip(output_props.iter_mut()).enumerate() {
        let input_info = BufferInfo::from_prop(input_prop, in_ch);
        let mut output_info = BufferInfo::from_prop_mut(output_prop, out_ch);

        // ── Step 1: evaluate_buffer ──────────────────────────────────────
        let (action, mut output_flags) = evaluate_buffer(input_prop.buffer_flags, params.allow_silent_buffer);

        match action {
            BufferAction::Skip | BufferAction::Silent => {
                output_info.zero();
                output_prop.buffer_flags = APO_BUFFER_FLAGS::Silent;
                continue;
            }
            BufferAction::Process => { /* 继续 */ }
        }

        let input_slice = input_info.as_slice();
        let output_slice = output_info.as_slice_mut();

        // ── Step 2: 交织 → 去交织 ───────────────────────────────────────
        deinterleave_into(input_slice, &mut temp_buffers[..in_ch], in_ch, frames);

        // ── Step 3: is_silent 优化检测（去交织空间） ─────────────────────
        // allow_silent_buffer 场景下，如果实际数据确实静音，跳过 DSP 处理
        if output_flags == APO_BUFFER_FLAGS::Silent {
            if is_silent(&temp_buffers[..out_ch], frames) {
                output_info.zero();
                output_prop.buffer_flags = APO_BUFFER_FLAGS::Silent;
                continue;
            } else {
                // 标记非静音（系统可能错误标记），恢复正常处理
                output_flags = APO_BUFFER_FLAGS::Valid;
            }
        }

        // ── Step 4: mono 上混 ───────────────────────────────────────────
        // 单声道输入且输出 ≥2 时：通道 0 → 通道 1
        if in_ch == 1 && out_ch >= 2 {
            for f in 0..frames {
                temp_buffers[1][f] = temp_buffers[0][f];
            }
        }

        // ── Step 5: DSP 处理（去交织空间） ──────────────────────────────
        let result = chain.process(&mut temp_buffers[..out_ch], frames);

        // ── Step 6: 错误恢复 ────────────────────────────────────────────
        if result.is_err() {
            // chain.process 失败：按策略恢复
            // Bypass：需要原始交织输入，已在 input_slice 中
            // Silence：直接清零
            // 恢复后跳过 interleave，直接从 input_slice 构造输出
            apply_error_policy(result, input_slice, output_slice, params.error_policy, stats);
            output_prop.buffer_flags = match params.error_policy {
                ErrorPolicy::Bypass => APO_BUFFER_FLAGS::Valid,
                ErrorPolicy::Silence => APO_BUFFER_FLAGS::Silent,
            };
            continue;
        }

        // ── Step 7: 去交织 → 交织 ───────────────────────────────────────
        interleave_from(&temp_buffers[..out_ch], output_slice, out_ch, frames);
        output_prop.buffer_flags = APO_BUFFER_FLAGS::Valid;
    }
    Ok(())
}
```

**关键不变式**：
- 输入 Invalid → **强制静音**（evaluate_buffer 返回 Skip），不经过任何恢复路径
- 输入 Silent + !allow_silent_buffer → **强制静音**，不经过 DSP
- 输入 Silent + allow_silent_buffer + 实际数据静音 → **跳过 DSP**（优化快速路径）
- 输入 Silent + allow_silent_buffer + 实际数据非静音 → **正常处理**
- 只有输入有效时，`ErrorPolicy::Bypass` 才被执行（确保 input_slice 有意义）
- 边界转换使用 `deinterleave_into` / `interleave_from`（零分配）
- `temp_buffers` 由调用方预分配，不在 RT 路径分配
- RT 路径错误使用原子计数器，不使用 `format!()` 或任何堆分配
- 每个输出通道的 `buffer_flags` 由 `process_audio` 独立设置

**禁止**：不知道 `object/`、`config/`、`install/`

---

### 4.7 `pipeline/realtime/contract.rs`

**职责**：RT-safety 契约。实时上下文标记、断言、安全索引。

**引用来源**：`core::sync::atomic`

**导出给**：`pipeline/` 下所有实时路径模块

**公开 API**：

```rust
pub struct RtGuard;
impl RtGuard {
    pub fn enter() -> Self;
}
// Drop 时自动清除 RT 标志

pub unsafe trait RtSafe: Send + Sync {}
pub unsafe trait RtCopy: Copy {}

unsafe impl RtCopy for f32/f64/i8/i16/i32/i64/u8/u16/u32/u64/usize/isize/bool {}

#[macro_export] macro_rules! rt_assert_in_rt { ... }
#[macro_export] macro_rules! rt_assert_not_in_rt { ... }

pub unsafe fn rt_index<T>(slice: &[T], index: usize) -> &T;
pub unsafe fn rt_index_mut<T>(slice: &mut [T], index: usize) -> &mut T;
```

**Release 模式行为**：所有 `cfg(debug_assertions)` 代码被编译移除，零开销。

---

### 4.8 `pipeline/realtime/ring.rs`

**职责**：SPSC 无锁环形缓冲区。存储固定大小、实现 `Copy` 的类型。

**引用来源**：`core::sync::atomic`、`core::cell::UnsafeCell`

**导出给**：`telemetry/logger.rs`

**公开 API**：

```rust
pub struct RingBuffer<T: Copy> { ... }
impl<T: Copy> RingBuffer<T> {
    pub const fn new(capacity: usize) -> Self;
    pub fn push(&self, data: T) -> bool;
    pub fn pop(&self) -> Option<T>;
    pub fn is_empty(&self) -> bool;
    pub fn is_full(&self) -> bool;
    pub fn capacity(&self) -> usize;
}
```

**原子序**：`push` 使用 `Release` 序，`pop` 使用 `Acquire` 序。

**性能约束**：使用原子操作，无锁，实时安全。

---

### 4.9 `pipeline/dsp/filter.rs`

**职责**：Filter trait 定义 + FilterCreateResult + FilterFactory + DspContext + ConfigLoader。纯 Rust 定义，不包含任何 Windows API 依赖。

**引用来源**：`crate::utils::vx_error::VxApoError`

**导出给**：`pipeline/dsp/*.rs`、`pipeline/dsp/factory.rs`、`config/commands/*.rs`

---

#### Filter trait

```rust
/// 音频处理过滤器 trait。
///
/// 所有 DSP 滤波器实现此 trait。
/// process 方法运行在实时音频线程中，禁止堆分配、互斥锁、I/O、panic。
pub trait Filter: Send + Sync + std::fmt::Debug {
    /// 处理一帧音频数据（实时路径，去交织空间）。
    ///
    /// samples[channel][frame]，就地修改。
    /// 不得进行堆分配、I/O、panic。
    fn process(&mut self, samples: &mut [Vec<f32>], frame_count: usize);

    /// 初始化过滤器。
    /// 返回输出通道名列表（Some = 通道选择，None = 通道不变）。
    /// 不执行 I/O，不应失败。
    fn initialize(&mut self, sample_rate: u32, channel_names: &[String]) -> Option<Vec<String>>;

    /// 是否为 Channel: 类型命令（通道选择标记）。默认 false。
    fn is_channel_select(&self) -> bool { false }

    /// 过滤器的延迟（采样数）。默认 0。
    fn latency(&self) -> u32 { 0 }

    /// 最大帧数约束。默认 None（无约束）。Some(n) 表示 process 每次最多处理 n 帧。
    fn max_frame_count(&self) -> Option<usize> { None }

    /// 重置过滤器内部状态。默认空实现。
    fn reset(&mut self) {}
}
```

---

#### PassthroughFilter

```rust
/// 透传过滤器——不修改任何采样数据。
#[derive(Debug, Clone, Copy)]
pub struct PassthroughFilter;

impl Filter for PassthroughFilter {
    fn process(&mut self, _samples: &mut [Vec<f32>], _frame_count: usize) {}
    fn initialize(&mut self, _sample_rate: u32, _channel_names: &[String]) -> Option<Vec<String>> { None }
}
```

---

#### FilterCreateResult

```rust
/// 工厂创建结果（四种状态）。
#[derive(Debug)]
pub enum FilterCreateResult {
    /// 匹配成功，产生了过滤器实例。
    Filter(Box<dyn Filter>),
    /// 匹配成功，但不需要产生过滤器（如 Device: 匹配、Include: 已递归加载）。
    NoFilter,
    /// 匹配失败，继续尝试下一工厂。
    NoMatch,
    /// 当前文件应停止解析（如 Device: 不匹配）。
    AbortFile,
}
```

---

#### FilterFactory trait

```rust
/// 过滤器工厂 trait。
/// 每个配置命令对应一个工厂实现。
/// 遍历工厂列表，第一个返回 Filter 或 NoFilter 的工厂胜出。
pub trait FilterFactory: Send {
    /// 尝试根据配置行参数创建过滤器。
    ///
    /// - params：配置行冒号后的值部分（已 trim）
    /// - ctx：引擎上下文
    /// - loader：配置加载器回调（用于 Include: 递归）
    ///
    /// 返回 FilterCreateResult 四种状态之一。
    fn create_filter(
        &self,
        params: &str,
        ctx: &DspContext,
        loader: &dyn ConfigLoader,
    ) -> FilterCreateResult;

    /// 此工厂匹配的命令关键字（用于日志和调试）。
    fn command_name(&self) -> &str;
}
```

---

#### DspContext

```rust
/// 引擎统一上下文。纯数据结构，不包含任何 Windows 类型。
///
/// 由调用方从 PipelineContext + 额外参数构造（调用方依赖 context.rs）。
/// filter.rs 不依赖 pipeline/context.rs，保持模块独立性。
#[derive(Debug, Clone)]
pub struct DspContext {
    pub sample_rate: u32,
    pub channel_count: u32,
    pub channel_mask: u32,
    pub channel_names: Vec<String>,   // 通道名列表（Copy: 命令依赖）
    pub max_frame_count: u32,          // 最大帧数（Convolution 预分配依赖）
    pub bits_per_sample: u32,
    pub device_type: DeviceType,
    pub stage: ProcessingStage,
    pub variables: HashMap<String, f64>, // Eval: 命令变量存储
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum DeviceType { Render, Capture }

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ProcessingStage { None, PreMix, PostMix, Capture }
```

> 调用方构造示例（`object/apo.rs` LockForProcess 中）：
> ```rust
> let dsp_ctx = DspContext {
>     sample_rate: ctx.sample_rate,
>     channel_count: ctx.input_channels,
>     channel_mask: ctx.channel_mask,
>     max_frame_count: ctx.max_frame_count as u32,
>     channel_names,
>     bits_per_sample: 32,
>     device_type: DeviceType::Render,
>     stage: ProcessingStage::None,
>     variables: HashMap::new(),
> };
> ```

---

#### ConfigLoader trait

```rust
/// 配置加载器回调 trait。
/// Include: 命令通过此 trait 回调 parser.rs 递归加载子配置文件。
/// cmd_include.rs 不能直接 import parser.rs，必须通过此 trait 解耦。
pub trait ConfigLoader {
    /// 加载指定路径的配置文件。返回该文件产生的过滤器列表。
    /// 文件不存在或解析失败时返回空列表（降级为 passthrough）。
    fn load_config(&self, path: &str, ctx: &DspContext) -> Vec<Box<dyn Filter>>;
}
```

---

### 4.10 `pipeline/dsp/factory.rs`

**职责**：`FilterRegistry` + 工厂注册 + 工厂遍历匹配。

**引用来源**：
- `crate::pipeline::dsp::filter::*`
- `crate::utils::vx_error::VxApoError`

**导出给**：`config/commands/*.rs`、`config/parser.rs`

---

#### 工厂注册表

```rust
/// 过滤器工厂注册表。
/// 持有按优先级排序的工厂列表。
pub struct FilterRegistry {
    factories: Vec<Box<dyn FilterFactory>>,
}

impl FilterRegistry {
    pub fn new() -> Self;
    pub fn register(&mut self, factory: Box<dyn FilterFactory>);
    pub fn len(&self) -> usize;
    pub fn is_empty(&self) -> bool;

    /// 按索引获取工厂引用。
    pub fn get(&self, index: usize) -> Option<&dyn FilterFactory>;

    /// 遍历所有工厂的命令名。
    pub fn factory_names(&self) -> Vec<&str>;

    /// 按遍历顺序尝试创建过滤器。
    /// 第一个返回 Filter 或 NoFilter 的工厂胜出。
    pub fn try_create(
        &self,
        params: &str,
        ctx: &DspContext,
        loader: &dyn ConfigLoader,
    ) -> TryCreateOutcome;
}
```

---

#### 匹配结果

```rust
/// try_create 的返回结果。
#[derive(Debug)]
pub struct TryCreateOutcome {
    pub result: OutcomeKind,
    pub factory_index: Option<usize>,
    pub factory_name: Option<String>,
}

pub enum OutcomeKind {
    FilterAdded(Box<dyn Filter>),
    MatchedNoFilter,
    Aborted,
    Unmatched,
}
```

---

#### 工厂索引常量

```rust
pub const FACTORY_COUNT: usize = 15;

pub mod index {
    pub const DEVICE: usize = 0;
    pub const IF: usize = 1;
    pub const EVAL: usize = 2;
    pub const INCLUDE: usize = 3;
    pub const STAGE: usize = 4;
    pub const CHANNEL: usize = 5;
    pub const IIR: usize = 6;
    pub const BIQUAD: usize = 7;
    pub const PREAMP: usize = 8;
    pub const DELAY: usize = 9;
    pub const COPY: usize = 10;
    pub const CONVOLUTION: usize = 11;
    pub const GRAPHIC_EQ: usize = 12;
    pub const VST_PLUGIN: usize = 13;
    pub const LOUDNESS_CORRECTION: usize = 14;
}
```

---

#### 工厂注册

```rust
/// 创建默认工厂列表（15 个，按优先级排序）。
pub fn create_default_registry() -> Vec<Box<dyn FilterFactory>>;

/// 注册所有内置 Filter 工厂到 FilterRegistry。
pub fn register_builtin_filters(registry: &mut FilterRegistry);
```

---

### 4.11 `pipeline/dsp/transition.rs`

**职责**：升余弦混合 + 过渡因子生成（纯算法，不管理状态机）。`mix_buffers` 操作交织格式缓冲区。

**引用来源**：无外部依赖

**导出给**：`object/apo.rs`、`pipeline/process.rs`

**公开 API**：

```rust
pub fn raised_cosine(counter: u32, length: u32) -> f32;

pub struct SmoothingProvider { ... }
impl SmoothingProvider {
    pub fn new(length: u32) -> Self;
    pub fn begin(&mut self);
    pub fn advance(&mut self) -> Option<f32>;
    pub fn is_active(&self) -> bool;
    pub fn length(&self) -> u32;
    pub fn counter(&self) -> u32;
    pub fn progress(&self) -> f32;
    pub fn reset(&mut self);
    pub fn set_length(&mut self, length: u32);
}

pub fn default_smoothing_length(sample_rate: u32) -> u32;   // sample_rate / 20 ≈ 50ms
pub fn short_smoothing_length(sample_rate: u32) -> u32;     // sample_rate / 100 ≈ 10ms

/// 交织格式缓冲区混合：output = old × (1 - factor) + new × factor
///
/// # Safety
/// - `old`、`new`、`output` 必须指向长度 ≥ count 的有效 f32 缓冲区
/// - 三块缓冲区不得重叠
/// - `factor` 范围 [0.0, 1.0]
pub unsafe fn mix_buffers(
    old: *const f32, new: *const f32, output: *mut f32,
    count: usize, factor: f32,
);
```

> **说明**：`mix_buffers` 操作交织格式连续缓冲区。在 `object/apo.rs` 过渡模式中，`temp_buffer_old` 和 `temp_buffer_new` 已在交织空间，由 `process_chain_interleaved` 产出。

---

### 4.12 `pipeline/dsp/biquad.rs`

**职责**：双二阶滤波器实现。

**引用来源**：`crate::pipeline::dsp::filter::Filter`、`crate::utils::vx_error::VxApoError`

**导出给**：`pipeline/dsp/peq.rs`、`pipeline/dsp/hp_lp.rs`（不导出给 `config/`）

**公开 API**：

```rust
pub struct BiquadCoeffs {
    pub b0: f32, pub b1: f32, pub b2: f32,
    pub a1: f32, pub a2: f32,
}

impl BiquadCoeffs {
    pub const BYPASS: Self;
    pub fn lowpass(fc: f32, q: f32, sample_rate: f32) -> Self;
    pub fn highpass(fc: f32, q: f32, sample_rate: f32) -> Self;
    pub fn peak(fc: f32, gain_db: f32, q: f32, sample_rate: f32) -> Self;
    pub fn lowshelf(fc: f32, gain_db: f32, q: f32, sample_rate: f32) -> Self;
    pub fn highshelf(fc: f32, gain_db: f32, q: f32, sample_rate: f32) -> Self;
    pub fn notch(fc: f32, q: f32, sample_rate: f32) -> Self;
    pub fn allpass(fc: f32, q: f32, sample_rate: f32) -> Self;
    pub fn is_valid(&self) -> bool;
}

pub struct BiquadFilter { ... }
impl BiquadFilter {
    pub fn new(coeffs: BiquadCoeffs, structure: BiquadStructure) -> Self;
}
impl Filter for BiquadFilter { ... }

pub fn compute_coeffs(filter_type: BiquadType, fc: f32, gain_db: f32, q: f32, sample_rate: u32) -> BiquadCoeffs;
```

---

### 4.13 `pipeline/dsp/peq.rs`

**职责**：参量均衡器（级联 biquad）。

**引用来源**：`crate::pipeline::dsp::filter::Filter`、`crate::pipeline::dsp::biquad::*`

**导出给**：仅 `pipeline/dsp/graphic_eq.rs`

**公开 API**：

```rust
pub struct PeqFilter { ... }
impl PeqFilter {
    pub fn new() -> Self;
    pub fn add_band(&mut self, freq: f32, gain_db: f32, q: f32, sample_rate: f32);
}
impl Filter for PeqFilter { ... }
```

---

### 4.14 `pipeline/dsp/hp_lp.rs`

**职责**：高通/低通滤波器。

**引用来源**：`crate::pipeline::dsp::filter::Filter`、`crate::pipeline::dsp::biquad::*`

**导出给**：仅 `pipeline/dsp/` 内部

**公开 API**：

```rust
pub struct HighLowPassFilter { ... }
impl HighLowPassFilter {
    pub fn new(filter_type: BiquadType, fc: f32, q: f32) -> Self;
}
impl Filter for HighLowPassFilter { ... }
```

---

### 4.15 `pipeline/dsp/gain.rs`

**职责**：增益滤波器（含平滑插值）。

**引用来源**：`crate::pipeline::dsp::filter::Filter`

**导出给**：仅 `pipeline/dsp/` 内部

**公开 API**：

```rust
pub struct GainFilter { ... }
impl GainFilter {
    pub fn new(gain_db: f32) -> Self;
    pub fn set_gain(&mut self, gain_db: f32);
    pub fn set_smoothing(&mut self, length: u32);
}
impl Filter for GainFilter { ... }
```

---

### 4.16 `pipeline/dsp/delay.rs`

**职责**：延迟线（环形缓冲实现）。

**引用来源**：`crate::pipeline::dsp::filter::Filter`

**导出给**：仅 `pipeline/dsp/` 内部

**公开 API**：

```rust
pub struct DelayFilter { ... }
impl DelayFilter {
    pub fn new(delay_ms: f32) -> Self;
    pub fn set_delay(&mut self, delay_ms: f32, sample_rate: f32);
    pub fn set_mix(&mut self, dry: f32, wet: f32);
}
impl Filter for DelayFilter { ... }
```

---

### 4.17 `pipeline/dsp/copy.rs`

**职责**：通道复制/混音。

**引用来源**：`crate::pipeline::dsp::filter::Filter`、`crate::pipeline::channel::*`

**导出给**：仅 `pipeline/dsp/` 内部

**公开 API**：

```rust
pub struct CopyFilter { ... }
impl CopyFilter {
    pub fn new(mappings: Vec<ChannelMapping>) -> Self;
}
impl Filter for CopyFilter { ... }

pub struct ChannelMapping {
    pub src: String,
    pub dst: String,
    pub gain: f32,
}

pub fn parse_copy_ops(spec: &str, channel_names: &[String]) -> Option<Vec<ChannelMapping>>;
```

---

### 4.18 `pipeline/dsp/graphic_eq.rs`

**职责**：图形均衡器（多段 PEQ 级联）。

**引用来源**：`crate::pipeline::dsp::filter::Filter`、`crate::pipeline::dsp::peq::PeqFilter`

**导出给**：仅 `pipeline/dsp/` 内部

**公开 API**：

```rust
pub struct GraphicEqFilter { ... }
impl GraphicEqFilter {
    pub fn new(bands: Vec<(f32, f32)>) -> Self;
    pub fn set_bands(&mut self, bands: Vec<(f32, f32)>, sample_rate: f32);
}
impl Filter for GraphicEqFilter { ... }

pub fn parse_graphic_eq_params(spec: &str) -> Option<Vec<(f32, f32)>>;
```

---

### 4.19 `pipeline/dsp/convolution.rs`

**职责**：卷积（FFT 骨架）。

**引用来源**：`crate::pipeline::dsp::filter::Filter`

**导出给**：仅 `pipeline/dsp/` 内部

**公开 API**：

```rust
pub struct ConvolutionFilter { ... }
impl ConvolutionFilter {
    pub fn new(path: &str, gain_db: f32) -> Self;
}
impl Filter for ConvolutionFilter { ... }

pub fn parse_convolution_params(spec: &str) -> Option<(String, f32)>;
```

---

### 4.20 `pipeline/dsp/vst.rs`

**职责**：VST 插件加载。

**引用来源**：`crate::pipeline::dsp::filter::Filter`

**导出给**：仅 `pipeline/dsp/` 内部

**公开 API**：

```rust
pub struct VstFilter { ... }
impl VstFilter {
    pub fn new(path: &str, name: &str) -> Self;
}
impl Filter for VstFilter { ... }

pub fn parse_vst_params(spec: &str) -> Option<(String, String, String)>;
```

> **注意**：`feature gate = ["vst"]`，编译时通过 Cargo feature 控制。

---

### 4.21 `pipeline/dsp/loudness.rs`

**职责**：ISO 226 等响曲线。

**引用来源**：`crate::pipeline::dsp::filter::Filter`

**导出给**：仅 `pipeline/dsp/` 内部

**公开 API**：

```rust
pub struct LoudnessFilter { ... }
impl LoudnessFilter {
    pub fn new(phon: f32, reference_phon: f32) -> Self;
}
impl Filter for LoudnessFilter { ... }

pub fn parse_loudness_params(spec: &str) -> Option<(f32, f32)>;
```