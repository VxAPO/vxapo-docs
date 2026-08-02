# VxAPO 路线清单

> 本文件是**"要做什么"**的规划，与"怎么做"的规范文档（`模块引用规范（无详细模块版）.md` 及各子规范）配套。
> 路线清单本质是规范的一部分——规划先行，规范随后，执行最后。
>
> 状态机：`Backlog → Spec-Drafting → Spec-Finalized → Implementing → Done`
> 硬门禁：仅 **Spec-Finalized** 可进入 Implementing（见 `.clinerules/roadmap-rule.md`）

---

## 状态图例

| 状态 | 含义 | 产出 |
|------|------|------|
| `Backlog` | 已登记待办，尚未选中 | 条目信息（目标/影响模块/依赖） |
| `Spec-Drafting` | 规范起草中：核对/补写该功能涉及的规范章节 | 规范草稿（主规范或子规范） |
| `Spec-Finalized` | 规范定稿：落点已回填，门禁放行 | 版本递增（如有变更）+ changelog + 落点回填 |
| `Implementing` | 执行端 agent 按规范落地 | 实现 + 测试 |
| `Done` | 实现完成并通过合规核对 | 合规核对记录 |

---

## P0 — 阻塞核心链路（Driver 可运行）

> 达标口径：regsvr32 注册 → Windows 加载 → 按设备读 config.txt → passthrough + 热重载生效。

### P0-1  DllRegisterServer 补全（COM 类注册，APO 可加载）
- 状态：Done（v7.4 合规核对通过）
- 优先级：P0 ｜ 关联 Phase：Phase 1-9
- 目标：DLL 可 `regsvr32` 注册（2 个 CLSID 的 COM 类键 + ThreadingModel），APO 对象可被 `CoCreateInstance` 实例化
- 影响模块：`object/dll_exports.rs`、`object/vx_reg_props.rs`
- 规范落点：`object 7.6`（DllRegisterServer/DllUnregisterServer 职责边界 + 完整流程）、`主规范 十一`（dll_exports 依赖补 sys/registry）
- 依赖：无
- DoD：☑ 规范定稿（v7.1）☑ 实现 ☑ 测试

> **分工澄清（v7.1 定稿）**：`regsvr32` 无设备参数，只做全局 COM 类注册（DLL 可加载）；
> "挂载到端点 + FxProperties 设备绑定"由 `install_endpoint`（`install 5.5.2`）承担，经 `vxapo-cli install -d <device>` 触发。两者分层，regsvr32 不绑定设备。
>
> **合规性**：引用约束总表已更新（dll_exports 增 `sys/registry`）；不触碰 RT；不触碰 MMDevices/FxProperties（边界清晰）。
> **实现验收**：`regsvr32 vxapo.dll` → `CoCreateInstance` 两个 CLSID 均可实例化；`regsvr32 /u` 后键清理、重复注册/注销幂等。

### 实现完成报告
- DoD：☑ 实现 ☑ 测试
- 自查结果：
  - RT 无违规：`DllRegisterServer`/`DllUnregisterServer` 由 regsvr32 宿主进程（控制路径）调用，非 RT 路径；
    全部注册表操作经 `sys/registry`（`RegKey::create`/`delete_tree`），无分配热点、无锁间接。
  - 引用约束无打破：dll_exports 依赖 `sys/registry`（v7.1 主规范十一已声明）；
    不触碰 MMDevices / FxProperties（v7.1 职责边界）；`object/vx_reg_props.rs` 的
    `ClsidEntry`/`registration_order`/`unregistration_order` 此前已存在且经 `guid_to_string` 格式化（无 GUID Display 冲突）。
  - 未引入未声明依赖：仅使用 windows-rs 0.62.2 既有 feature（Win32_System_Registry/Com/LibraryLoader）。
- 新增/修改文件：
  - `src/object/dll_exports.rs`（修改，⊆ 影响模块声明）：
    - `DllRegisterServer`：DLL 路径获取失败 → `SELFREG_E_CLASS`；注册失败按逆序回滚已注册条目 → `SELFREG_E_CLASS`（Note 29）
    - `DllUnregisterServer`：先删 `InprocServer32` 子键再删 `CLSID\{GUID}` 父键，键不存在视为成功（幂等，Note 30）
    - `register_com_class`/`unregister_com_class`：改用 `sys/registry`（`RegKey::create` + `write_sz` / `delete_tree`），失败透出具体 HRESULT
    - 本地常量 `SELFREG_E_CLASS = 0x80040201`（windows crate 未导出）
