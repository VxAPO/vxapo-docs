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
pipeline/interleave.rs — deinterleave_into（交织 → 去交织，零分配）
  │
  ▼
pipeline/chain.rs — 逐 Filter 调用（去交织空间）
  │   └── filter.process(&mut [Vec<f32>], frame_count)
  │
  ▼
pipeline/interleave.rs — interleave_from（去交织 → 交织，零分配）
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
├── interleave.rs       # 通道数据搬运（去交织/交织）
├── realtime.rs         # 实时安全基础设施模块入口
├── realtime/
│   ├── ring.rs         # SPSC 无锁环形缓冲区
│   └── contract.rs     # RT 上下文断言（RtGuard / assert_in_rt）
├── dsp.rs              # DSP 算法模块入口
└── dsp/
    ├── filter.rs       # Filter trait + DspContext + PassthroughFilter + ChannelScopedFilter（v9.11）
    ├── factory.rs      # v9.11 静态分派 create_from_model（match EffectType）
    ├── transition.rs   # SmoothingProvider + raised_cosine + mix_buffers
    ├── biquad.rs       # 双二阶滤波器
    ├── fir.rs          # SIMD dot / 分块 FFT（v9.11）
    ├── model.rs        # ChainModel / EffectType（v9.11）
    ├── peq_hybrid.rs   # 混合式 PEQ（200 Hz IIR + 最小相位 FIR，v9.11）
    ├── gain.rs         # 增益（含内部平滑插值）
    ├── loudness.rs     # ISO 226 等响曲线
    ├── aural.rs        # Aural Enhancer（谐波激励）
    ├── compressor.rs   # Compressor（全声道联动 RMS + 软膝）
    ├── reverb.rs       # Dattorro 板式混响
    └── wide.rs         # 立体声加宽
```

---

### 引用约束总表

> v9.17（单一事实源）：本表不再独立维护——以主规范
> `模块引用规范（无详细模块版）.md` 第十一节「引用约束总表」为唯一基线，
> pipeline 各文件的允许/禁止依赖逐行见主规范。

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
| Silent | true | Process | **内容按全零处理，不读残留**（v9.15：SILENT 标志权威，对齐 EAPO C11） |
| Valid | 任意 | Process | 正常处理 |

> **v9.15 修订（脏静音缓冲）**：引擎在流切换/静音时可能发 `BUFFER_SILENT` 但复用
> 上一帧缓冲——内存残留的是**本 APO 上一帧输出**。若把残留内容当真处理，输出会再
> 作为下一帧输入形成自我反馈爆音（日志实证输出峰值 17–95401 倍满刻度）。因此
> SILENT 输入在**去交织前按全零填充**（绝不读取内容），链状态照常衰减，输出强制
> 清零 + `BUFFER_SILENT`；`is_silent` 内容判定仅对 VALID 输入生效。

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

### 4.4 `pipeline/interleave.rs`

**职责**：通道数据搬运（去交织/交织）。不包含掩码映射或通道名逻辑（已迁移至 `sys/audio_defs.rs`）。

**引用来源**：无外部依赖

**导出给**：`pipeline/process.rs`、`pipeline/chain.rs`

**公开 API**：

```rust
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
> - **`initialize()` 必须在首次 `process()` 前调用**：GraphicEQ/PEQ/IIR/Delay/Convolution 等滤波器在 `initialize` 中预计算系数/分配状态；漏调会变成空处理（声音不变）
> - **v9.0：`initialize()` 末尾重算 `total_latency`**——卷积型滤波器的延迟要到 initialize 才知道（IR/分块），不能在 `add_filter` 时一次性累计

**公开 API**：

