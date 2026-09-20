## 六、`config/` 模块规范

**边界**：不知道 `install/`、`object/`。只负责解析配置文件，构建 Filter 链。**不直接操作 `Chain`**。

**允许依赖**：
- `sys/`
- `utils/`
- `pipeline/dsp/filter.rs`（Filter trait + DspContext）
- `pipeline/dsp/model.rs`（ChainModel / EffectType，v9.11 转换目标）
- `pipeline/dsp/factory.rs`（`create_from_model` 静态分派，v9.11）
- （通道名由调用方注入 `DspContext.channel_names`，config **不直接依赖** `sys/audio_defs`）

**禁止依赖**：
- `install/`、`object/`
- `pipeline/process.rs`、`pipeline/chain.rs`、`pipeline/context.rs`
- **任何 `pipeline/dsp/*.rs` 具体实现文件（如 `peq_hybrid.rs`、`wide.rs`）**

---

### 完整模块树

```
config/
├── parser.rs              # TOML 配置解析（FileModel → ChainModel → 链）
├── model.rs               # FileModel（serde，含 name/group/meta APP 元数据）
├── error.rs               # ConfigError 错误类型
├── watcher.rs             # 配置文件变更监控
```

### 6.0 v9.11 TOML 配置模型（现行，取代旧逐行命令）

> v9.11 起配置文件为 `config.toml`（per-device `C:\ProgramData\VxAPO\{GUID}\config.toml`）。
> 旧 `config.txt` 逐行命令体系（`config/commands/*`、EAPO 对齐、GraphicEQ: 等）
> 全部移除；迁移工具 `vxapo-cli config convert` 做一次性转换。

**双模型分层（定稿）**：
- `FileModel`（`config/model.rs`）：TOML 反序列化目标，含 `version` /
  顶层 `enabled`（v9.18 总开关，false = 整链 passthrough）/ `[meta]` /
  `[[effects]]`（`name`/`group` 为 APP 元数据）；
- `ChainModel`（`pipeline/dsp/model.rs`）：纯 DSP 语义模型（类型、参数、
  `enabled`、`channels`），无 APP 元数据；
- 转换（`FileModel → ChainModel`，含范围/段数/声道名校验）是 **config 层
  职责**；依赖方向保持 `config → pipeline/dsp`。

**Schema（v1）**：

```toml
version = 1

[meta]
app = "vxapo"
schema = 1

[[effects]]
type = "peq"
name = "脚步声增强"          # 可选，APP 元数据，driver 忽略
group = "FPS 预设"           # 可选，APP 元数据，driver 忽略
crossover_hz = 200
[[effects.bands]]
fc = 1000
gain_db = 3.0
q = 1.0
```

**效果器类型（v1 保留集）**：`peq`（混合式，见 `pipeline 4.22`）、`preamp`、
`aural`、`reverb`、`compressor`、`wide`、`loudness`；`enabled = false` 旁路；
`channels = ["FL", "FR"]` 限定作用声道（缺省全部）。旧 `maximizer` / `leveler`
类型与旧参数键被接受但忽略，映射为 `compressor` 默认参数。

**校验**：未知 type / 未知键 / 不适用字段 / 缺必填键 / 超范围 / PEQ 段数
不在 [1, 31]（全局合计 ≤ 31）/ 声道名重复或不存于设备 → 整文件解析失败，保留旧链。

**指纹与热重载**：spec 指纹 = `EffectConfig::spec()`（DSP 字段稳定序列化，
`name`/`group`/`meta` 不参与）；watcher 监控 `config.toml`。
```

---

### 引用约束总表

> v9.17（单一事实源）：本表不再独立维护——以主规范
> `模块引用规范（无详细模块版）.md` 第十一节「引用约束总表」为唯一基线，
> config 各文件的允许/禁止依赖逐行见主规范。

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
    /// TOML 反序列化失败（v9.11）。
    TomlError { file: String, message: String },
    /// FileModel → ChainModel 转换/校验失败（v9.11）。
    ModelError { file: String, message: String },
    /// AbortFile 终止（Device: 命令设置）。
    AbortFile,
}

impl std::fmt::Display for ConfigError { ... }
impl std::error::Error for ConfigError {}
```

