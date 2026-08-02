## 六、`config/` 模块规范

**边界**：不知道 `install/`、`object/`。只负责解析配置文件，构建 Filter 链。**不直接操作 `Chain`**。

**允许依赖**：
- `sys/`
- `utils/`
- `pipeline/dsp/filter.rs`（Filter trait + ConfigLoader trait）
- `pipeline/dsp/factory.rs`（FilterFactory、FilterRegistry、DspContext）
- （通道名由调用方注入 `DspContext.channel_names`，config **不直接依赖** `sys/audio_defs`）

**禁止依赖**：
- `install/`、`object/`
- `pipeline/process.rs`、`pipeline/chain.rs`、`pipeline/context.rs`
- **任何 `pipeline/dsp/*.rs` 具体实现文件（如 `peq.rs`、`biquad.rs`）**

---

### 完整模块树

```
config/
├── parser.rs              # 配置文件解析器 + ConfigParser 结构体
├── error.rs               # ConfigError 错误类型
├── watcher.rs             # 配置文件变更监控
├── commands.rs            # 命令工厂入口（注册所有命令工厂）
└── commands/
    ├── channel.rs         # Channel: → 通道选择
    ├── cond.rs            # If:/ElseIf:/Else:/EndIf: → 条件分支系统
    ├── device.rs          # Device: → 设备匹配 + AbortFile
    ├── expr.rs            # Eval: → 变量赋值 + 表达式求值
    ├── include.rs         # Include: → 递归加载子配置文件
    ├── stage.rs           # Stage: → 处理阶段标志
    ├── graphic.rs         # GraphicEQ: → 图形均衡器（通过注册表）
    ├── preamp.rs          # Preamp: → 前级增益（通过注册表）
    ├── copy.rs            # Copy: → 通道复制（通过注册表）
    ├── delay.rs           # Delay: → 延迟（通过注册表）
    ├── filter.rs          # Filter: ON HP/LP/PK/... → 参数滤波器（通过注册表）
    └── rew.rs             # REW 导出格式 → 解析后通过注册表创建
```

---

### 引用约束总表

| 模块 | 可依赖 | 不可依赖 |
|------|--------|----------|
| `config/error.rs` | `std` | 所有其他 |
| `config/parser.rs` | `config/error`、`config/commands/*`、`pipeline/dsp/filter`、`pipeline/dsp/factory`、`sys/registry`、`utils/` | `install/`、`object/`、`pipeline/chain`、`pipeline/process`、`pipeline/context`、任何 `pipeline/dsp/*.rs` 具体实现、`sys/audio_defs` |
| `config/watcher.rs` | `config/error`、`utils/` | 其他 |
| `config/commands.rs` | `config/commands/*`、`pipeline/dsp/factory` | 其他 |
| `config/commands/channel.rs` | `config/error`、`config/parser`(ParseContext)、`pipeline/dsp/filter`(Filter) | `pipeline/chain` |
| `config/commands/cond.rs` | `config/error`、`config/parser`(ParseContext) | `pipeline/` |
| `config/commands/device.rs` | `config/error`、`config/parser`(ParseContext) | `pipeline/` |
| `config/commands/expr.rs` | `config/error` | `pipeline/` |
| `config/commands/include.rs` | `config/error`、`config/parser`(read_config_file, parse_content)、`pipeline/dsp/filter`(ConfigLoader) | `pipeline/chain` |
| `config/commands/stage.rs` | `config/error`、`config/parser`(ParseContext) | `pipeline/` |
| `config/commands/*.rs`（DSP 命令） | `config/error`、`config/parser`(ParseContext)、`pipeline/dsp/filter`(Filter)、`pipeline/dsp/factory`(FilterRegistry) | `pipeline/chain`、具体 Filter 实现 |

---

### 6.0 `config/error.rs`

**职责**：配置解析专用错误类型。

**引用来源**：无外部依赖

**导出给**：`config/` 所有子模块

**公开 API**：

```rust
#[derive(Debug, Clone)]
pub enum ConfigError {
    /// 文件 I/O 错误。
    IoError { path: String, message: String },
    /// 命令语法错误（含文件名和行号）。
    SyntaxError { file: String, line: usize, message: String },
    /// AbortFile 终止（Device: 命令设置）。
    AbortFile,
}

impl std::fmt::Display for ConfigError { ... }
impl std::error::Error for ConfigError {}
```