```rust
impl Chain {
    pub fn new() -> Self;
    pub fn add_filter(&mut self, filter: Box<dyn Filter>) -> Result<()>;
    pub fn initialize(&mut self, sample_rate: u32, channel_names: &[String]);
    pub fn total_latency(&self) -> u32;
    pub fn filter_count(&self) -> usize;

    /// 链是否为空（零滤波器，R3/v6.9）。
    ///
    /// `filters.is_empty()`。`process_audio` 据此走零拷贝快路径——空链时
    /// `temp_buffers` 原样即输出，直接复制到交织输出，跳过整条链遍历。
    pub fn is_empty(&self) -> bool;

    /// 全链是否均就地处理（E1/v6.7）。
    ///
    /// `filters.iter().all(|f| f.is_in_place())`。调用方（process_audio）据此
    /// 决定是否可走零拷贝快路径：全链 `true` 时去交织缓冲即最终输出，
    /// 无需任何滤波间中间副本。
    pub fn is_fully_in_place(&self) -> bool;

    /// 在去交织空间执行 Filter 链。
    ///
    /// `samples` 为预分配的去交织平面缓冲区（`samples[channel][frame]`）。
    /// 逐 Filter 调用 `filter.process(samples, frame_count)`。
    /// 纯计算操作：无锁、无分配、无 I/O。
    ///
    /// **in-place 语义（E1）**：默认全链就地修改同一 `samples`。
    /// 若某滤波器 `is_in_place() == false`，Chain 需在调用其 `process` 前
    /// 将 `samples` 副本保存至内部工作缓冲（E2 未来落点；当前内置滤波器均返回 true）。
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
- `crate::pipeline::interleave::{deinterleave_into, interleave_from}`
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
                    // in-place APO 输入输出可能重叠：逐元素拷贝（memmove 语义），
                    // 不能用 copy_from_slice（重叠 UB，2026-08-10 实证）。
                    for i in 0..copy_len {
                        output_buffer[i] = input_slice[i];
                    }
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
/// v9.0 防御：若 `frame_count` 超过 `temp` 平面缓冲或 `input/output` 交织缓冲容量，
/// 直接逐元素旁通（memmove 语义），不 panic、不谎报帧数。
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
/// 5. 输出通道扩展（后通道清零 + mono 上混）
/// 6. chain.process（去交织空间）
/// 7. interleave_from → 交织输出切片（零分配）
/// 8. apply_error_policy（chain.process 失败时）
/// 9. 设置输出 APO_CONNECTION_PROPERTY 的 buffer_flags
///
/// 预分配要求：temp_buffers 由调用方（object/apo.rs）在 LockForProcess 时预分配，
/// 通道数 = max(input_channels, output_channels)，每通道帧数 = max_frame_count。
/// v9.0：临时缓冲额外预留延迟余量；若引擎传入帧数超过 temp/输入/输出任一容量，
/// 直接旁通并只报实际写入帧数，不 panic、不谎报帧数。
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
        // v9.15：SILENT 输入按全零填充（内容无效，可能残留上一帧输出——
        // 直接读取会形成自我反馈爆音）；VALID 输入正常去交织。
        if input_prop.buffer_flags == APO_BUFFER_FLAGS::Silent {
            for ch in temp_buffers[..in_ch].iter_mut() {
                ch[..frames].fill(0.0);
            }
        } else {
            deinterleave_into(input_slice, &mut temp_buffers[..in_ch], in_ch, frames);
        }

        // ── Step 3: is_silent 优化检测（去交织空间） ─────────────────────
        // v9.15：仅对 VALID 输入做内容判定——SILENT 标志已由引擎声明，
        // 残留大数值不得被误判为有效音频。
        if output_flags == APO_BUFFER_FLAGS::Silent && input_prop.buffer_flags != APO_BUFFER_FLAGS::Silent {
            if is_silent(&temp_buffers[..out_ch], frames) {
                output_info.zero();
                output_prop.buffer_flags = APO_BUFFER_FLAGS::Silent;
                continue;
            } else {
                // 标记非静音（系统可能错误标记），恢复正常处理
                output_flags = APO_BUFFER_FLAGS::Valid;
            }
        }

        // ── Step 4: 输出通道扩展（后通道清零 + mono 上混） ───────────────
        // 输入通道数 < 输出通道数时，先清零 temp_buffers[in_ch..out_ch]。
        // temp_buffers 是跨处理循环复用的预分配缓冲区，不清零会导致
        // 上一帧残留数据泄漏到 DSP 链和最终输出。
        if out_ch > in_ch {
            for ch in in_ch..out_ch {
                temp_buffers[ch][..frames].fill(0.0);
            }
        }
        // mono 上混：单声道输入且输出 ≥2 时：通道 0 → 通道 1
        if in_ch == 1 && out_ch >= 2 {
            for f in 0..frames {
                temp_buffers[1][f] = temp_buffers[0][f];
            }
        }

        // ── Step 5: DSP 处理（去交织空间） ──────────────────────────────
        // E1（v6.7）：全链 in-place 时 temp_buffers 即最终输出（零拷贝快路径）；
        // 存在非就地滤波器时 Chain 内部负责输入保护（当前内置全 true，无需分支）。
        // R3（v6.9）：空链（is_empty）时直接复制去交织结果到输出（近似 memcpy），跳过链遍历。
        let result = if chain.is_empty() {
            copy_buffers(&temp_buffers[..out_ch], &mut temp_buffers[..out_ch], frames);
            Ok(())
        } else {
            chain.process(&mut temp_buffers[..out_ch], frames)
        };

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
        if input_prop.buffer_flags == APO_BUFFER_FLAGS::Silent {
            // 引擎声明静音：输出必须为静音（链状态已照常更新，但内容不落到输出）。
            output_slice.fill(0.0);
            output_prop.buffer_flags = APO_BUFFER_FLAGS::Silent;
        } else {
            output_prop.buffer_flags = APO_BUFFER_FLAGS::Valid;
        }
    }
    Ok(())
}
```