> `ConditionFalse` 不作为错误返回——条件不满足时跳过行即可，不影响解析流程。

---

### 6.1 `config/parser.rs`

**职责**：配置文件解析器（v9.11 TOML）。读取 `config.toml` → `toml::from_str::<FileModel>`
→ `into_chain_model`（校验）→ `factory::create_from_model` 构造 Filter 链，并产出配置指纹
（`EffectConfig::spec()` 序列——供 object 层热重载判定配置是否实质变化）。

**引用来源**：
- `crate::config::error::ConfigError`
- `crate::config::model::FileModel`
- `crate::pipeline::dsp::model::{ChainModel, EffectType}`
- `crate::pipeline::dsp::filter::{Filter, DspContext}`
- `crate::pipeline::dsp::factory::create_from_model`
- （通道名由 `DspContext.channel_names` 注入，config 不直接依赖 `sys/audio_defs`）

**导出给**：`object/apo.rs`（Lock/热重载）

---

#### ConfigParser

```rust
/// 配置指纹（v9.11）：`EffectConfig::spec()` 稳定序列化（DSP 字段，
/// `name`/`group`/`meta` 不参与）。有序 spec chain 逐项比较用于热重载判定。
pub type FilterSpec = String;

/// 配置指纹（spec chain）：一次完整解析产出的 filter_spec 有序序列
/// （`[[effects]]` 数组表顺序即处理顺序）。
pub type SpecChain = Vec<FilterSpec>;

/// 配置文件解析器。
/// v9.11：无状态（模型校验在 config/model.rs），TOML → FileModel → ChainModel
/// → `factory::create_from_model` 构造 Filter 链。
pub struct ConfigParser;

impl ConfigParser {
    pub fn new() -> Self;

    /// 解析配置文件（config.toml）。缺失 = passthrough（v9.12）；
    /// 语法/模型校验失败 → 整文件失败（调用方按场景降级：Lock → passthrough，
    /// 热重载 → 保留旧链）。
    /// 返回 Vec<Box<dyn Filter>>。
    pub fn parse_file(&self, path: &str, ctx: &DspContext) -> Result<Vec<Box<dyn Filter>>, ConfigError>;

    /// 解析配置文件，同时产出配置指纹（v7.9，P0-4 配置变更检测主入口）。
    /// 返回 `(滤波器列表, SpecChain)` 双元组；128KB 上限 `MAX_CONFIG_FILE_SIZE`。
    pub fn parse_file_with_spec(&self, path: &str, ctx: &DspContext)
        -> Result<(Vec<Box<dyn Filter>>, SpecChain), ConfigError>;

    /// 解析配置字符串（TOML）。
    pub fn parse_string(&self, content: &str, ctx: &DspContext) -> Result<Vec<Box<dyn Filter>>, ConfigError>;

    /// 解析行列表（兼容入口：按换行拼接后走 TOML 解析）。
    pub fn parse_lines(&self, lines: &[String], ctx: &DspContext) -> Result<Vec<Box<dyn Filter>>, ConfigError>;
}
```

---

##### filter_spec 产出契约（v7.9，P0-4；v9.11 起为 `EffectConfig::spec()`）

**原则**：spec 是**文本指纹**，非 "DSP 语义等价"——v9.11 起由模型层
`EffectConfig::spec()` 统一产出（type/enabled/channels/全部参数，`{:.6}` 定精度），
`name`/`group`/`meta` 不参与（APP 元数据变化不触发热重载）。