> `ConditionFalse` 不作为错误返回——条件不满足时跳过行即可，不影响解析流程。

---

### 6.1 `config/parser.rs`

**职责**：配置文件解析器。读取文件、逐行分发到命令处理器，返回 `Vec<Box<dyn Filter>>`，
并产出配置指纹（filter_spec 序列，v7.9——供 object 层热重载判定配置是否实质变化）。

**引用来源**：
- `crate::config::error::ConfigError`
- `crate::config::commands::*`（所有命令处理器）
- `crate::pipeline::dsp::filter::{Filter, DspContext, DeviceType, ProcessingStage as DspProcessingStage}`
- `crate::pipeline::dsp::factory::{FilterFactory, FilterRegistry}`
- （通道名由 `DspContext.channel_names` 注入，config 不直接依赖 `sys/audio_defs`）

**导出给**：`object/apo.rs`、`config/commands/*`

---

#### ConfigParser

```rust
/// 一条成功解析命令的规范化指纹（v7.9，配置变更检测）。
///
/// **单一字符串**：命令名（小写）+ `\x1F` + token 级规范化参数。
/// 仅用于 `==` 等值比较（有序 spec chain 逐项比较）；不可逆向还原 config 原文。
/// 分隔符 `\x1F`（ASCII Unit Separator）——config.txt 文本中不可能出现，杜绝冲突。
pub type FilterSpec = String;

/// 配置指纹（spec chain）：一次完整解析产出的 filter_spec 有序序列。
/// 保持 config.txt 原始行顺序（滤波器串联顺序影响处理结果）。
pub type SpecChain = Vec<FilterSpec>;

/// 配置文件解析器。
///
/// 持有 FilterRegistry，提供文件/字符串/行列表三种解析入口。
/// 返回 `Vec<Box<dyn Filter>>`，由调用方（object/apo.rs）添加到 Chain。
pub struct ConfigParser {
    registry: FilterRegistry,
}

impl ConfigParser {
    pub fn new(registry: FilterRegistry) -> Self;

    /// 解析配置文件。UTF-8 优先，非 UTF-8 使用 ANSI 降级。
    /// 返回 Vec<Box<dyn Filter>>。
    pub fn parse_file(&self, path: &str, ctx: &DspContext) -> Result<Vec<Box<dyn Filter>>, ConfigError>;

    /// 解析配置文件，同时产出配置指纹（v7.9，P0-4 配置变更检测主入口）。
    ///
    /// - 返回 `(滤波器列表, SpecChain)` 双元组；
    /// - 128KB **逐文件**上限：主文件与每个 Include 子文件分别判定
    ///   （`MAX_CONFIG_FILE_SIZE = 128 * 1024`），任一超限 → 整体解析失败；
    /// - Include 子文件在 parser 层递归展开，子文件命令的 filter_spec 内联到
    ///   主 spec chain——子文件变更经「重读主 config → 递归展开」自然反映；
    /// - Include 失败 = 整体解析失败（与语法错误同级，杜绝"残缺 spec 污染基线"）。
    pub fn parse_file_with_spec(&self, path: &str, ctx: &DspContext)
        -> Result<(Vec<Box<dyn Filter>>, SpecChain), ConfigError>;

    /// 解析配置字符串。
    pub fn parse_string(&self, content: &str, ctx: &DspContext) -> Result<Vec<Box<dyn Filter>>, ConfigError>;

    /// 解析行列表。
    pub fn parse_lines(&self, lines: &[String], ctx: &DspContext) -> Result<Vec<Box<dyn Filter>>, ConfigError>;
}
```

---

##### filter_spec 产出契约（v7.9，P0-4）

**原则**：spec 是**文本指纹**，非 "DSP 语义等价"——由 parser 在 **分发层** 统一产出，
不侵入各 handle 内部（各 handle 只保留原解析职责），零 DSP 知识（不剥离单位）。