**关键不变式**：
- 输入 Invalid → **强制静音**（evaluate_buffer 返回 Skip），不经过任何恢复路径
- 输入 Silent + !allow_silent_buffer → **强制静音**，不经过 DSP
- 输入 Silent + allow_silent_buffer → **按全零处理**（不读残留），输出强制
  SILENT；链状态照常更新（v9.15，对齐 EAPO C11）
- 只有输入有效时，`ErrorPolicy::Bypass` 才被执行（确保 input_slice 有意义）
- 边界转换使用 `deinterleave_into` / `interleave_from`（零分配）
- `temp_buffers` 由调用方预分配，不在 RT 路径分配
- RT 路径错误使用原子计数器，不使用 `format!()` 或任何堆分配
- 每个输出通道的引脚标志 `u32BufferFlags`（windows-rs 字段名，取值 `BUFFER_VALID` / `BUFFER_SILENT` / `BUFFER_INVALID`）由 `process_audio` 独立设置

**禁止**：不知道 `object/`、`config/`、`install/`

---

### 4.7 `pipeline/realtime/contract.rs`

**职责**：RT-safety 契约。实时上下文标记（编译期见证）、断言、安全索引。

**引用来源**：`core::sync::atomic`、`core::marker::PhantomData`

**导出给**：`pipeline/` 下所有实时路径模块、`pipeline/dsp/filter.rs`（`DspContext::rt_marker`）

**公开 API**：