```rust
/// 产出单条 filter_spec。**统一在分发层调用**（显式 handle 命令与裸注册表命令同走此路径）。
fn produce_spec(cmd: &str, value: &str) -> String {
    let sep = '\x1F';
    if value.is_empty() {
        // v7.11：无冒号行已提前 Err（split_command_value 单冒号校验），
        // 此分支保留为防御（正常不会再走到）。
        normalize_tokens(cmd, sep)
    } else {
        // 有冒号行（Preamp: -6.0 dB）：命令名小写 + 分隔符 + 参数体 token 规范化
        format!("{}{}{}", cmd.to_ascii_lowercase(), sep, normalize_tokens(value, sep))
    }
}
```

> **单位 token 边界**：token 规范化只对可 parse 为 f64 的 token 做 `normalize_number`，
> **不剥离单位**（`-6.0 dB` → `-6\x1FdB` ≠ `-6`）。单位 token 属 DSP 知识，
> 剥离会违背「spec 由 parser 产出、零 DSP 知识」原则。单位差异（`-6.0 dB` vs `-6.0`）
> 触发一次过渡——手动改文件本就是边缘场景，过渡有升余弦无听感爆音，可接受（overview/项目概览.md）。

**裸命令可达性修正反转（v7.11，执行端潜在问题② → 产品决策）**：
> **无冒号行不应被解析**（overview/项目概览.md「语法严格性」：冒号前字符串决定解析目标，必须是严格关键字）。
> **EAPO 行为事实（v8.3 补注，S5）**：EAPO 对无冒号行是**静默跳过**（FilterEngine.cpp 329-330：
> `pos = line.find(':')`，`pos==-1` 时整行不解析、无错误）——**VxAPO「拒绝报错」比 EAPO 更严格**，
> 属**有意差异**（intent「输入严格保证解析宽容」：写错必有反馈），非对齐。详见 `Equalizer 行为文档.md` C23。
> 因此 **v7.9 的裸命令可达性修正删除**：
> - `split_command_value` 零冒号 → `SyntaxError「缺少冒号」`（整体失败），不落 registry、不产出 spec；
> - 效果：`BogusCommand`（无冒号）→ 明确「缺少冒号」错误，而非被 Convolution 宽容语义
>   误接为「路径加载失败」；`Convolution: ir.wav -6` 只能通过显式冒号触发。
> - `produce_spec(cmd, "")` 分支保留为防御（正常不会再走到——零冒号已提前 Err）。
> - 旧 v7.9 文本「值空且非条件/配置关键字时 try_create(cmd)（整行作参数）」**废弃**。

```rust
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
| 注释/空行 | `# 注释` | 不产出 |
| 纯配置命令（Device/If/Eval/Stage/Channel） | `Device: AbortFile` | 不产出自身；仅间接影响后续命令 |
| Include | `Include: "sub.txt"` | 不产出自身；递归展开子文件，子文件命令各自产出并内联 |
| DSP 显式 handle（Filter/GraphicEQ/Preamp/Copy/Delay/REW） | `Filter: OFF PK Fc 500 Hz Gain 0 dB Q 1.0` | `filter\x1Foff\x1Fpk\x1F500\x1F0\x1F1`（分发层对**原始行**产出，标准化由 handle 解析后的语义归一） |

> **单位 token 边界**：token 规范化只对可 parse 为 f64 的 token 做 `normalize_number`，
> **不剥离单位**（`-6.0 dB` → `-6\x1FdB` ≠ `-6`）。单位 token 属 DSP 知识，
> 剥离会违背「spec 由 parser 产出、零 DSP 知识」原则。单位差异（`-6.0 dB` vs `-6.0`）
> 触发一次过渡——手动改文件本就是边缘场景，过渡有升余弦无听感爆音，可接受（overview/项目概览.md）。

> **无冒号行在产出表中不再出现**（v7.11）：无冒号行已被 `split_command_value` 单冒号
> 校验拒绝（SyntaxError 整体失败），不产出 spec、不落 registry。旧 v7.9「裸命令可达性
> 修正」段落已废弃删除（见「裸命令可达性修正反转」注）。