```rust
/// 产出单条 filter_spec。**统一在分发层调用**（显式 handle 命令与裸注册表命令同走此路径）。
fn produce_spec(cmd: &str, value: &str) -> String {
    let sep = '\x1F';
    if value.is_empty() {
        // 无冒号行（IIR, PK, Fc 1000 Hz, ...）：整行小写 + token 规范化
        normalize_tokens(cmd, sep)
    } else {
        // 有冒号行（Preamp: -6.0 dB）：命令名小写 + 分隔符 + 参数体 token 规范化
        format!("{}{}{}", cmd.to_ascii_lowercase(), sep, normalize_tokens(value, sep))
    }
}

/// token 级规范化：按逗号/空格拆 token；可 parse 为 f64 的经 normalize_number，
/// 其余原样（'dB'、路径 'impulse.wav' 等不变）；用 sep 重新连接。
fn normalize_tokens(s: &str, sep: char) -> String { ... }

/// 数值规范化：去尾零 + 统一有效数字精度（如 6 位），
/// 使 `1000.0`/`1e3`/`1000` 归一为同一 spec。
fn normalize_number(f: f64) -> String { ... }
```

**产出规则**：

| 行格式 | 例子 | 产出 |
|--------|------|------|
| 冒号分隔 | `Preamp: -6.0 dB` | `preamp\x1F-6\x1FdB` |
| 无冒号（裸命令） | `IIR, PK, Fc 1000 Hz, Gain +3.2 dB, Q 1.4` | `iir\x1Fpk\x1F1000\x1F3.2\x1F1.4`（整行 token 规范化） |
| 注释/空行 | `# 注释` | 不产出 |
| 纯配置命令（Device/If/Eval/Stage/Channel） | `Device: AbortFile` | 不产出自身；仅间接影响后续命令 |
| Include | `Include: "sub.txt"` | 不产出自身；递归展开子文件，子文件命令各自产出并内联 |
| DSP 显式 handle（Filter/GraphicEQ/Preamp/Copy/Delay/REW） | `Filter: OFF PK Fc 500 Hz Gain 0 dB Q 1.0` | `filter\x1Foff\x1Fpk\x1F500\x1F0\x1F1`（分发层对**原始行**产出，标准化由 handle 解析后的语义归一） |

> **单位 token 边界**：token 规范化只对可 parse 为 f64 的 token 做 `normalize_number`，
> **不剥离单位**（`-6.0 dB` → `-6\x1FdB` ≠ `-6`）。单位 token 属 DSP 知识，
> 剥离会违背「spec 由 parser 产出、零 DSP 知识」原则。单位差异（`-6.0 dB` vs `-6.0`）
> 触发一次过渡——手动改文件本就是边缘场景，过渡有升余弦无听感爆音，可接受（intent.md）。

**裸命令可达性修正（v7.9，P0-2 遗留补齐）**：`parse_lines_impl` 对裸无冒号命令
（如 `PK Fc 1000`）当前 `split_command_value` 把整行放 cmd、value="" → try_create 收空串
→ Unmatched，**实际到达不了工厂**。v7.9 在分发层补一条：值空且非条件/配置关键字时，
`try_create(cmd)`（整行作参数）。由此裸命令经 `FilterAdded` 分支产出
`produce_spec(cmd, "")` = 整行小写 + token 规范化；`MatchedNoFilter`/`Aborted`/`Unmatched`
**不产出**（与「未创建滤波器则无指纹」一致）。

---

#### ParseContext（解析期间可变状态）

```rust
/// 配置解析上下文。
///
/// 在解析一个配置文件期间维护状态。不暴露给 FilterFactory（工厂使用 DspContext）。
pub struct ParseContext<'a> {
    /// 当前正在构建的过滤器列表。
    pub filters: &'a mut Vec<Box<dyn Filter>>,
    /// 过滤器工厂注册表。
    pub registry: &'a FilterRegistry,
    /// 引擎上下文（只读，供工厂创建 Filter 使用）。
    pub dsp_ctx: &'a DspContext,
    /// 当前处理阶段（解析期间可变，Stage: 命令修改）。
    pub stage: ParseStage,
    /// 当前设备类型（解析期间可变）。
    pub is_capture: bool,
    /// 当前文件路径（用于错误报告和 Include 相对路径）。
    ///
    /// v7.4 修订（P0-2 实现反馈②）：由 `&'a Path` 改为所有权 `PathBuf`——
    /// Include 子解析需独立持有子文件路径，借用无法跨递归层安全表达
    /// （旧方案 `Box::leak` 会导致路径泄漏）。
    pub current_file: PathBuf,
    /// 当前行号（用于错误报告）。
    pub line_number: usize,
    /// AbortFile 标志（Device: 命令设置）。
    pub abort_file: bool,
    /// 条件栈（If/ElseIf/Else/EndIf 嵌套）。
    pub cond_stack: CondStack,
    /// 变量存储（Eval: 命令写入，If: 条件引用）。
    pub variables: Variables,
    /// Include 递归深度。
    pub include_depth: usize,
    /// 当前通道名称子集（Channel: 命令修改）。
    pub current_channels: Vec<String>,
    /// 所有通道名称（来自 DspContext，不可变）。
    pub all_channels: Vec<String>,
}
```

---

#### ParseStage（解析期间阶段标志）

```rust
/// 解析期间的处理阶段。
///
/// 比 DspProcessingStage 多一个 Capture 变体。
/// 解析完成后转换为 DspProcessingStage 传递给 DspContext。
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ParseStage {
    None,
    PreMix,
    PostMix,
    Capture,
}

