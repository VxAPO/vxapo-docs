## 六、`config/` 模块规范

**边界**：不知道 `install/`、`object/`。只负责解析配置文件，构建 Filter 链。**不直接操作 `Chain`**。

**允许依赖**：
- `sys/`
- `utils/`
- `pipeline/dsp/filter.rs`（Filter trait + ConfigLoader trait）
- `pipeline/dsp/factory.rs`（FilterFactory、FilterRegistry、DspContext）
- `pipeline/channel.rs`（`default_channel_mask`、`get_channel_names`）

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
| `config/parser.rs` | `config/error`、`config/commands/*`、`pipeline/dsp/filter`、`pipeline/dsp/factory`、`pipeline/channel`、`sys/registry`、`utils/` | `install/`、`object/`、`pipeline/chain`、`pipeline/process`、`pipeline/context`、任何 `pipeline/dsp/*.rs` 具体实现 |
| `config/watcher.rs` | `config/error`、`utils/` | 其他 |
| `config/commands.rs` | `config/commands/*`、`pipeline/dsp/factory` | 其他 |
| `config/commands/channel.rs` | `config/error`、`config/parser`(ParseContext)、`pipeline/dsp/filter`(Filter)、`pipeline/channel` | `pipeline/chain` |
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

**职责**：配置文件解析器。读取文件、逐行分发到命令处理器，返回 `Vec<Box<dyn Filter>>`。

**引用来源**：
- `crate::config::error::ConfigError`
- `crate::config::commands::*`（所有命令处理器）
- `crate::pipeline::dsp::filter::{Filter, DspContext, DeviceType, ProcessingStage as DspProcessingStage}`
- `crate::pipeline::dsp::factory::{FilterFactory, FilterRegistry}`
- `crate::pipeline::channel::*`

**导出给**：`object/apo.rs`、`config/commands/*`

---

#### ConfigParser

```rust
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

    /// 解析配置字符串。
    pub fn parse_string(&self, content: &str, ctx: &DspContext) -> Result<Vec<Box<dyn Filter>>, ConfigError>;

    /// 解析行列表。
    pub fn parse_lines(&self, lines: &[String], ctx: &DspContext) -> Result<Vec<Box<dyn Filter>>, ConfigError>;
}
```

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
    pub current_file: &'a Path,
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

**职责**：配置文件变更监控。

**引用来源**：
- `crate::config::error::ConfigError`
- `crate::utils::vx_error::VxApoError`

**导出给**：`object/apo.rs`

**公开 API**：

```rust
/// 监控到的变更事件类型。
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum WatchEvent {
    ConfigFileChanged(PathBuf),
    ConfigFileDeleted(PathBuf),
    RegistryChanged,
}

/// 事件去重器（Note 50：500ms 窗口内相同事件只触发一次）。
pub struct Deduplicator;

/// 配置目录变更监控器。
///
/// 轮询模式：每 poll_interval_ms 毫秒扫描一次目录。
/// 支持文件修改时间比对 + 注册表变更哈希比对。
pub struct ConfigWatcher { ... }

impl ConfigWatcher {
    /// 创建监控器。
    /// - watch_dir：监控目录
    /// - poll_interval_ms：轮询间隔（默认 2000ms）
    /// - dedup_window_ms：去重窗口（默认 500ms）
    pub fn new(watch_dir: PathBuf, poll_interval_ms: u64, dedup_window_ms: u64) -> Self;

    /// 执行一次轮询扫描，返回去重后的变更事件列表。
    pub fn poll(&mut self) -> Vec<WatchEvent>;

    /// 检查注册表变更（传入当前哈希，与上次比对）。
    pub fn poll_registry(&mut self, current_hash: u64) -> Option<WatchEvent>;

    /// 是否到达轮询时间。
    pub fn should_poll(&self) -> bool;
}
```

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
/// 注册所有命令工厂和内置 DSP 过滤器工厂。
pub fn register_all_commands(registry: &mut FilterRegistry) {
    // DSP 工厂由 pipeline/dsp 统一注册（完全下沉）
    crate::pipeline::dsp::register_builtin_filters(registry);

    // 纯配置语义命令由 config 自己注册
    registry.register(DeviceFactory::new());      // Device:
    registry.register(IfFactory::new());           // If:
    registry.register(EvalFactory::new());         // Eval:
    registry.register(IncludeFactory::new());      // Include:
    registry.register(StageFactory::new());        // Stage:
    registry.register(ChannelFactory::new());      // Channel:
    registry.register(RewFactory::new());          // REW 格式
}
```

**工厂优先级**：`config/` 注册的工厂排在 `pipeline/dsp/` 工厂之前（`Device:` / `If:` 等需优先匹配）。

---

### 6.4 `config/commands/channel.rs`

**职责**：解析 `Channel:` 命令。创建 `ChannelFilter` 设置后续过滤器操作的通道子集。

**引用来源**：
- `crate::config::error::ConfigError`
- `crate::config::parser::ParseContext`
- `crate::pipeline::dsp::filter::Filter`
- `crate::pipeline::channel::*`

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