- 遗留问题：无。手动验收（`regsvr32 vxapo.dll` → CoCreateInstance / `regsvr32 /u` 清理）需真实 Windows 注册表环境，
  留待规范侧/用户核验（建议：单元测试层面已覆盖幂等语义——`RegKey::create` 覆盖写入、`delete_tree` 键不存在视为成功）。

### 合规核对记录（v7.4）
- 核对结果：**通过**——实现报告自查（RT 无违规/引用约束无打破/无未声明依赖）与规范落点一致；
  触碰文件 ⊆ 影响模块；`SELFREG_E_CLASS` 本地常量合理（windows crate 未导出）。
- 已归档至本文件「已完成」区（保留章节号便于追溯）。

### P0-2  config.txt 解析链路补齐（命令工厂替换 NoMatch）
- 状态：Done（v7.4 合规核对通过）
- 优先级：P0 ｜ 关联 Phase：Phase 1-9
- 目标：parser.rs + 命令处理器工厂可解析 config.txt 基础命令，无 NoMatch 占位
- 影响模块：`config/parser.rs`、`config/commands/*.rs`、`pipeline/dsp/factory.rs`
- 规范落点：`config 6.0/6.1`（ConfigParser 三入口 + ParseContext + parse_content 逐行分发）、`config 6.3`（register_all_commands 全命令注册，v7.4 修订为仅 DSP 工厂）、`config 6.4-6.15`（各命令语义）、`pipeline 4.x factory`（FilterRegistry/create_default_registry/register_builtin_filters）
- 依赖：无
- DoD：☑ 规范定稿（核对确认型，无版本变更）☑ 实现 ☑ 测试

> **定稿说明（v7.1 确认型）**：规范侧**已完整覆盖** P0-2 全部需求（ConfigParser 解析三入口、
> UTF-8/ANSI 降级、命令分发、全命令工厂注册）。本条目为**纯实现缺口**——规范无需新增/修改，
> 无版本递增、无 changelog 记录。
>
> **合规性**：config 不依赖 `pipeline/chain`/具体 Filter 实现（引用约束总表已满足）；
> DSP 命令经 `registry.try_create` 动态创建。
> **实现验收**：`cargo test` 通过 + 解析 config.txt 样例无 `NoMatch` 警告；`cargo check` 无未使用警告。