impl Default for ParseStage {
    fn default() -> Self { Self::None }
}
```

---

#### 解析流程

```rust
/// 读取配置文件内容。
///
/// UTF-8 优先。检测 UTF-8 BOM (EF BB BF) 则跳过。
/// 非 UTF-8 内容使用 ANSI 代码页降级（非 ASCII 字节替换为 '?'）。
pub fn read_config_file(path: &Path) -> Result<String, ConfigError>;

/// 解析文件内容字符串，逐行分发到命令处理器。
///
/// 解析前保存通道名称快照，解析后恢复（Note 51）。
pub fn parse_content(
    content: &str,
    filters: &mut Vec<Box<dyn Filter>>,
    registry: &FilterRegistry,
    dsp_ctx: &DspContext,
    file_path: &Path,
    depth: usize,
) -> Result<(), ConfigError>;

/// 分割 `Command: value` 格式。
fn split_command_value(line: &str) -> (&str, &str);
```

**逐行分发逻辑**：

```rust
for (i, line) in content.lines().enumerate() {
    ctx.line_number = i + 1;
    let trimmed = line.trim();

    if trimmed.is_empty() || trimmed.starts_with('#') { continue; }

    let (command, value) = split_command_value(trimmed);

    // 条件分支始终处理（管理栈状态）
    match command {
        "If"     => { cond::handle_if(value, &mut ctx)?; continue; }
        "ElseIf" => { cond::handle_elseif(value, &mut ctx)?; continue; }
        "Else"   => { cond::handle_else(&mut ctx)?; continue; }
        "EndIf"  => { cond::handle_endif(&mut ctx)?; continue; }
        _ => {}
    }

    // 条件跳过：当前处于 false 分支时跳过其他命令
    if cond::is_skipping(&ctx.cond_stack) { continue; }

    match command {
        "Device"    => device::handle(value, &mut ctx)?,
        "Stage"     => stage::handle(value, &mut ctx)?,
        "Channel"   => channel::handle(value, &mut ctx)?,
        "Eval"      => expr::handle(value, &mut ctx)?,
        "Include"   => include::handle(value, &mut ctx)?,
        // REW 动态命令名：Filter N:（命令关键字为动态 `Filter 1`/`Filter 12` 等，
        // 静态 match 无法命中；v7.4 修订，P0-2 实现反馈③）
        _ if command.to_ascii_lowercase().starts_with("filter ") => {
            rew::handle(value, &mut ctx)?
        }
        // DSP 命令通过 FilterRegistry 匹配
        _ => {
            let outcome = ctx.registry.try_create(value, ctx.dsp_ctx, ...);
            match outcome.result {
                OutcomeKind::FilterAdded(f) => ctx.filters.push(f),
                OutcomeKind::MatchedNoFilter => {},
                OutcomeKind::Aborted => { ctx.abort_file = true; }
                OutcomeKind::Unmatched => {
                    log::warn!("unknown command '{}'", command);
                }
            }
        }
    }

    if ctx.abort_file { return Ok(()); }
}