---

#### ParseContext（解析期间可变状态）

> **v9.11 废弃**：以下 ParseContext / 逐行分发 / 命令处理器细节为 v9.11 前 txt
> 命令体系实现，已删除；保留为历史参考，不代表现行实现。

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

/// 分割 `Command: value` 格式（v7.11 严格化——单冒号校验）。
///
/// - 恰好一个冒号：正常返回 `(命令关键字, 参数体)`
/// - 零个冒号 → `Err(SyntaxError「缺少冒号：命令必须为 关键字: 参数 格式」)`
///   （无冒号行拒绝解析——**有意差异**：EAPO 静默跳过（FilterEngine.cpp 329-330），
///     VxAPO 拒绝报错更严格，overview/项目概览.md「语法严格性」；v8.3 S5 修正「对齐」表述；
///     v7.9 裸命令可达性修正废弃，v7.11 反转）
/// - 多于一个冒号 → `Err(SyntaxError「多余冒号：一行仅允许一个冒号」)`
///   （参数内再含冒号拒绝，确保 value 恒非空、单语义）
fn split_command_value(line: &str) -> Result<(&str, &str), ConfigError>;
```

**逐行分发逻辑**：

```rust
for (i, line) in content.lines().enumerate() {
    ctx.line_number = i + 1;
    let trimmed = line.trim();

    if trimmed.is_empty() || trimmed.starts_with('#') { continue; }

    let (command, value) = split_command_value(trimmed)?;

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
        // DSP 命令（v7.12 白名单三段式——冒号校验 → 命令名白名单 → try_create_named）
        _ => {
            let cmd_lower = command.to_ascii_lowercase();
            // ── 白名单校验（v7.12）：命令名必须已知 ──
            // 静态命令（Device/Stage/Channel/Eval/Include/Filter/GraphicEQ/Preamp/Copy/Delay）
            // 与 REW `Filter N:` 前缀已在上方静态/guard 分支命中；此处剩余命令名必须 ∈
            // registry.factory_names()（IIR/Biquad/Convolution/VSTPlugin/LoudnessCorrection/
            // AuralEnhancer/Reverb/Maximizer/Wide，v9.2）——
            // 否则 SyntaxError「未知命令」，**不落 registry**（修复 v7.11「Unmatched 判定失效」：
            // Convolution 宽容解析不再接管未知命令）。
            if !is_known_dsp_command(&cmd_lower, &ctx.registry.factory_names()) {
                return Err(ConfigError::SyntaxError {
                    file: ctx.current_file.clone(),
                    line: ctx.line_number,
                    message: format!("未知命令 '{}'", command),
                });
            }
            // VSTPlugin 特判（v7.12）：命令名已知（白名单）但功能未启用（v7.11 预留），
            // 不过 try_create（VstFactory 恒 NoMatch）——直接明确报错。
            if cmd_lower == "vstplugin" {
                return Err(ConfigError::SyntaxError {
                    file: ctx.current_file.clone(),
                    line: ctx.line_number,
                    message: format!("命令无效 '{}'：该命令当前未启用（预留）", command),
                });
            }
            // v9.1：改为按命令名精确分派（try_create_named）——只尝试与命令名一致的工厂，
            // 避免 Convolution 等宽容工厂把已知命令的非法参数（如
            // `AuralEnhancer: Bogus 1`）当作自己的 IR 路径吞掉。
            let outcome = ctx.registry.try_create_named(command, value, ctx.dsp_ctx, ...);
            match outcome.result {
                OutcomeKind::FilterAdded(f) => ctx.filters.push(f),
                OutcomeKind::MatchedNoFilter => {},
                OutcomeKind::Aborted => { ctx.abort_file = true; }
                OutcomeKind::Unmatched => {
                    // v7.12：Unmatched 收窄为「已知命令的参数无效」——命令已过白名单
                    // （参数无法解析），不再误报「未知命令」。
                    return Err(ConfigError::SyntaxError {
                        file: ctx.current_file.clone(),
                        line: ctx.line_number,
                        message: format!("命令无效 '{}'：参数无法解析", command),
                    });
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

##### 命令关键字白名单（v7.12，P0-4 二次反馈——「未知命令」判定失效修复）

**问题根因**：v7.11「Unmatched → SyntaxError」失效——`BogusCommand: x` 有冒号、value 非空，
落 `try_create(x)` 后被 Convolution 宽容解析（任意非空字符串=IR 路径）接管 → 创建成功 →
永远到不了 Unmatched → 「未知命令」永不触发。

**决策（定性）**：**任意 DSP 有效命令关键字 parser 都应知晓**——三段式流程：

```
split_command_value（冒号数量严格化，v7.11 已有）
  ↓
命令名 ∈ 白名单？                                ← 本小节（v7.12 新增）
  ├─ 否 → SyntaxError「未知命令 'X'」，不落 registry
  └─ 是 → 静态 match / REW `Filter N:` 前缀 / try_create(value)
              ↓
         try_create Unmatched（语义收窄为「已知命令的参数无效」）
              → SyntaxError「命令无效 'X'：参数无法解析」
```

**白名单来源**（parser 知晓全部合法关键字）：

| 类别 | 命令名 | 命中路径 |
|------|--------|---------|
| 静态命令 | `Device`/`Stage`/`Channel`/`Eval`/`Include`/`Filter`/`GraphicEQ`/`Preamp`/`Copy`/`Delay` | 上方静态 `match`（条件分支 `If`/`ElseIf`/`Else`/`EndIf` 在栈管理阶段先行处理） |
| REW 动态前缀 | `Filter N:`（`Filter 1:`/`Filter 12:`） | guard 分支 `starts_with("filter ")` |
| registry 命令名 | `IIR`/`Biquad`/`Convolution`/`VSTPlugin`/`LoudnessCorrection`（=`registry.factory_names()`，pipeline 4.10） | `_` 分支 `is_known_dsp_command` |

```rust
/// 判断命令名是否为已知 DSP 命令（v7.12 白名单校验）。
///
/// 静态命令与 REW `Filter N:` 前缀已由上层分支拦截；本函数仅检查
/// registry 命令名集合（`FilterRegistry::factory_names()`，pipeline 4.10，
/// 小写比较）。命令名不在此集合 → 未知命令 → 直接
/// SyntaxError「未知命令」——**不落 registry**（修复 Convolution 宽容接管）。
fn is_known_dsp_command(cmd_lower: &str, registry_command_names: &[&str]) -> bool {
    registry_command_names.iter().any(|n| n.eq_ignore_ascii_case(cmd_lower))
}
```

**VSTPlugin 特判（v7.12）**：`VSTPlugin` 在 registry 命令名白名单（功能已注册、预留），
但 `VstFactory` 恒 NoMatch（功能未启用，pipeline 4.20）。白名单命中后**不过 try_create**，
直接 `SyntaxError「命令无效 'VSTPlugin'：该命令当前未启用（预留）」`——消息诚实且准确定位
（区别于「参数无法解析」的误导）。

**Unmatched 语义收窄（v7.12）**：`try_create` 返回 Unmatched 仅表示「**已知命令的参数无效**」
（命令名已过白名单）——不再承担「未知命令」判定。文案：`命令无效 'X'：参数无法解析`。

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
    /// 触发方（object/apo.rs hot_reload）重新解析 config.toml，经 spec 指纹
    /// 比对决定是否真正切换（内容未变 → 幂等跳过，无听感副作用）。
    DirectoryChanged(PathBuf),    // watch_dir
    RegistryChanged,              // 保留：poll_registry 哈希兜底（低频）
}

/// 配置目录变更监控器（v7.8/v7.9/v7.10 事件驱动——**外部驱动模型**）。
///
/// 核心：监控 **目录**（非文件——文件被删除重建时句柄失效，目录天然健壮）。
/// **线程模型（v7.10 澄清，P0-4 执行端反馈①）**：
///   - `ConfigWatcher` **不自启线程**——只管理监控句柄 + 阻塞等待 API；
///   - 线程由**调用方**（`object/apo.rs::start_watcher`，7.1.9）创建并驱动：
///     循环调用 `wait_and_handle()` → 返回 `DirectoryChanged` 时触发 `hot_reload`（7.1.18）
///   - 去重窗口 10ms（对齐 EAPO：FindNextChangeNotification 后 WaitFor(1,10ms)
///     合并编辑器「写临时文件 + rename」的多次通知）
///   - 退出：shutdown_event 置位 → `wait_and_handle` 返回 false → 调用方线程退出
///     （UnlockForProcess 时 SetEvent + join 线程，v7.9 生命周期随锁定周期）
pub struct ConfigWatcher { ... }

impl ConfigWatcher {
    /// 创建监控器（**不启动线程**，v7.10 澄清）——句柄创建 + 状态初始化。
    ///
    /// - watch_dir：**监控目录**（如 `C:\ProgramData\VxAPO\{GUID}`），非 config.toml 文件本身
    /// - shutdown_event：外部持有的退出事件句柄（由 APO 实例持有；UnlockForProcess 时
    ///   `SetEvent` → `wait_and_handle` 返回 false → 调用方 join 线程——v7.9 生命周期随锁定周期）
    /// - 去重窗口固定 10ms（v7.8，对齐 EAPO 实战；旧 500ms 过保守，延迟过高）
    ///
    /// **调用方接线（object 7.1.9 start_watcher）**：
    ///   spawn 线程 → 循环 `wait_and_handle()`：
    ///   1. FindFirstChangeNotificationW(watch_dir, TRUE, FILE_NAME | LAST_WRITE)
    ///   2. WaitForMultipleObjects(shutdown_event | notification_handle)  → 异步等事件（非轮询）
    ///   3. 变更（**目录级，不区分文件**）→ 返回 `Some(DirectoryChanged)` → 触发 hot_reload
    ///   4. FindNextChangeNotification → WaitForMultipleObjects(1, handle, 10ms)【去重】
    ///   5. 循环（spec 指纹短路 + 加载闸门由 reloading 防覆盖）
    ///   `shutdown_event` 置位 → 返回 None/false → 线程退出
    pub fn new(watch_dir: PathBuf, shutdown_event: HANDLE) -> Self;

    /// 等待并处理一个事件（阻塞直到文件变更或 shutdown）。
    /// 返回 false 表示 shutdown（`shutdown_event` 已置位，调用方线程应退出）。
    pub fn wait_and_handle(&mut self) -> bool;

    /// 检查注册表变更（哈希比对，低频；与目录监控并行）。
    pub fn poll_registry(&mut self, current_hash: u64) -> Option<WatchEvent>;

    /// 停止监控并释放句柄（`FindCloseChangeNotification` + `CloseHandle`）。
    /// **不 join 线程**——join 由调用方（`object 7.1.10 stop_watcher`）负责：
    /// 先 `SetEvent(shutdown_event)` → join 线程 → 再 `watch.shutdown()`（v7.10 澄清）。
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

### 6.3 历史命令体系（已删除）

v9.11 起，`config/commands/*` 逐行命令体系已整体移除。当前解析只使用 TOML 模型，
效果器由 `pipeline/dsp/factory.rs` 静态分派构造。旧命令语法、`GraphicEQ:`、
`Channel:` 等行命令不再作为运行时格式；迁移由 `vxapo-cli config convert` 完成。

详细效果器参数与 DSP 行为见 `driver/zh/配置与DSP设计.md`。
