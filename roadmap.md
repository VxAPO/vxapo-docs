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
- 状态：Spec-Finalized
- 优先级：P0 ｜ 关联 Phase：Phase 1-9
- 目标：DLL 可 `regsvr32` 注册（2 个 CLSID 的 COM 类键 + ThreadingModel），APO 对象可被 `CoCreateInstance` 实例化
- 影响模块：`object/dll_exports.rs`、`object/vx_reg_props.rs`
- 规范落点：`object 7.6`（DllRegisterServer/DllUnregisterServer 职责边界 + 完整流程）、`主规范 十一`（dll_exports 依赖补 sys/registry）
- 依赖：无
- DoD：☑ 规范定稿（v7.1）☐ 实现 ☐ 测试

> **分工澄清（v7.1 定稿）**：`regsvr32` 无设备参数，只做全局 COM 类注册（DLL 可加载）；
> "挂载到端点 + FxProperties 设备绑定"由 `install_endpoint`（`install 5.5.2`）承担，经 `vxapo-cli install -d <device>` 触发。两者分层，regsvr32 不绑定设备。
>
> **合规性**：引用约束总表已更新（dll_exports 增 `sys/registry`）；不触碰 RT；不触碰 MMDevices/FxProperties（边界清晰）。
> **实现验收**：`regsvr32 vxapo.dll` → `CoCreateInstance` 两个 CLSID 均可实例化；`regsvr32 /u` 后键清理、重复注册/注销幂等。

### P0-2  config.txt 解析链路补齐（命令工厂替换 NoMatch）
- 状态：Spec-Finalized
- 优先级：P0 ｜ 关联 Phase：Phase 1-9
- 目标：parser.rs + 命令处理器工厂可解析 config.txt 基础命令，无 NoMatch 占位
- 影响模块：`config/parser.rs`、`config/commands/*.rs`、`pipeline/dsp/factory.rs`
- 规范落点：`config 6.0/6.1`（ConfigParser 三入口 + ParseContext + parse_content 逐行分发）、`config 6.3`（register_all_commands 全命令注册）、`config 6.4-6.15`（各命令语义）、`pipeline 4.x factory`（FilterRegistry/create_default_registry/register_builtin_filters）
- 依赖：无
- DoD：☑ 规范定稿（核对确认型，无版本变更）☐ 实现 ☐ 测试

> **定稿说明（v7.1 确认型）**：规范侧**已完整覆盖** P0-2 全部需求（ConfigParser 解析三入口、
> UTF-8/ANSI 降级、命令分发、全命令工厂注册）。本条目为**纯实现缺口**——规范无需新增/修改，
> 无版本递增、无 changelog 记录。
>
> **合规性**：config 不依赖 `pipeline/chain`/具体 Filter 实现（引用约束总表已满足）；
> DSP 命令经 `registry.try_create` 动态创建。
> **实现验收**：`cargo test` 通过 + 解析 config.txt 样例无 `NoMatch` 警告；`cargo check` 无未使用警告。

### P0-3  per-device 配置路径
- 状态：Spec-Finalized
- 优先级：P0 ｜ 关联 Phase：Phase 10
- 目标：`APOInitSystemEffects` 反查设备 GUID → `Documents\VxAPO\{GUID}\config.txt`；目录不存在自动创建，config 不存在写入默认 passthrough
- 影响模块：`object/apo.rs`（Initialize）、`config/watcher.rs`、`install/device`
- 规范落点：`sys 3.6`（known_folder）、`sys 3.3.1b`（APOInitSystemEffects）、`object 7.1.8`（Initialize per-device 路径解析 + config_path 规则）、`主规范 十一`（引用约束同步）
- 依赖：P0-1、P0-2（已 Spec-Finalized ✅）
- DoD：☑ 规范定稿（v7.2）☐ 实现 ☐ 测试

> **定稿说明（v7.2）**：新增 `sys/known_folder.rs`（SHGetKnownFolderPath FFI 收窄）、
> re-export `APOInitSystemEffects`（端点 GUID 提取）、`object 7.1.8` 定义
> `Documents\VxAPO\{GUID}\config.txt` 规则（目录自动创建 / 默认 passthrough / `_default` 兜底）。
>
> **合规性**：known_folder 只做 FFI 收窄（不拼接路径）；对象层负责业务拼接；
> Initialize 为控制线程（I/O 允许）；不触碰 RT / install / config 边界。
> **实现验收**：Initialize 后 `config_path` == `Documents\VxAPO\{GUID}\config.txt`；
> 目录不存在自动创建；config 缺失写默认 passthrough；无 GUID 时回退 `_default`。

### P0-4  配置热重载全链路（watcher + swap + 过渡）
- 状态：Backlog
- 优先级：P0 ｜ 关联 Phase：Phase 10
- 目标：监控线程检测 config.txt 变更 → swap 串联 → 升余弦过渡；修改文件实时生效且无爆音（对齐 v6.9 R1-R4）
- 影响模块：`config/watcher.rs`、`object/apo.rs`（hot_reload/APOProcess）
- 规范落点：（定稿时回填）
- 依赖：P0-3
- DoD：☐ 规范定稿 ☐ 实现 ☐ 测试（含手动听感验证）

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

_（暂无）_