// 检查条件栈平衡
if !ctx.cond_stack.is_empty() {
    return Err(ConfigError::SyntaxError { ... "unterminated If: block" });
}
```

---

### 6.2 `config/watcher.rs`

**职责**：配置文件变更监控（**Win32 事件驱动**，v7.8 修订——对齐 EAPO `notificationThread`）。
**不再使用轮询模式**（旧 2000ms 轮询延迟高、浪费 CPU）。

**引用来源**：
- `crate::config::error::ConfigError`
- `crate::utils::vx_error::VxApoError`
- `windows::Win32::Storage::FileSystem::{FindFirstChangeNotificationW, FindNextChangeNotification, FindCloseChangeNotification}`
- `windows::Win32::System::Threading::{WaitForMultipleObjects, WaitForSingleObject, CreateEventW, SetEvent}`
- `windows::Win32::Foundation::{HANDLE, WAIT_OBJECT_0}`

**导出给**：`object/apo.rs`

**公开 API**：

```rust
/// 监控到的变更事件类型。
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum WatchEvent {
    /// 监控目录内发生变更（**目录级通知**——`FindFirstChangeNotificationW`
    ///   不提供具体文件名（v7.9 澄清），无法逐文件过滤）。
    /// 触发方（object/apo.rs hot_reload）重新解析 config.txt，经 spec 指纹
    /// 比对决定是否真正切换（内容未变 → 幂等跳过，无听感副作用）。
    DirectoryChanged(PathBuf),    // watch_dir
    RegistryChanged,              // 保留：poll_registry 哈希兜底（低频）
}

/// 配置目录变更监控器（v7.8/v7.9 事件驱动）。
///
/// 核心：监控 **目录**（非文件——文件被删除重建时句柄失效，目录天然健壮）。
/// 线程模型：
///   - watcher 线程（控制路径）：FindFirstChangeNotificationW 阻塞等待变更
///   - 去重窗口 10ms（对齐 EAPO：FindNextChangeNotification 后 WaitFor(1,10ms)
///     合并编辑器「写临时文件 + rename」的多次通知）
///   - 退出：shutdown_event 置位 → 线程退出（UnlockForProcess 时 SetEvent + join，v7.9）
pub struct ConfigWatcher { ... }

impl ConfigWatcher {
    /// 创建监控器并启动 watcher 线程。
    ///
    /// - watch_dir：**监控目录**（如 `Documents\VxAPO\{GUID}`），非 config.txt 文件本身
    /// - shutdown_event：外部持有的退出事件（由 APO 实例持有；UnlockForProcess 时
    ///   `SetEvent` 后 join 线程——v7.9 生命周期随锁定周期）
    /// - 去重窗口固定 10ms（v7.8，对齐 EAPO 实战；旧 500ms 过保守，延迟过高）
    ///
    /// 线程内循环：
    ///   1. FindFirstChangeNotificationW(watch_dir, TRUE, FILE_NAME | LAST_WRITE)
    ///   2. WaitForMultipleObjects(shutdown_event | notification_handle)  → 异步等事件（非轮询）
    ///   3. 变更（**目录级，不区分文件**）→ 触发回调（object 层 hot_reload 处理）
    ///   4. FindNextChangeNotification → WaitForMultipleObjects(1, handle, 10ms)【去重】
    ///   5. 触发回调（object 层 hot_reload；spec 指纹短路 + 加载闸门由 reloading 防覆盖）
    pub fn new(watch_dir: PathBuf, shutdown_event: HANDLE) -> Self;

    /// 等待并处理一个事件（阻塞直到文件变更或 shutdown）。
    /// 返回 false 表示 shutdown（线程应退出）。
    pub fn wait_and_handle(&mut self) -> bool;

    /// 检查注册表变更（哈希比对，低频；与目录监控并行）。
    pub fn poll_registry(&mut self, current_hash: u64) -> Option<WatchEvent>;