```rust
/// 零尺寸 RT 上下文标记——**编译期见证**（tympan-apo 借鉴，v6.6 引入）。
///
/// `RealtimeContext` 无字段、无用户可达构造函数；RT harness（pipeline/process.rs
/// 的 RT 内部函数）在实时路径创建后按引用传递。其出现在调用栈中即为
/// "此代码路径 RT 安全"的编译期证明——比运行时 `rt_assert_in_rt!` 更强。
///
/// 设计原则：**编译期能解决的问题，绝不拖到运行时**。
pub struct RealtimeContext { _private: () }

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

### 4.8 `utils/ring.rs`

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

**职责**：Filter trait 定义 + DspContext + PassthroughFilter + ChannelScopedFilter
（v9.11 删除 FilterCreateResult / FilterFactory / ConfigLoader——静态分派后不再需要）。
纯 Rust 定义，不包含任何 Windows API 依赖。

**引用来源**：`crate::utils::vx_error::VxApoError`

**导出给**：`pipeline/dsp/*.rs`、`pipeline/dsp/factory.rs`、`config/parser.rs`、`pipeline/chain.rs`

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

    /// 是否就地处理（默认 true，E1/v6.7，EqualizerAPO `IFilter::getInPlace` 借鉴）。
    ///
    /// - `true`：此滤波器在 `samples[ch][f]` 上**逐采样覆盖式**就地修改，
    ///   不依赖"进入本滤波器前"的相邻采样旧值（Biquad/Gain/GraphicEQ/Copy 等）。
    /// - `false`：处理过程中需要读取 `samples` 的**原始输入**（非就地语义，
    ///   如未来叠加式 Delay / 部分 FFT Convolution 实现）。
    ///
    /// 返回 false 的滤波器由 `Chain::process` 在调用前以工作缓冲保存输入副本
    /// （E2 未来落点；当前既有 Delay/Convolution 均有内部环形缓冲，无需 chain 干预）。
    fn is_in_place(&self) -> bool { true }

    /// 重置过滤器内部状态。默认空实现。
    fn reset(&mut self) {}
}
```

> **`is_in_place` 与 `process_audio` 快路径（E1）**：全链 `is_in_place() == true` 时，
> 去交织工作缓冲 `temp_buffers` 既是链输入又是链输出——无需在滤波器间插入任何
> 中间副本（现有流程天然满足零拷贝）。链中存在 `false` 滤波器时才需额外的
> 输入保护缓冲。当前所有内置滤波器（含 Delay 环形缓冲、Convolution 骨架）
> 均可安全声明 `true`。
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
    pub rt_marker: PhantomData<RealtimeContext>, // RT 编译期见证（v6.6）
}
```

> **`rt_marker`（O1：RT 编译期见证）**：`PhantomData<RealtimeContext>` 零尺寸字段——
> 将"此配置/上下文服务于实时路径"的语义**前移到编译期**。DspContext 由 object/apo.rs
> 在非 RT 路径构造，但其内在方法（Filter 创建/初始化）均服务于 RT `process`；
> `rt_marker` 使 pipeline 内部方法可通过 `&RealtimeContext` 借贷链传递 RT 状态，
> 逐步将 `rt_assert_in_rt!` 运行时断言升级为编译期见证（tympan-apo 借鉴）。
>
> **演进方向**：新代码优先声明 `fn f(&self, rt: &RealtimeContext, ...)`；存量使用
> `rt_assert_in_rt!` 的路径按批次迁移。禁止在非 RT 路径构造 `RealtimeContext`（无公开构造器）。

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

**职责**：v9.11 起为 **模型 → Filter 静态分派**：`create_from_model(EffectConfig,
DspContext) -> Box<dyn Filter>`，`match EffectType` 穷尽 7 个类型；已删除
`FilterFactory` / `FilterRegistry` / `FilterCreateResult` / `OutcomeKind` /
`index` 常量与注册顺序测试（不再 EAPO 对齐）。

**引用来源**：
- `crate::pipeline::dsp::filter::*`
- `crate::pipeline::dsp::model::*`（EffectConfig/EffectType，v9.11）
- `crate::pipeline::dsp/*.rs`（**具体 Filter 实现：工厂注册中心必要例外**——注册必须实例化具体类型）
- `crate::utils::vx_error::VxApoError`

> **例外说明**：工厂注册中心必须直接引用具体 Filter 类型（`biquad`/`peq`/`convolution`/`copy`/`delay`/`graphic_eq`/`hp_lp`/`loudness`/`vst` 等）才能实例化并注册工厂。此例外仅适用于 `factory.rs`；`config/` 仍禁止直接引用具体实现。

**导出给**：`config/parser.rs`

---

#### 工厂注册表

> **v9.11 废弃**：以下 FilterRegistry / FilterFactory / create_default_registry /
> register_builtin_filters 为 v9.11 前动态注册表实现，已删除；现行为
> `create_from_model` 静态 match 分派（见本节上方职责）。保留为历史参考。

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

    /// 按命令名精确分派（v9.1）。
    /// 只尝试 `command_name()` 与给定命令（不区分大小写）一致的工厂；
    /// 无工厂命中时返回 `Unmatched`。parser 默认分支使用此方法，
    /// 避免 Convolution 等宽容工厂吞掉已知命令的非法参数。
    pub fn try_create_named(
        &self,
        command: &str,
        params: &str,
        ctx: &DspContext,
        loader: &dyn ConfigLoader,
    ) -> TryCreateOutcome;
}
```

---

#### 匹配结果（v9.11 已删除，历史参考）

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

#### 工厂索引常量（v9.11 已删除，历史参考）

```rust
pub const FACTORY_COUNT: usize = 19;

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
    pub const AURAL_ENHANCER: usize = 15;
    pub const REVERB: usize = 16;
    pub const MAXIMIZER: usize = 17;
    pub const WIDE: usize = 18;
}
```

---

#### 工厂注册（v9.11 已删除，历史参考）

> 以下 EAPO 风格命令语法（`AuralEnhancer:` / `Reverb:` / `Maximizer:` / `Wide:`）与
> 工厂注册顺序均不再存在；现行配置为 TOML `[[effects]]`，效果器与参数见本节 4.22。

```rust
/// 创建默认工厂列表（19 个，按优先级排序）。
pub fn create_default_registry() -> Vec<Box<dyn FilterFactory>>;

/// 注册所有内置 Filter 工厂到 FilterRegistry。
pub fn register_builtin_filters(registry: &mut FilterRegistry);
```

**注册顺序（v9.2）**：IIR → Biquad → Preamp → Delay → Copy → Convolution → GraphicEQ
→ VSTPlugin → LoudnessCorrection → **AuralEnhancer → Reverb → Maximizer → Wide**。

**命令语法（v9.3，`AuralEnhancer:` / `Reverb:` / `Maximizer:` / `Wide:`）**：

- `AuralEnhancer: TuneHz 1760 Drive 1.77 Odd 1.5 Even 0.0 Wet 1.0 Dry 0.0`
  - TuneHz 默认 1760 Hz（原 Quick preset 1 / MIDI 53），范围 [500, 10000] Hz；
    Drive [0, 4.25]，Odd [0, 1.5]，Even [0, 0.75]，Wet/Dry [0, 1]。
- `Reverb: RoomSize 1.0 Decay 0.566 Damping 0.408 Bandwidth 0.350
  Density 1.0 Lat5 0.70 Lat6 0.50 PreDelay 0 ms MotionRate 0.11 MotionDepth 0.63 ms Wet 0.3 Dry 0.9`
  - 各参数 clamp 到效果器自身区间：RoomSize [0.5, 1.5]、Decay/Damping/Bandwidth/Density [0, 1]、
    PreDelay [0, 100] ms、MotionRate [0.05, 2.0]、MotionDepth [0, 2.0] ms。
  - v9.3 起为 Dattorro 板式混响语义：RoomSize 缩放槽内延迟；Decay → 环路反馈；
    Damping/Bandwidth → 槽内/输入低通（0=暗淡，1=明亮）；Density → 扩散系数；
    Lat5/Lat6 → 早反射/尾音电平；MotionRate → LFO 频率（Hz）；MotionDepth → 调制深度
    （2 ms = 论文 EXCURSION 16 采样@29761 Hz）。
- `Maximizer: GainBoost 6 dB MaxOutput -0.3 dB Release 100 ms Target 0.32 Lookahead 0.75 ms Dither Shaped`
  - GainBoost [0, 30] dB、MaxOutput [-30, 0] dB、Release [0.1, 100] ms、
    Dither ∈ None|Uniform|Triangular|Shaped（None 不做量化，其余 16-bit 量化 + 抖动）。
- `Wide: Intensity 0.354331`
  - Intensity [0, 1]（默认 0.354331，Wide32.c Starting Presets）；
    0 时严格直通（`1+3·0` 侧增益 / `1-0.3·0` 中央补偿均为 1）。

> 四个命令均通过工厂注册进入 `factory_names()`，parser 默认分支**无需静态分发**；
> 参数解析失败返回 `NoMatch`，由 parser 精确分派（`try_create_named`）落到
> `SyntaxError「命令无效」`。

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

pub fn default_smoothing_length(sample_rate: u32) -> u32;   // sample_rate / 100 ≈ 10ms（v6.9，EAPO 对齐）
pub fn short_smoothing_length(sample_rate: u32) -> u32;     // sample_rate / 100 ≈ 10ms
```

> **过渡周期（R4，v6.9）**：默认平滑长度由 `sample_rate/20`（50ms）下调至 `sample_rate/100`（10ms），
> 与 EqualizerAPO 对齐。理由：
> - 过渡窗口内每帧双链处理（双处理模型），10ms 使 CPU 峰值最短、RT 稳定性最好
> - 10ms 升余弦已被 EAPO 多年验证无听感跳变
> - **"乱切也无所谓"**——过渡够短、开销够小，频繁配置变更不再构成负担
> 原 `short_smoothing_length`（10ms）与新的默认值合并为同一公式，保留为别名以便语义区分。

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

**导出给**：`pipeline/dsp/peq_hybrid.rs`、`pipeline/dsp/loudness.rs`、`pipeline/dsp/wide.rs`（原 `peq.rs` / `hp_lp.rs` 已删除；不导出给 `config/`）

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

> **v9.11 起以下各节已作废**：4.13 / 4.14 / 4.16 / 4.17 / 4.18 / 4.19 / 4.20 所描述的文件与实现
> **均已从代码中删除**——PEQ 现为 `pipeline/dsp/peq_hybrid.rs`，双二阶与滤波在 `biquad.rs` /
> `filter.rs`，卷积与延迟已移除。各节标题已加删除线并保留为历史参考；**现行实现见本文件 4.22**。
> 现状文件清单（2026-09）：`aural` `biquad` `compressor` `factory` `filter` `fir` `gain` `loudness`
> `math` `model` `peq_hybrid` `reverb` `specs` `transition` `wide`（共 15 个 `.rs`）。

### 4.13 ~~`pipeline/dsp/peq.rs`~~（已删除）

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

### 4.14 ~~`pipeline/dsp/hp_lp.rs`~~（已删除）

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

### 4.16 ~~`pipeline/dsp/delay.rs`~~（已删除）

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

### 4.17 ~~`pipeline/dsp/copy.rs`~~（已删除）

**职责**：通道复制/混音。

**引用来源**：`crate::pipeline::dsp::filter::Filter`、`crate::pipeline::interleave::*`

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

### 4.18 ~~`pipeline/dsp/graphic_eq.rs`~~（已删除）

> **v9.11 废弃**：`GraphicEQ:` 命令与 `graphic_eq.rs` 已移除，由
> `pipeline/dsp/peq_hybrid.rs`（混合式 PEQ，TOML `[[effects]] type = "peq"`）
> 取代。本节保留为历史记录。

**职责**：图形均衡器（EqualizerAPO 对齐：对数频率插值 + 最小相位 FIR + 直接时域卷积）。

**引用来源**：`crate::pipeline::dsp::filter::Filter`、`crate::pipeline::dsp::convolution::ConvolutionFilter`、`rustfft`

**导出给**：仅 `pipeline/dsp/` 内部

**行为要点（v9.0 建立，v9.5 分块 FFT，v9.6 改回直接 FIR，v9.7 1024 点 + SIMD）**：
- 节点增益在 `log(freq)` 上线性插值，频带外取端点增益；
- 用 cepstrum 生成 **1024 点最小相位 FIR**，`initialize` 时通过 `ConvolutionFilter::with_ir_direct`
  建立直接时域卷积——**无块缓冲**：每个输入采样立即产生输出，
  流停止时不丢尾音（分块 FFT 会把最后 ≤128 采样压在块缓冲被引擎硬停丢弃，
  实测导致“从 M16+ 切换走时嗡一声”，v9.6 实锤后改回直接 FIR）；
- **性能（v9.7）**：直接 FIR 改为“旧→新连续段切片 + 逆序系数点积”，
  AVX2+FMA 8 路向量化（运行时探测，回退标量 mul_add）——1024 点 CPU 低于
  旧 512 点标量实现，200Hz 以下低频分辨率 ≈46.9Hz bin @48k；
- FIR 生成结果按（频段指纹, 采样率）进程内缓存（v9.6），设置页/多流批量实例化时
  不再重复 FFT；
- **不向引擎上报延迟**（GetLatency 无 child 返回 0，与 EAPO 对齐）；
- `GraphicEQ:` **空参数 = 显式移除 EQ**（不产生滤波器，等同 passthrough），
  且产出 spec 指纹（与“无此命令”不同）——热重载据此把旧 EQ 链切换为空链（v9.5）；
- 旧的「多段 biquad 级联」已废弃：Q=1.414 级联会让相邻负增益叠加，实测中心衰减 -11~-12 dB 而非目标 -3 dB。

**公开 API**：

```rust
pub struct GraphicEqFilter { ... }
impl GraphicEqFilter {
    pub fn new(bands: Vec<EqBand>) -> Self;
    pub fn band_count(&self) -> usize;
}
impl Filter for GraphicEqFilter { ... }

pub struct EqBand { pub frequency: f32, pub gain_db: f32 }
pub fn parse_graphic_eq_params(spec: &str) -> Option<Vec<EqBand>>;
```

---

### 4.19 ~~`pipeline/dsp/convolution.rs`~~（已删除）

> **v9.11 废弃**：`convolution.rs` 已删除（不做 IR 卷积；SIMD dot/分块 FFT 迁至
> `fir.rs`）。本节保留为历史参考。

**职责**：卷积（短 IR 直接时域 FIR + 长 IR 分区 FFT，支持调用方注入 IR）。

**引用来源**：`crate::pipeline::dsp::filter::Filter`

**导出给**：`pipeline/dsp/graphic_eq.rs`、`config/commands/convolution.rs`

**v9.0 新增 API**：

```rust
impl ConvolutionFilter {
    /// 使用内存 IR，并强制直接时域 FIR（GraphicEQ 用，无分区块延迟）。
    pub fn with_ir_direct(ir: Vec<f32>, gain_db: f32) -> Self;
}
```

**公开 API**：

```rust
pub struct ConvolutionFilter { ... }
impl ConvolutionFilter {
    pub fn new(path: &str, gain_db: f32) -> Self;
    pub fn with_ir(ir: Vec<f32>, gain_db: f32) -> Self;
    pub fn with_ir_direct(ir: Vec<f32>, gain_db: f32) -> Self;
}
impl Filter for ConvolutionFilter { ... }

/// 解析 `Convolution:` 参数（v7.11 严格化——执行端潜在问题① → 调用方决策）。
///
/// 语法：`<路径> [增益dB]`，路径可为含空格引号路径。
/// - 1 token（裸路径）→ `Ok((path, 0.0))`（gain 默认 0）
/// - 2 tokens（路径 + 数值）→ `Ok((path, gain))`（第 2 个必须 parse 为 f64）
/// - **≥3 tokens / 第 2 个非数值 → `Err(ParseError)`**——不再静默忽略多余 token
///   （`ir.wav -6 abc` 现在报错而非忽略 `abc`；「配置写错必有反馈」，overview/项目概览.md 语法严格性）
pub fn parse_convolution_params(spec: &str) -> Result<(String, f32), ParseError>;
```

> **宽语法回收（v7.11）**：v1 中 `Convolution:` 的宽容解析（前 2 token，多余忽略）
> 曾导致无冒号行被误接——现由 `config 6.1` 单冒号校验（无冒号 → SyntaxError）彻底
> 封堵入口；本函数自身仍保持「路径可含空格」的宽语法（引号路径），但参数**严格计数**。
> `ParseError` 由 config 层包装为 `ConfigError::SyntaxError`（文件 + 行号），最终
> 写入 `log/` 诊断日志（overview/项目概览.md「诊断日志归属」）。

---

### 4.20 ~~`pipeline/dsp/vst.rs`~~（已整体删除）

> **v9.11 起 `vst.rs` 与 VST 宿主整体移除**：本节以下文字提到的 `FACTORY_COUNT`、
> `index::VST_PLUGIN`、`register_builtin_filters`、`VstFactory`、`FilterCreateResult::NoMatch`
> **现均已不存在**（全仓搜索 `vst` 为 0 处）。现行未知 `type` 的处理见 `config 模块规范.md`
> 校验条：模型校验失败 → 整文件拒绝并保留旧链。

> **v9.11 废弃**：`vst.rs` 已整体删除。本节保留为历史参考。

**v9.11 前状态（v6.5 决策 + v7.11 严格化对齐；相关符号现均已删除）**：VST 功能曾回退为 `NoMatch` 模式——配置中出现 `VSTPlugin:` 时，`VstFactory` 曾恒定返回 `FilterCreateResult::NoMatch`。v7.11 起（config 6.1 Unmatched → SyntaxError），解析器**不再静默跳过**：`VSTPlugin:` → `SyntaxError「未知命令 'VSTPlugin'」`，整体解析失败（保留旧链）。
> 语义说明：`VSTPlugin` 仍是**合法命令关键字**（工厂已注册），返回 NoMatch 表示"功能未启用"。v7.11 严格化后与未知关键字同样落 SyntaxError——对用户是**诚实反馈**（此命令当前无效），符合 overview/项目概览.md「配置写错必有反馈」。

**文件内容**：v6.5–v9.11 期间仅保留模块注释；**v9.11 起文件本身已删除**，无任何实现与测试。

**恢复成本（不再是「补一个工厂」）**：TOML 化后不存在可挂载的动态工厂注册表。恢复 VST2/VST3 需
新增 `EffectType` 分支、`pipeline/dsp/factory.rs::create_from_model` 的 match 分支，以及对应的
`Filter` 实现（动态库加载需重新引入 `libloading` 与协议绑定）；`[features] vst` 已随 v9.11 删除。

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

---

### 4.22 `pipeline/dsp/` 效果器（peq/preamp/aural/reverb/compressor/wide/loudness）

> 现行配置模型是 TOML（`config/model.rs` → `ChainModel`），效果器由
> `pipeline/dsp/factory.rs::create_from_model` 按 `EffectType` 静态 `match` 构造。
> 旧的 EAPO 风格 `Key Value` 文本命令解析与 `XxxFactory` 动态注册表已删除；
> 参数范围、键白名单与 PEQ 段数校验全部在 config 层完成（见 `配置与DSP设计.md`）。

`peq_hybrid.rs`：混合式 PEQ——200 Hz 分频，`Fc<200` 段 IIR biquad 级联、
`Fc≥200` 段采样率自适应最小相位 FIR（1024–8192 抽头，≤2048 直接 FIR / >2048 分块 FFT）；
FIR 目标 = 总目标 − IIR 频响（级联精确拟合）。

**来源与许可**：效果器均为独立实现（原创代码，无 AGPL 版权头）——
`reverb.rs`（按 Jon Dattorro 1997 论文）、`compressor.rs`（全声道联动 RMS 压缩器）、
`wide.rs`（线性相位 FIR 分频 + 高频 M/S 去相关 + 空气吸收 + 软限幅，原创实现）、
`aural.rs`（二阶 Butterworth 高通 + 电平跟随 + tanh 奇次软饱和）。
文件直接平铺在 `pipeline/dsp/` 下，无 `fxsound/` 子目录、无 `mod.rs`。

| 文件 | 效果 | `type` |
|------|------|--------|
| `aural.rs` | Aural Enhancer（二阶 Butterworth 高通 + 峰值电平跟随 + tanh 软饱和奇次 + 半波整流偶次，Wet/Dry） | `aural` |
| `reverb.rs` | Dattorro 板式混响（输入 4 级 AllPass 扩散 + 双槽交叉反馈 + 14 抽头输出；`low_cut_hz` 低频瞬态保护） | `reverb` |
| `compressor.rs` | Compressor（全声道联动 RMS 检测 + 软膝静态曲线 + dB 域 attack/release 平滑 + makeup 增益） | `compressor` |
| `wide.rs` | Wide（线性相位 FIR 分频 + 高频 M/S 去相关 + ITD + 空气吸收 + 输出软膝限幅） | `wide` |
| `peq_hybrid.rs` | 混合式 PEQ（IIR biquad 级联 + 最小相位 FIR；段类型 peaking/low_shelf/high_shelf/low_pass/high_pass） | `peq` |
| `gain.rs` / `loudness.rs` | 全局增益 / 等响补偿 | `preamp` / `loudness` |
| `model.rs` / `factory.rs` | `EffectType`/`EffectParams`/spec 指纹 / `create_from_model` 静态分派 | — |

**引用来源**：`crate::pipeline::dsp::filter::Filter`（各 Filter 均实现该 trait）。

**导出给**：`pipeline/dsp/factory.rs::create_from_model`（按 `EffectType` 静态 `match` 构造）；
`config/` 禁止直接引用具体实现。

**RT 约束**：
- `initialize` 预计算系数并分配状态/延迟线（Reverb 缓冲按「当前延迟 + 调制余量 +
  最大抽头」预留，保证小 RoomSize 下输出抽头仍有效；调制深度按论文
  EXCURSION=16 采样@29761Hz 换算）；
- `process` 零分配、无锁、无 I/O、无 panic；输出非有限时置 0；
- `latency()`：由各 Filter 内部实现；但 `pipeline/process.rs` 在初始化时把
  `latency_frames_atomic` 置 0，**对外不上报延迟**（避免帧协商错位）；
- 参数变更走 config 热重载（Filter 重建），不支持流内实时改写。

**公开 API**：

> 各效果器 Filter 由 `pipeline/dsp/factory.rs::create_from_model` 静态构造
> （`match EffectType`），动态工厂 `FilterFactory` / `FilterRegistry` 已删除。
> 参数结构定义在各自文件，`config` 层负责反序列化与校验。

```rust
pub struct AuralParams { pub tune_hz: f32, pub drive: f32, pub odd: f32,
    pub even: f32, pub wet: f32, pub dry: f32 }
pub struct AuralEnhancerFilter { ... }   // impl Filter

pub struct ReverbParams { pub room_size: f32, pub decay: f32, pub damping: f32,
    pub bandwidth: f32, pub density: f32, pub lat5: f32, pub lat6: f32,
    pub pre_delay_ms: f32, pub motion_rate: f32, pub motion_depth: f32,
    pub low_cut_hz: f32, pub wet: f32, pub dry: f32 }
pub struct ReverbFilter { ... }          // impl Filter

pub struct CompressorParams { pub threshold_db: f32, pub ratio: f32, pub knee_db: f32,
    pub attack_ms: f32, pub release_ms: f32, pub makeup_gain_db: f32,
    pub wet: f32, pub dry: f32 }
pub struct CompressorFilter { ... }      // impl Filter

pub struct WideParams { pub gain: f32, pub air: f32, pub air_side: f32,
    pub mix: f32, pub crossover_hz: f32 }
pub struct WideFilter { ... }            // impl Filter

pub struct HybridPeqFilter { ... }       // impl Filter（peq_hybrid.rs）
pub struct GainFilter { ... }            // impl Filter（preamp）
pub struct LoudnessFilter { ... }        // impl Filter（loudness）
```

**默认值**（`Default` 实现；Wet/Dry 均可覆盖）：
- Aural：`tune_hz 1760` / `drive 1.76993` / `odd 1.5` / `even 0.25` / `wet 0.5` / `dry 0.5`；
- Reverb：`room_size 1.0` / `decay 0.41` / `damping 0.408290` / `bandwidth 0.350110` /
  `density 1.0` / `lat5 0.70` / `lat6 0.50` / `pre_delay_ms 0` / `motion_rate 0.110871` /
  `motion_depth 0.63` / `low_cut_hz 100` / `wet 0.27` / `dry 0.73`；
- Compressor：`threshold_db -18` / `ratio 4` / `knee_db 3` / `attack_ms 10` /
  `release_ms 100` / `makeup_gain_db 6` / `wet 1.0` / `dry 0.0`；
- Wide：`gain 0` / `air 0.354331` / `air_side 0` / `mix 0.6` / `crossover_hz 200`。

**关键算法**：
- Aural：二阶 Butterworth 高通（`omega = 2π·tune_hz/sr`）+ 峰值电平跟随
  （瞬时 attack / 指数 release，τ≈120ms，声道共享）；`s = filt/env`，
  `odd = env·tanh(drive·s)`，`even = env·HP20(0.5·(y+|y|))`，Wet/Dry 混合。
- Reverb（Dattorro）：参考采样率 29761 Hz；输入扩散 142/107/379/277；
  槽内 672/908（正交 LFO 调制 APF，深度 16 采样@29761Hz）、4453/4217、1800/2656、3720/3163；
  输出抽头按论文 Table 2；`low_cut_hz` 分频点以下逐声道旁路混响、原样直通（20 ≈ 关闭）。
- Compressor：全声道瞬时 RMS → dBFS 检测电平；静态曲线
  `over = level - threshold`，`slope = 1 - 1/ratio`，软膝带内二次插值；
  增益削减在 dB 域按 attack（压缩增加）/release 平滑；输出乘 makeup 后 Wet/Dry 混合。
- Wide：线性相位 FIR 分频（Kaiser 窗，抽头数随采样率/分频点缩放），低频支路直通；
  Mid 走空气吸收（高频架 + 二阶 Bessel 低通）；Side 增强量由 `gain` 控制
  （0→1×，1→+1.5×），带 10ms/120ms 动态包络与 >1.5kHz 的 ITD 去相关（左 +5 / 右 +7 采样）；
  侧输出再过 `air_side` 空气吸收；处理增量先过截止 = 分频点的一阶高通，再 tanh 限幅、乘 `mix`；
  输出端软膝限幅兜底。`gain/air/air_side` 全为 0 时严格直通，单声道直通。
