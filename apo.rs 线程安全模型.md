## 十、线程安全模型（`object/apo.rs`）

**调用线程**：

| 方法 | 调用线程 | 触发时机 |
|------|---------|---------|
| `CreateInstance` | 应用程序线程 | COM 对象创建 |
| `Initialize` | 应用程序线程 | APO 初始化 |
| `LockForProcess` | 应用程序线程 | 音频格式协商 |
| `UnlockForProcess` | 应用程序线程 | 音频流停止 |
| `GetRegistrationProperties` | 应用程序线程 | 注册属性查询 |
| `IsInputFormatSupported` | 应用程序线程 | 格式协商 |
| `IsOutputFormatSupported` | 应用程序线程 | 格式协商 |
| `GetInputChannelCount` | 应用程序线程 | 通道数查询 |
| `CalcInputFrames` | Windows 音频引擎实时线程 | 帧数计算 |
| `CalcOutputFrames` | Windows 音频引擎实时线程 | 帧数计算 |
| `APOProcess` | Windows 音频引擎实时线程 | 每帧音频处理 |
| `GetLatency` | 应用程序线程 | 延迟查询 |
| `Reset` | 应用程序线程 | 状态重置 |
| `hot_reload` | 配置监控后台线程 | config.toml 变更 |

---

**保护机制**：

可变状态通过多层独立的保护机制管理，不存在嵌套锁：

| 保护对象 | 机制 | 访问路径 |
|----------|------|----------|
| `ApoObjectInner`（chain、transition、pipeline_context、temp_buffers、pending_reload） | `self.mutex: Mutex<ApoObjectInner>` | APOProcess、LockForProcess、UnlockForProcess、Reset、hot_reload |
| `ApoObjectState`（clsid、is_locked、sample_rate、channels、bits_per_sample） | `self.ap_state: Mutex<ApoObjectState>` | LockForProcess、UnlockForProcess、GetRegistrationProperties、GetInputChannelCount |
| 状态机（Created / Initialized / Locked） | `self.state_cell: StateCell`（`AtomicU8` + CAS） | Initialize、LockForProcess、UnlockForProcess、APOProcess（只读检查） |
| 延迟采样数 | `self.latency_samples: AtomicU32` | LockForProcess（写）、GetLatency（读）、Reset（写） |
| 延迟帧数 | `self.latency_frames_atomic: AtomicU32` | v9.12 起恒 0（不上报引擎：实证上报/补偿导致帧协商错位播放卡住）；Lock/Reset 写 0；CalcInputFrames/CalcOutputFrames 读 |

---

**`self.mutex` 互斥保证**：

- `ApoObjectInner` 中的所有字段在同一个 `Mutex` 下保护
- 每个方法获取锁后执行完整操作，然后释放
- APOProcess 持有锁约 1-3ms（取决于 Filter 链复杂度和帧大小）
- LockForProcess 持有锁约 10-100ms（含配置解析和 Chain 构建）
- hot_reload 在锁外解析配置并构建新 Chain（10-100ms），仅在锁内执行 `Box` 指针替换（亚微秒级）
- Reset 持有锁极短（字段清零 + 原子写入）
- APOProcess 与 hot_reload 互斥——如果 hot_reload 恰好在 APOProcess 期间触发，hot_reload 等待 APOProcess 释放锁后执行

**`self.ap_state` 互斥保证**：

- 保护格式和通道状态
- 获取锁时间极短（字段读写级别，微秒以下）
- LockForProcess / UnlockForProcess 中在 `self.mutex` 之后获取 `self.ap_state`，但不同时持有两把锁
- GetRegistrationProperties 和 GetInputChannelCount 仅获取 `self.ap_state`，不持有 `self.mutex`

**`state_cell` 原子操作**：

- 使用 `AtomicU8` + `compare_exchange`（`AcqRel` / `Acquire`）实现无锁状态机
- 不依赖 `Mutex`，无阻塞

**延迟原子变量**：

- `latency_samples` 和 `latency_frames_atomic` 使用 `AtomicU32` 独立保护
- 写入在 `self.mutex` 内完成（LockForProcess / UnlockForProcess / Reset）
- 读取在锁外完成（CalcInputFrames / CalcOutputFrames 读 `latency_frames_atomic`，恒 0）

---

**RT 阻塞可接受性论证**：

- **`self.mutex` 阻塞来源**：
  - LockForProcess：应用程序显式调用，不在播放期间触发
  - hot_reload：锁内仅做 `Box` 指针替换（亚微秒级），配置解析在锁外完成
  - APOProcess 持锁期间不会有其他 APOProcess 竞争（Windows 音频引擎串行调用）
  - Reset：应用程序显式调用，不在播放期间触发
- **`self.ap_state` 阻塞来源**：
  - LockForProcess / UnlockForProcess / Initialize：均不在播放期间触发
  - GetRegistrationProperties / GetInputChannelCount：锁持有时间极短（微秒以下），且在播放开始前或格式协商阶段调用
- **`state_cell`、`latency_samples`、`latency_frames_atomic`**：原子操作，无阻塞
- 对于系统级 APO（采样率 48kHz、缓冲区 128 帧 = 2.67ms 周期），偶尔的亚微秒级 mutex 等待不造成可听问题

---

**重入禁止不变式**：

- **`self.mutex` 不可重入**：任何方法不得在持有 `self.mutex` 的情况下调用同对象的其他需要获取 `self.mutex` 的方法
- **`self.ap_state` 不可重入**：任何方法不得在持有 `self.ap_state` 的情况下调用同对象的其他需要获取 `self.ap_state` 的方法
- **两锁不可交叉**：`self.mutex` 和 `self.ap_state` 在任何执行路径上都不会被同一线程同时持有
- **具体约束**：
  - `CalcInputFrames`、`CalcOutputFrames` 与 `APOProcess` 由 Windows 音频引擎串行调用，不存在重入
  - `CalcInputFrames` / `CalcOutputFrames` 通过 `latency_frames_atomic` 免锁读取延迟值，不获取任何锁
  - `LockForProcess` 内部不得调用 `CalcInputFrames` / `CalcOutputFrames`。v9.12 起延迟不上报（`latency_frames_atomic` 恒 0）；v9.13 同 config/格式的 Relock 复用现有链（`last_lock_key` 指纹，保留滤波器状态）
  - `hot_reload` 内部不得调用需要 `self.mutex` 的方法——整个 `hot_reload` 已持有 `self.mutex`
  - `GetLatency` 在 `self.mutex` 内读取 `pipeline_context.sample_rate`，通过 `latency_samples` 原子加载延迟值，不获取 `self.ap_state`
  - `GetRegistrationProperties` 和 `GetInputChannelCount` 仅获取 `self.ap_state`，不获取 `self.mutex`
- 违反上述不变式在 debug 模式下会触发 `std::sync::Mutex` 的 panic（锁重入）