    /// 停止监控（SetEvent + join watcher 线程）。v7.9：仅在 UnlockForProcess 调用；
    /// 重新 Lock 时重新创建实例。
    pub fn shutdown(self);
}
```

> **目录级语义（v7.9 澄清，P0-4）**：`FindFirstChangeNotificationW` 是**目录级通知**——
> 只告知"监控目录下有变更"，**不提供具体文件名**（文件名信息只有 `ReadDirectoryChangesW`
> 扩展才有）。因此 `WatchEvent::ConfigFileChanged(PathBuf)` / 文件名校验（旧 6.2）
> **无法实现**，v7.9 移除，统一为 `DirectoryChanged(watch_dir)`。
> object 层 `hot_reload`（7.1.18）对任何目录变更：128KB 闸门 → 重新解析 →
> spec 指纹比对（与 active_spec 逐项比较）——内容未变（无关文件/注释/重存）幂等跳过。
>
> **生命周期（v7.9）**：`ConfigWatcher` 由 `ApoObject.watcher: Option<ConfigWatcher>` 持有，
> `LockForProcess` 末尾创建（7.1.9）、`UnlockForProcess` `SetEvent` + join（7.1.10）。
> unlocked 期间无音频流，配置热重载无意义；重新 Lock 必然重读 config 建立新基线。
>
> **旧轮询模式废弃（v7.8）**：`poll()` / `should_poll()` / `poll_interval_ms` / `Deduplicator(500ms)`
> 移除——事件驱动 + 10ms 去重启用了相同防雨强语义，但延迟从 2000ms 级降至 10ms 级且无空闲 CPU。
> 仅 `poll_registry`（哈希比对）保留，因配置目录监控不依赖注册表、频次极低，无实时性要求。

---

### 6.3 `config/commands.rs`

**职责**：命令工厂入口。注册所有命令工厂到 FilterRegistry。

**引用来源**：
- 所有 `config/commands/*.rs` 子模块
- `crate::pipeline::dsp::factory::FilterRegistry`
- `crate::pipeline::dsp::register_builtin_filters`

**导出给**：`config/parser.rs`、`object/apo.rs`

**公开 API**：

```rust
/// 注册内置 DSP 过滤器工厂到 FilterRegistry。
///
/// v7.4 修订（P0-2 实现反馈①）：**只注册 DSP 工厂**——纯配置语义命令
/// （Device:/If:/Eval:/Include:/Stage:/Channel:/Rew）由 `config/parser.rs`
/// 的逐行分发逻辑（6.1）**静态分发**，不注册进 FilterRegistry。
///
/// 原因：FilterFactory::create_filter 只接收**冒号后的 value**（不含命令关键字），
/// config 命令工厂无法从 value 反推命令名（`Filter:` 的 value 是 `ON PK...`、
/// `Device:` 的 value 是设备路径），注册后永远无法命中。
/// DSP 命令（Preamp:/Filter:/GraphicEQ:/Copy:/Delay: 等）仍通过 registry 动态创建。
pub fn register_all_commands(registry: &mut FilterRegistry) {
    // DSP 工厂由 pipeline/dsp 统一注册（完全下沉）
    crate::pipeline::dsp::register_builtin_filters(registry);
}
```

> **分发权威**：纯配置命令的分发以 6.1 逐行分发逻辑为准（静态 `match` + REW `Filter N:` 前缀分支）；
> `FilterRegistry` 仅承载 DSP 工厂，作为未知/动态参数的兜底创建路径。
> 不再有"config 命令工厂注册进 registry"的意图；`config/commands.rs` 退化为 DSP 工厂注册入口。

---

### 6.4 `config/commands/channel.rs`

**职责**：解析 `Channel:` 命令。创建 `ChannelFilter` 设置后续过滤器操作的通道子集。

**引用来源**：
- `crate::config::error::ConfigError`
- `crate::config::parser::ParseContext`
- `crate::pipeline::dsp::filter::Filter`
- （通道名由 `DspContext.channel_names` **注入**，config 不直接依赖 `sys/audio_defs`）

**导出给**：`config/commands.rs`

**语法**：
- `Channel: L R C LFE` — 仅操作指定通道
- `Channel: *` — 操作所有通道（重置为 allChannels）

**语义**：解析通道名列表，创建 `ChannelFilter`。Filter 的 `initialize()` 方法将通道名设置到 Chain 上（通过 `Filter.initialize()` 返回 `Some(channel_names)` 触发通道选择）。

**实现要点**：
- 验证每个通道名存在于 `allChannels` 中
- `ChannelFilter.initialize()` 返回 `Some(names)` 表示通道选择生效
- 通道变更时清除映射缓存

**禁止**：不直接操作 `Chain`，通过 `Filter` trait 间接实现

---

### 6.5 `config/commands/cond.rs`

**职责**：If:/ElseIf:/Else:/EndIf: 条件分支系统。

**引用来源**：
- `crate::config::error::ConfigError`
- `crate::config::parser::{ParseContext, ParseStage}`

**导出给**：`config/commands.rs`、`config/parser.rs`（parser 直接调用条件处理函数）

**语法**：

```text
If: <condition>
  ... (条件为 true 时执行)
ElseIf: <condition>
  ... (前一条件为 false 时检查)
Else:
  ... (所有条件均为 false 时执行)
EndIf:
```

**条件表达式**：
- `variable == value` / `!=` / `>` / `<` / `>=` / `<=`
- `variable`（非零为 true）
- `!variable`（零为 true）
- `true` / `false` 字面量
- `device_type == render` / `capture`
- `stage == premix` / `postmix` / `capture`

**嵌套支持**：If/EndIf 可嵌套，内部 If 在外层跳过时不计数（只压栈不评估条件）。

---

#### CondState / CondStack

```rust
/// 单层条件状态。
#[derive(Debug, Clone)]
pub struct CondState {
    /// 本层是否正在执行（条件为 true 且外层也允许执行）。
    pub executing: bool,
    /// 本层是否已经有过一个 true 分支（If 或 ElseIf 命中）。
    /// 一旦为 true，后续 ElseIf / Else 跳过。
    pub had_true: bool,
}

/// 条件栈。
pub type CondStack = Vec<CondState>;
```

---

#### 处理函数

```rust
/// 检查当前是否在跳过状态（栈中任何一层 executing == false）。
pub fn is_skipping(stack: &CondStack) -> bool;

/// 处理 If: 命令。外层已跳过时不评估条件，直接压入 false。
pub fn handle_if(value: &str, ctx: &mut ParseContext) -> Result<(), ConfigError>;

/// 处理 ElseIf: 命令。已有 true 分支时跳过。
pub fn handle_elseif(value: &str, ctx: &mut ParseContext) -> Result<(), ConfigError>;

/// 处理 Else: 命令。已有 true 分支时 executing = false。
pub fn handle_else(ctx: &mut ParseContext) -> Result<(), ConfigError>;

/// 处理 EndIf: 命令。弹出栈顶。
pub fn handle_endif(ctx: &mut ParseContext) -> Result<(), ConfigError>;
```

---

#### 条件求值

```rust
/// 求值条件表达式。
///
/// 支持：true/false 字面量、取反（!）、比较运算、
/// device_type == render/capture、stage == premix/postmix/capture、
/// 单变量（非零为 true）。
fn eval_condition(expr: &str, ctx: &ParseContext) -> Result<bool, ConfigError>;
```

---

### 6.6 `config/commands/device.rs`

**职责**：解析 `Device:` 命令。

**引用来源**：
- `crate::config::error::ConfigError`
- `crate::config::parser::ParseContext`

**导出给**：`config/commands.rs`

**语法**：
- `Device: <path>` — 绑定配置到指定设备路径
- `Device: AbortFile` — 终止当前文件解析（不终止 Include 的父文件）

**语义**：`AbortFile` 设置 `ctx.abort_file = true` 并返回 `Err(ConfigError::AbortFile)`。设备路径匹配由调用方在解析前通过 `DspContext.device_type` 预处理。

---

### 6.7 `config/commands/expr.rs`

**职责**：解析 `Eval:` 命令。定义变量，存入 `ParseContext.variables`。

**引用来源**：
- `crate::config::error::ConfigError`
- `crate::config::parser::ParseContext`

**导出给**：`config/commands.rs`

**语法**：
- `Eval: gain = -3.0` — 常量赋值
- `Eval: x = 1 + 2 * 3` — 算术运算
- `Eval: y = db_to_linear(-6)` — 函数调用
- `Eval: z = x + 1` — 变量引用

**实现**：简易递归下降解析器。

---

#### 变量存储

```rust
/// 变量存储类型。
pub type Variables = HashMap<String, f64>;
```

---

#### 表达式求值

```rust
/// 处理 Eval: 命令。
///
/// 格式：`variable = expression`
/// 解析表达式，将结果存入 ctx.variables。
pub fn handle(value: &str, ctx: &mut ParseContext) -> Result<(), ConfigError>;
```

**支持的运算符**：`+`、`-`、`*`、`/`（标准优先级），括号

**支持的函数**：`abs`、`sqrt`、`sin`、`cos`、`tan`、`log`/`ln`、`log10`、`floor`、`ceil`、`round`、`db_to_linear`、`linear_to_db`

**内置常量**：`pi`、`e`

---

### 6.8 `config/commands/include.rs`

**职责**：解析 `Include:` 命令。递归加载子配置文件。

**引用来源**：
- `crate::config::error::ConfigError`
- `crate::config::parser::{read_config_file, parse_content, ParseContext}`
- `crate::pipeline::dsp::filter::ConfigLoader`（可选，用于解耦回调）

**导出给**：`config/commands.rs`

**语法**：`Include: "presets/default.txt"`

**语义**：递归加载子配置文件。路径相对于当前文件所在目录。最大递归深度 16 层。

**实现**：

```rust
/// 最大 Include 递归深度。
pub const MAX_INCLUDE_DEPTH: usize = 16;

/// 处理 Include: 命令。
///
/// 1. 检查递归深度限制
/// 2. 解析相对路径（相对于当前文件目录）
/// 3. 检查文件存在
/// 4. 读取文件内容
/// 5. 调用 parse_content 解析（depth + 1）
pub fn handle(value: &str, ctx: &mut ParseContext) -> Result<(), ConfigError>;
```

---

### 6.9 `config/commands/stage.rs`

**职责**：解析 `Stage:` 命令。

**引用来源**：
- `crate::config::error::ConfigError`
- `crate::config::parser::{ParseContext, ParseStage}`

**导出给**：`config/commands.rs`

**语法**：`Stage: PreMix | PostMix | Capture`

**语义**：设置当前处理阶段标志。支持多种别名：
- `PreMix` / `pre-mix` / `pre_mix`
- `PostMix` / `post-mix` / `post_mix`
- `Capture` / `capture`

---

### 6.10 `config/commands/graphic.rs`

**职责**：解析 `GraphicEQ:` 命令。

**引用来源**：
- `crate::config::error::ConfigError`
- `crate::config::parser::ParseContext`
- `crate::pipeline::dsp::filter::Filter`
- `crate::pipeline::dsp::factory::FilterRegistry`

**导出给**：`config/commands.rs`

**语法**：`GraphicEQ: 20 -3.1; 25 -3.1; ...`

**语义**：解析频段参数，通过 `registry.try_create("GraphicEQ", ...)` 动态创建。不直接 import `graphic_eq.rs`。

---

### 6.11 `config/commands/preamp.rs`

**职责**：解析 `Preamp:` 命令。

**引用来源**：
- `crate::config::error::ConfigError`
- `crate::config::parser::ParseContext`
- `crate::pipeline::dsp::factory::FilterRegistry`

**导出给**：`config/commands.rs`

**语法**：`Preamp: -6.0 dB`

---

### 6.12 `config/commands/copy.rs`

**职责**：解析 `Copy:` 命令。

**引用来源**：
- `crate::config::error::ConfigError`
- `crate::config::parser::ParseContext`
- `crate::pipeline::dsp::factory::FilterRegistry`

**导出给**：`config/commands.rs`

**语法**：`Copy: L2=L R2=R`

---

### 6.13 `config/commands/delay.rs`

**职责**：解析 `Delay:` 命令。

**引用来源**：
- `crate::config::error::ConfigError`
- `crate::config::parser::ParseContext`
- `crate::pipeline::dsp::factory::FilterRegistry`

**导出给**：`config/commands.rs`

**语法**：`Delay: 500 ms`

---

### 6.14 `config/commands/filter.rs`

**职责**：解析 `Filter: ON HP/LP/PK/...` 命令。

**引用来源**：
- `crate::config::error::ConfigError`
- `crate::config::parser::ParseContext`
- `crate::pipeline::dsp::filter::Filter`
- `crate::pipeline::dsp::factory::FilterRegistry`

**导出给**：`config/commands.rs`

**语法**：`Filter: ON PK Fc 1000 Hz Gain +3.0 dB Q 1.0`

**语义**：解析 `ON/OFF` 开关和滤波器类型，通过 `registry.try_create(type, args, ctx)` 动态创建。`OFF` 标志由 ParseContext 中的启用标志控制（创建 PassthroughFilter）。

**支持类型**：PK、LP、HP、LS、HS、AP、NO、Modal

---

### 6.15 `config/commands/rew.rs`

**职责**：解析 REW 导出格式。

**引用来源**：
- `crate::config::error::ConfigError`
- `crate::config::parser::ParseContext`
- `crate::pipeline::dsp::filter::Filter`
- `crate::pipeline::dsp::factory::FilterRegistry`

**导出给**：`config/commands.rs`

**语法**：`Filter 1: ON PK Fc 50,0 Hz Gain -10,0 dB Q 2,50`

**语义**：解析 REW Room EQ V5 格式行（含逗号小数点），转换为标准参数后通过 `registry.try_create("PK", ...)` 创建。