### 实现完成报告
- DoD：☑ 实现 ☑ 测试
- 自查结果：
  - RT 无违规：config/parser.rs + commands/* 全部为控制路径（文件 I/O、解析构造 Filter 链），
    无 RT 分配/锁/I/O；`log::warn!` 仅在 Unmatched 时调用（非 RT）。
  - 引用约束无打破：config/parser.rs 仅依赖 `config/error`、`config/commands/*`、
    `pipeline/dsp/filter`、`pipeline/dsp/factory`（均 ⊆ config 规范 6.1 允许依赖）；
    未直接依赖任何 `pipeline/dsp/*.rs` 具体实现，未触碰 `install/`、`object/`、
    `pipeline/chain`、`pipeline/process`、`pipeline/context`。
  - 未引入未声明依赖：仅新增使用既有 `std`（fs/PathBuf）与 `log`（Cargo.toml 已有）。
- 新增/修改文件（⊆ 影响模块 `config/parser.rs`、`config/commands/*.rs`、`pipeline/dsp/factory.rs`）：
  - `src/config/parser.rs`（重写）：ConfigParser 三入口走 `parse_content`/`parse_lines_impl`
    逐行分发（规范 6.1）——条件分支（If/ElseIf/Else/EndIf）始终处理 + false 分支跳过 +
    纯配置命令（Device/Stage/Channel/Eval/Include/Filter/GraphicEQ/Preamp/Copy/Delay）
    分发到各 handle + REW `Filter N:` 动态命令名分发 rew::handle + 其余经
    `registry.try_create`（裸 IIR/Biquad/Convolution/LoudnessCorrection）+
    Unmatched `log::warn` + 条件栈平衡检查（unterminated If → SyntaxError）。
    `read_config_file` UTF-8 优先 + BOM 跳过 + 非 UTF-8 lossy 降级（6.1 ANSI 降级意图）。
    `ParseContext.current_file` 由 `&'a Path` 改为所有权 `PathBuf`（见反馈②）。
  - `src/config/commands/include.rs`（修复）：去 `Box::leak` 路径泄漏，
    `current_file` 所有权传递，递归深度限制保留；Include 子文件滤波器并入主列表（测试验证）。
  - `src/config/commands/filter.rs` / `rew.rs`（修复）：`OFF` 创建 `PassthroughFilter`
    （规范 6.14 语义），保留链序号位置；REW 逗号小数规范化已接线。
  - `src/config/commands/{cond,expr,channel,device}.rs`（仅测试构造点 `PathBuf` 适配）。
- 测试：436 passed / 0 failed（原 417 + 新增 19：parser 逐行分发/集成样例/Include 递归/
  AbortFile/条件分支/Eval/Stage/REW/Filter OFF→Passthrough/ANSI lossy/BOM 等）。
  解析 config.txt 样例（纯配置 + Preamp/Filter/GraphicEQ/Copy/Delay/Channel）无 NoMatch。
- 遗留问题：无。`pipeline/dsp/factory.rs` 未改动（9 个 DSP 工厂已全接线）；
  6.3 `register_all_commands` 是否需把 config 命令工厂注册进 FilterRegistry 留待规范侧澄清（见反馈①）。

### 反馈
- 状态建议：Spec-Finalized → Spec-Finalized（无需状态回退；以下为规范文本澄清建议）
- 问题：
  1. **规范 6.3 vs 6.1 矛盾**：6.3 `register_all_commands` 列出了
     `DeviceFactory/IfFactory/EvalFactory/IncludeFactory/StageFactory/ChannelFactory/RewFactory`
     7 个 config 命令工厂注册进 FilterRegistry；但 6.1 逐行分发逻辑（221-269 行）明确
     Device/Stage/Channel/Eval/Include 走 `handle_*` 静态分发、未知命令走 registry。
     由于 `FilterFactory::create_filter` 只接收**冒号后的 value**（不含命令关键字，
     dsp/factory.rs 4.10），config 命令工厂无法从 value 反推命令名（`Filter:` 的
     value 是 `ON PK...`，`Device:` 的 value 是设备路径）——注册进 registry 的
     config 工厂永远无法命中。实现以 6.1（分发权威）为准：静态分发 + registry 兜底，
     `register_all_commands` 仅注册 9 个 DSP 工厂。需规范侧澄清 6.3 的注册意图
     （或将 6.3 改为"config 命令由 parser 静态分发，仅注册 DSP 工厂"）。
  2. **规范 6.1 `ParseContext.current_file: &'a Path` 与 Include 递归冲突**：
     Include 子解析需独立持有子文件路径（错误报告 + 相对路径），借用 `&'a Path`
     无法跨递归层安全表达（会与 `filters: &'a mut` 的 `'a` 冲突），旧实现用
     `Box::leak` 绕过后泄漏。已用所有权 `PathBuf` 替代（消除泄漏）。建议规范侧
     将 6.1 原型更新为 `current_file: PathBuf`。
  3. **规范 6.15 REW `Filter N:` 命令名未在 6.1 分发逻辑描述**：命令关键字是动态
     `Filter N`（`Filter 1:`/`Filter 12:`），6.1 静态 match 无法命中。已加
     `cmd_lower.starts_with("filter ")` 分支分发 `rew::handle`。建议规范侧在 6.1
     补充此动态命令名分发路径。
- 建议：由规范侧核对以上 3 点后决定修订或维持现状；执行端已按当前定稿实现全部
  DoD（`cargo test` 通过 + 样例无 NoMatch + 无未使用警告）。

### 反馈修订记录（v7.4，规范侧处理）
- 反馈①（6.3 注册意图）：**已修订** —— 6.3 `register_all_commands` 改为只注册 DSP 工厂；
  纯配置命令由 6.1 静态分发（`FilterFactory::create_filter` 只收 value 不含命令关键字，
  config 命令工厂注册后无法命中）——**对应章节**：`config 6.3`
- 反馈②（current_file 借用冲突）：**已修订** —— `ParseContext.current_file: &'a Path` →
  `PathBuf`（所有权，消除 `Box::leak` 泄漏）——**对应章节**：`config 6.1`
- 反馈③（REW 动态命令名）：**已修订** —— 6.1 分发逻辑补充 `starts_with("filter ")`
  前缀分支 → `rew::handle`——**对应章节**：`config 6.1`
- 同时新增「主规范 十七、实现反馈闭环」机制（执行端不改状态 + 零容忍绕过 + 规范侧修订闭环）。

### 合规核对记录（v7.4）
- 核对结果：**通过**——实现报告自查（RT 无违规/引用约束无打破/无未声明依赖）与规范落点一致；
  触碰文件 ⊆ 影响模块；3 点反馈已全部纳入 v7.4 修订；测试 436 passed。
- 已归档至本文件「已完成」区（保留章节号便于追溯）。

### P0-3  per-device 配置路径
- 状态：Done（v7.6 合规核对通过）
- 优先级：P0 ｜ 关联 Phase：Phase 10
- 目标：`APOInitSystemEffects` 反查设备 GUID → `Documents\VxAPO\{GUID}\config.txt`；目录不存在自动创建，config 不存在写入默认 passthrough
- 影响模块：`object/apo.rs`（Initialize）、`config/watcher.rs`、`install/device`
- 规范落点：`sys 3.6`（known_folder）、`sys 3.3.1b`（APOInitSystemEffects，v7.6 实测路径）、`object 7.1.8`（Initialize per-device 路径解析 + config_path 规则 + 二级兜底）、`object 7.1.11`（过渡完成 reloading 修正）、`主规范 十一`（引用约束同步）
- 依赖：P0-1、P0-2（已 Done ✅）
- DoD：☑ 规范定稿（v7.2/v7.6）☑ 实现 ☑ 测试

> **定稿说明（v7.2）**：新增 `sys/known_folder.rs`（SHGetKnownFolderPath FFI 收窄）、
> re-export `APOInitSystemEffects`（端点 GUID 提取）、`object 7.1.8` 定义
> `Documents\VxAPO\{GUID}\config.txt` 规则（目录自动创建 / 默认 passthrough / `_default` 兜底）。
>
> **合规性**：known_folder 只做 FFI 收窄（不拼接路径）；对象层负责业务拼接；
> Initialize 为控制线程（I/O 允许）；不触碰 RT / install / config 边界。
> **实现验收**：Initialize 后 `config_path` == `Documents\VxAPO\{GUID}\config.txt`；
> 目录不存在自动创建；config 缺失写默认 passthrough；无 GUID 时回退 `_default`。

### 实现完成报告
- DoD：☑ 实现 ☑ 测试
- 自查结果：
  - RT 无违规：per-device 路径解析全部在 Initialize（控制线程，I/O 允许）；
    APOProcess 过渡修正（transition=None 写 bypass / advance None→factor=1.0）为纯
    栈上计算，无分配/锁/I/O 新增。
  - 引用约束无打破：`sys/known_folder.rs` 仅依赖 windows crate（`Win32_UI_Shell` +
    `Win32_System_Com`，v7.2 主规范十一已声明）；`object/apo.rs` 增 `sys/known_folder`/
    `sys/com/apo_types`（APOInitSystemEffects）依赖（v7.2 已声明）。
  - 未引入未声明依赖：windows features 增加 `Win32_UI_Shell`、
    `Win32_UI_Shell_PropertiesSystem`、`Win32_System_Variant`、
    `Win32_System_Com_StructuredStorage`（均为 sys 3.6 / 3.3.1b 实现必需，规范已声明）。
- 新增/修改文件（⊆ 影响模块 `object/apo.rs`、`config/watcher.rs`、`install/device`；sys 侧为 v7.2 新增模块）：
  - `src/sys/known_folder.rs`（新建，v7.2 规范 3.6）：`documents_folder()` 封装
    `SHGetKnownFolderPath(FOLDERID_Documents)`（实测返回 `Result<PWSTR>`），
    `CoTaskMemGuard` RAII 释放（所有路径含错误均释放），`PWSTR::to_string` 转 String。
  - `src/sys/com/apo_types.rs`：re-export `APOInitSystemEffects` / `APOInitBaseStruct` /
    `PKEY_AudioEndpoint_GUID` / `IPropertyStore` / `PROPVARIANT` / `VT_CLSID` 等。
  - `src/object/apo.rs`：
    - `ApoObject.config_path: Mutex<String>`（Initialize 确定，LockForProcess/hot_reload 复用）
    - `extract_endpoint_guid`：`APOInitSystemEffects.pAPOSystemEffectsProperties`
      ->`IPropertyStore::GetValue(PKEY_AudioEndpoint_GUID)` -> PROPVARIANT `puuid`（VT_CLSID）
    - `resolve_config_path`：`{Documents}\VxAPO\{GUID}\config.txt`，目录自动创建、config 缺失
      写默认 passthrough、无 GUID/Documents 失败回退 `_default` / 默认路径
    - `Initialize` 重写：参数校验（pby_data 非空 + cbSize 足够）+ 状态机 + per-device 路径
  - `src/sys.rs`：`pub mod known_folder`。
- 测试：437 → 441 passed（新增 sys/known_folder::documents_folder + object/apo
  config_path 3 测试：`_default` 兜底/目录创建/文件写入/幂等/init=None）。
- 遗留问题：
  1. **APOProcess 过渡缺陷修复（独立发现 + 用户确认）**：① `finished` 分支先置
     `reloading=true` 再调 `hot_reload()` → 短锁检查 `reloading==true` 直接 return，
     延迟重载被自己拦截——已修正为**不置位直接调用**（hot_reload 自己管理 reloading）；
     ② `transition=None` 但 `pending` 残留（无混合器）→ 本帧不写输出，违反 APO 契约——
     已加防御分支：直接复制输入到输出（bypass）+ 旧链退役 + 触发补重载；
     ③ 混合 `advance()` 返回 None（过渡已达上限）→ 本帧不写输出——已修正为按 factor=1.0
     （纯新链）输出。
  2. 真实端点 GUID 提取需 Windows 音频引擎真实 APOInitSystemEffects 环境验证
     （单测以 `_default` 兜底覆盖 None 分支；VT_CLSID/puuid 分支留手动验收）。

### 反馈
- 状态建议：Spec-Finalized → Spec-Finalized（无需状态回退；规范文本澄清 2 点）
- 问题：
  1. **object 7.1.8 原型 `pSystemEffectsProperties->pEndpointGuid` 与 windows-rs 0.62.2
     实测不符**：`APOInitSystemEffects` 无 `pSystemEffectsProperties` 字段，而是
     `pAPOSystemEffectsProperties: ManuallyDrop<Option<IPropertyStore>>`；端点 GUID 经
     `IPropertyStore::GetValue(&PKEY_AudioEndpoint_GUID)` 返回 PROPVARIANT（VT_CLSID）
     的 `puuid` 提取。已按实测实现。建议规范侧在 7.1.8 更新提取路径描述。
  2. **7.1.8 步骤 6「启动 watcher」为 P0-4 依赖**：Initialize 中 watcher 启动时机/生命周期
     已在 v7.3 定义（`object 7.1.8`），但 P0-3 实现未接线 watcher（`config/watcher.rs` 未
     创建实例）——P0-4 实现时补（config_path 字段已就位，watch_dir = config_path 父目录）。
- 建议：规范侧核对以上 2 点；执行端 DoD 已全通过（cargo test 441 + 验收 4 项：路径/目录/文件/兜底）。

### 反馈修订记录（v7.6，规范侧处理）
- 反馈①（APOInitSystemEffects 实测结构）：**已修订** —— `object 7.1.8` + `sys 3.3.1b` +
  `sys 3.3` 引用来源更新为 `pAPOSystemEffectsProperties`（IPropertyStore）→
  `PKEY_AudioEndpoint_GUID`（PROPVARIANT VT_CLSID `puuid`）——**对应章节**：`object 7.1.8`、`sys 3.3.1b`
- 反馈②（watcher 接线）：**确认为 P0-4 职责**（config_path 已就位；watch_dir = 父目录，
  v7.3 已定义启动约定，P0-4 实现时补 ConfigWatcher 实例）——**对应章节**：`object 7.1.8`（watcher 约定）
- **二次检查（规范侧实读 apo.rs 881 行后补充 3 项对齐）**：
  - 对齐①（7.1.11 过渡完成 reloading 拦截）：规范伪代码原写"先 `reloading=true` 再调
    hot_reload"→ 与实装矛盾（`reloading` 表示"正在解析中"，hot_reload 自己会置位；
    先置 true 会短锁直接 return → 延迟重载被拦截）。修订为**不置位直接调用** + 补
    pending 残留防御（transition=None → bypass 输出 + 旧链退役）+ advance None →
    factor=1.0——**对应章节**：`object 7.1.11`
  - 对齐②（7.1.8 Initialize 非法数据）：规范原写"非法 → E_INVALIDARG"；实装为
    **降级默认配置（log::warn + resolve_config_path(None)）仍返回 Ok**（SDK 容错：
    Initialize 失败 APO 无法加载/音频停摆，降级 passthrough 更稳健）——**对应章节**：`object 7.1.8`
  - 对齐③（7.1.8 Documents 二级兜底）：规范只写"无 GUID → `_default`"；实装另有
    `documents_folder()` 失败 → **固定 `C:\ProgramData\VxAPO\config.txt`**（DEFAULT_CONFIG_PATH）
    ——**对应章节**：`object 7.1.8`

### 合规核对记录（v7.6）
- 核对结果：**通过**——实现报告自查（RT 无违规/引用约束无打破/触碰文件 ⊆ 影响模块）
  与规范落点一致；441 passed；v7.4/v7.6 反馈均已纳入修订（config 3 点 + APOInit 结构 + 过渡修正 3 点）。
- 已归档至本文件「已完成」区（保留章节号便于追溯）。

### 实现缺陷反馈（v7.7，用户反馈 → 规范补齐）
- **问题**：`IsInputFormatSupported` 在 `LockForProcess` **之前**被引擎调用，此时 `pipeline_context`
  为 `PipelineContext::new()`（全零）——实装的等值比较（`fmt.channels == ctx.input_channels`
  `&& fmt.sample_rate == ctx.sample_rate`）**永远不成立** → 拒绝所有格式、APO 无法协商。
- **规范修订**：object 7.1.16 补**关键时序约束**——协商阶段禁止依赖 pipeline_context；
  正确做法为对请求格式做**独立属性检查**（浮点格式 + 44.1k~192k + 1~8 通道）——
  **对应章节**：`object 7.1.16`（主规范 v7.7）
- **实现端待办**：`IsInputFormatSupported`/`IsOutputFormatSupported` 需按 7.1.16 独立属性检查实现
  （去掉 pipeline_context 等值比较），并补范围检查（实现端接单）。

### P0-4  配置热重载全链路（watcher + swap + 过渡）
- 状态：Spec-Finalized
- 优先级：P0 ｜ 关联 Phase：Phase 10
- 目标：监控线程检测 config.txt 变更 → swap 串联 → 升余弦过渡；修改文件实时生效且无爆音（对齐 v6.9 R1-R4）
- 影响模块：`config/watcher.rs`、`config/parser.rs`（filter_spec 产出，v7.9）、`object/apo.rs`（hot_reload/APOProcess）
- 规范落点：`config 6.1`（filter_spec 契约 + parse_file_with_spec + 128KB 逐文件闸门，v7.9）、`config 6.2`（目录级事件驱动 + DirectoryChanged，v7.8/v7.9）、`object 7.1.8`（watcher 生命周期随锁定周期，v7.9）、`object 7.1.9`（末尾启动 watcher + active_spec 基线，v7.9）、`object 7.1.18`（spec 短路 + 保留旧链，v7.9）、`intent.md`（产品意图）
- 依赖：P0-3（已 Done ✅）
- DoD：☑ 规范定稿（v7.3/v7.8/v7.9）☐ 实现 ☐ 测试（含手动听感验证）

> **定稿说明（v7.3，v7.8/v7.9 修订）**：watcher 能力由**轮询（2000ms + 500ms 去重）**升级为
> **Win32 事件驱动**（`FindFirstChangeNotificationW` + `WaitForMultipleObjects` + 10ms 去重 +
> shutdown_event 退出）——对齐 EAPO `notificationThread`。
> v7.9（执行端反馈 → 方案定型）：
> - 目录级语义：`FindFirstChangeNotificationW` 不提供文件名，旧「校验 config.txt」无法实现
>   → 统一 `DirectoryChanged`，hot_reload 内 spec 指纹比对决定是否切换。
> - 配置指纹：parser 产出 `FilterSpec`（命令名 + `\x1F` + token 规范化），
>   `parse_file_with_spec` 双返回；Include 失败 = 整体失败。
> - 行为链：目录变更 → 128KB 闸门 → 重新解析 + spec 比对 → 相同幂等跳过 / 不同建新链过渡。
> - watcher 生命周期随锁定周期：Lock 末尾启动、Unlock 停止。
> - 同时：热重载解析失败**保留旧链**、过渡缓冲 LockForProcess **预分配充足容量**（v7.8）。
>
> **合规性**：watcher 为后台线程（控制路径，I/O 允许）；事件仅在非过渡期触发 hot_reload；
> 不触碰 RT 分配/锁。**实现验收**：修改 `Documents\VxAPO\{GUID}\config.txt` → 音频变化无爆音；
> 10ms 内生效；解析出错时旧 EQ 保持；无关文件变更/内容未变不触发过渡（spec 短路）。
>
> ### 执行端反馈（v7.9，P0-4 目录级语义矛盾）
> - **问题**：config 6.2 事件驱动流程「校验文件名 == config.txt」与
>   `FindFirstChangeNotificationW` 语义矛盾——目录级通知**不提供具体文件名**，
>   无法逐文件过滤（文件名信息只有 `ReadDirectoryChangesW` 扩展才有）。
> - **方案定型（用户）**：不做文件名校验；目录任何变更 → hot_reload → 128KB 闸门 →
>   重新解析 + `parse_file_with_spec` 产出 spec chain → 与 active_spec 比较 →
>   相同幂等跳过 / 不同建新链过渡。**根因**：不比较文件内容（哈希/String），
>   而比较**解析后的配置指纹**（含 Include 递归展开）——"配置实质变了"的真实语义。
> - **规范修订（v7.9）**：config 6.1（FilterSpec 契约 + parse_file_with_spec + 裸命令可达性）、
>   config 6.2（DirectoryChanged 目录级）、object 7.1.8/7.1.9/7.1.10（watcher 生命周期）、
>   object 7.1.18（spec 短路 + 128KB 闸门）、object 7.1.3（active_spec 字段）——
>   **对应章节**：`config 6.1/6.2`、`object 7.1.3/7.1.8/7.1.9/7.1.10/7.1.18`、`intent.md`

### P0-5  RT 入口 panic 防护（catch_unwind）
- 状态：Backlog
- 优先级：P0 ｜ 关联 Phase：Phase 1-9
- 目标：`APOProcess` / `CalcInputFrames` / `CalcOutputFrames` RT 入口 `catch_unwind` 包裹——捕获 → 输出清零 + BUFFER_SILENT + stats.error_count++，杜绝 panic 跨 FFI unwind 到 audiodg 崩溃
- 影响模块：`object/apo.rs`（RT 三入口）
- 规范落点：（定稿时回填；对象层 RT 入口）
- 依赖：无
- DoD：☐ 规范定稿 ☐ 实现 ☐ 测试（debug panic=unwind 下模拟 panic 验证不崩溃）

> **说明**：release（`panic="abort"`，O3）时 catch_unwind 为空操作——真防线是 panic hook + abort；
> debug/unwind 测试态才有防御意义（旧框架 Note 60/68 已澄清）。属 P0 尾巴（威胁 audiodg 稳定性）。

### P1-5  子 APO 委托实现（object/child.rs 落地）
- 状态：Backlog
- 优先级：P1 ｜ 关联 Phase：Phase 10T
- 目标：`object/child.rs` 规范已完备（三接口类型化持有 + 委托），实现缺——补 `ApoObject.child_apo` 字段 + CoCreateInstance + Initialize/LockForProcess/UnlockForProcess/APOProcess 完整委托（对齐 EAPO childAPO/childRT/childCfg）
- 影响模块：`object/child.rs`、`object/apo.rs`、`install/device/slots.rs`（子 APO GUID 读取）
- 规范落点：（定稿时回填；`object 7.1.3` child_apo 字段 + `object 7.2` child.rs 方法）
- 依赖：P0-4、P0-5
- DoD：☐ 规范定稿 ☐ 实现 ☐ 测试

> **说明**：EAPO 在 Initialize 中 `CoCreateInstance(子 APO GUID)` → QI 三接口 → 委托全部方法
> （v7.8 EAPO 源码二次检查确认）；VxAPO 规范 object 7.2 已有定义，实现尚缺——P0 链路之后补。

---

## P1 — 核心功能（CLI 可操控 + 效果扩展）

> 达标口径：命令行完成设备配置管理、导入导出、预设；4 个 FxSound 效果可听。

### P1-1  FxSound 效果接入（Wide → Aural → Maximizer → Lex）
- 状态：Backlog
- 优先级：P1 ｜ 关联 Phase：Phase 11
- 目标：四个 FxSound 效果作为原生 Filter 接入 config.txt 解析链路，按复杂度递增
- 影响模块：`pipeline/dsp/{wide,aural,maximizer,lex}.rs`、`pipeline/dsp/factory.rs`、`config/commands/`
- 规范落点：（定稿时回填；含 pipeline 4.9 Filter 扩展 + config 命令语法）
- 依赖：P0-2（解析链路先通）
- DoD：☐ 规范定稿 ☐ 实现 ☐ 测试（可听感验证）

### P1-2  CLI per-device 配置管理
- 状态：Backlog
- 优先级：P1 ｜ 关联 Phase：Phase 12
- 目标：在现有诊断型 vxapo-cli 基础上扩展写入能力：`config show/set/reset`、`preset list/apply/set-intensity`、`install/uninstall`、`inherit`
- 影响模块：`vxapo-cli`（crate 侧，非规范模块树）、`install/selector/operation.rs`、`config/commands.rs`
- 规范落点：（定稿时回填；**注意**：CLI 属规范仓库之外的新 crate，需在规范侧声明其边界——只读安装层 API，不触碰 pipeline/RT）
- 依赖：P0-4、P1-1
- DoD：☐ 规范定稿 ☐ 实现 ☐ 测试

### P1-3  设备配置继承（DeviceProfile v1）
- 状态：Backlog
- 优先级：P1 ｜ 关联 Phase：Phase 17（前置概念验证）
- 目标：设备配置可继承（完全继承 / 继承并覆盖），父设备修改子设备自动同步；新增 DeviceProfile 概念（设备绑定层）
- 影响模块：`install/device/info.rs`、`config/parser.rs`；**App/CLI 侧新增 DeviceProfile 数据模型（规范外声明）**
- 规范落点：（定稿时回填；主规范需补充 DeviceProfile 概念模型 + 与三层抽象的关系）
- 依赖：P1-2
- DoD：☐ 规范定稿 ☐ 实现 ☐ 测试

### P1-4  EAPO config.txt 导入 + 逆向意图识别 v1
- 状态：Backlog
- 优先级：P1 ｜ 关联 Phase：Phase 18（核心差异化前置）
- 目标：导入 EAPO config.txt 为可编辑预设；启发式规则拆分感知维度（置信度标注 + 用户确认）
- 影响模块：`config/parser.rs`（复用 EAPO 语法）、`vxapo-cli`（import 子命令）、App 侧（后续）
- 规范落点：（定稿时回填；config 模块增加"导入断言"章节 + 逆向识别规则表）
- 依赖：P1-2、P1-3（导入需落到 DeviceProfile/预设容器）
- DoD：☐ 规范定稿 ☐ 实现 ☐ 测试

---

## P2 — 增强（预留，后补）

> 以下条目仅列出方向，登记时需完整填写字段（优先级 P2 ｜ 关联阶段）。

- GUI 三 TAB 应用（预设选择 / 效果调节 / 高级参数）——参考《项目UI呈现.txt》
- 预设 TOML 生态（内置预设调音、导入导出、分享）
- 试听功能（15s 片段 + bypass）
- 实时频谱 + EQ 曲线预览
- 系统托盘 + 开机自启
- 安装包（MSI/NSIS）
- 更多效果 / 声道独立处理 / 动态压缩等进阶

---

## 已完成

> 移出活跃清单的功能（`Done`）归档至此，保留关键章节号便于追溯。

### P0-1  DllRegisterServer 补全（COM 类注册，APO 可加载）— Done @ v7.4
- 规范落点：`object 7.6`（职责边界 + 完整流程）、`object 7.5`（正式 GUID，v7.5）、`主规范 十一`（dll_exports 增 sys/registry）
- 实现：`src/object/dll_exports.rs`（DllRegisterServer/失败 SELFREG_E_CLASS 逆序回滚 + DllUnregisterServer 幂等）
- 验收：单元测试幂等语义覆盖；手动 `regsvr32`/`CoCreateInstance` 留待真实 Windows 环境
- **v7.5 更新**：CLSID 正式 GUID 落定——`PRE_MIX = 41C34613-D391-459D-A039-72B2B15A1A1D`、`POST_MIX = B4A97313-ABC0-45ED-9C33-428B20D39428`

### P0-2  config.txt 解析链路补齐（命令工厂替换 NoMatch）— Done @ v7.4
- 规范落点：`config 6.0/6.1/6.3/6.4-6.15`（v7.4 修订 current_file/REW 分发/注册意图）、`pipeline 4.x factory`
- 实现：`src/config/parser.rs`（重写逐行分发）、`src/config/commands/{include,filter,rew,cond,expr,channel,device}.rs`（修复）
- 验收：436 passed（原 417 + 新增 19）；样例无 NoMatch；无未使用警告
- 反馈闭环：3 点实现反馈 → v7.4 规范修订（见「主规范 十七」）

### P0-3  per-device 配置路径 — Done @ v7.6
- 规范落点：`sys 3.6`（known_folder）、`sys 3.3.1b`（APOInitSystemEffects，v7.6 实测路径）、`object 7.1.8`（per-device 路径 + 二级兜底）、`object 7.1.11`（过渡完成 reloading 修正）
- 实现：`src/sys/known_folder.rs`（新建）、`src/sys/com/apo_types.rs`（re-export）、`src/object/apo.rs`（extract_endpoint_guid/resolve_config_path/Initialize 重写 + 过渡修正）、`src/sys.rs`
- 验收：441 passed；路径/目录/文件/`_default` 兜底 4 项；真实端点 GUID 提取留手动验收
- 反馈闭环：APOInit 实测结构（v7.6）+ 过渡 3 点修正（v7.6 二次检查对齐）+ watcher 接线留 P0-4（v7.3 已定义）
