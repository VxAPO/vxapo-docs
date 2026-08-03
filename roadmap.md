# VxAPO 路线清单

> 本文件是**"要做什么"**的规划，与"怎么做"的规范文档（`模块引用规范（无详细模块版）.md` 及各子规范）配套。
> 路线清单本质是规范的一部分——规划先行，规范随后，执行最后。
>
> 状态机：`Backlog → Spec-Drafting → Spec-Finalized → Implementing → Done`
> 硬门禁：仅 **Spec-Finalized** 可进入 Implementing（见 `.clinerules/roadmap-rule.md`）
> **反馈外置（v8.0）**：执行端反馈统一记录于 `feedback.md`（维护规则 `.clinerules/feedback-rule.md`）；
> 本文件条目只保留状态、DoD、规范落点与反馈引用（`> 反馈记录：feedback.md #PX-X`）——保持清单精简。

---

## 状态图例

| 状态 | 含义 | 产出 |
|------|------|------|
| `Backlog` | 已登记待办，尚未选中 | 条目信息（目标/影响模块/依赖） |
| `Spec-Drafting` | 规范起草中：核对/补写该功能涉及的规范章节 | 规范草稿（主规范或子规范） |
| `Spec-Finalized` | 规范定稿：落点已回填，门禁放行 | 版本递增（如有变更）+ changelog + 落点回填 |
| `Implementing` | 执行端 agent 按规范落地 | 实现 + 测试 |
| `Done` | 实现完成并通过合规核对 | 合规核对记录（反馈/报告外置 feedback.md） |

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
> "挂载到端点 + FxProperties 设备绑定"由 `install_endpoint`（`install 5.5.2`）承担，经 `vxapo-cli install -d <device>` 触发。两者分层。

### P0-2  config.txt 解析链路补齐（命令工厂替换 NoMatch）
- 状态：Done（v7.4 合规核对通过）
- 优先级：P0 ｜ 关联 Phase：Phase 1-9
- 目标：parser.rs + 命令处理器工厂可解析 config.txt 基础命令，无 NoMatch 占位
- 影响模块：`config/parser.rs`、`config/commands/*.rs`、`pipeline/dsp/factory.rs`
- 规范落点：`config 6.0/6.1`（ConfigParser 三入口 + ParseContext + parse_content 逐行分发）、`config 6.3`（register_all_commands 全命令注册，v7.4 修订为仅 DSP 工厂）、`config 6.4-6.15`（各命令语义）、`pipeline 4.x factory`（FilterRegistry/create_default_registry/register_builtin_filters）
- 依赖：无
- DoD：☑ 规范定稿（核对确认型，无版本变更）☑ 实现 ☑ 测试

> **实现验收**：`cargo test` 通过 + 解析 config.txt 样例无 `NoMatch` 警告；`cargo check` 无未使用警告。
> 反馈记录：`feedback.md #P0-2-1`（6.3 注册意图）、`#P0-2-2`（current_file 借用）、`#P0-2-3`（REW Filter N:）——均已修订（v7.4）。

### P0-3  per-device 配置路径
- 状态：Done（v7.6 合规核对通过）
- 优先级：P0 ｜ 关联 Phase：Phase 10
- 目标：`APOInitSystemEffects` 反查设备 GUID → `Documents\VxAPO\{GUID}\config.txt`；目录不存在自动创建，config 不存在写入默认 passthrough
- 影响模块：`object/apo.rs`（Initialize）、`config/watcher.rs`、`install/device`
- 规范落点：`sys 3.6`（known_folder）、`sys 3.3.1b`（APOInitSystemEffects，v7.6 实测路径）、`object 7.1.8`（Initialize per-device 路径解析 + config_path 规则 + 二级兜底）、`object 7.1.11`（过渡完成 reloading 修正）、`主规范 十一`（引用约束同步）
- 依赖：P0-1、P0-2（已 Done ✅）
- DoD：☑ 规范定稿（v7.2/v7.6）☑ 实现 ☑ 测试

> **实现验收**：Initialize 后 `config_path` == `Documents\VxAPO\{GUID}\config.txt`；
> 目录不存在自动创建；config 缺失写默认 passthrough；无 GUID 时回退 `_default`。
> 反馈记录：`feedback.md #P0-3-1`（APOProcess 过渡缺陷）、`#P0-3-2`（APOInit 实测结构）——均已修订（v7.6）。
> 观察（v8.0 移出）：v7.6 二次检查 3 项对齐（reloading 拦截 / Initialize 降级 / Documents 兜底）——详见主规范对应章节，不再重复登记。

### P0-4  配置热重载全链路（watcher + swap + 过渡）
- 状态：Spec-Finalized
- 优先级：P0 ｜ 关联 Phase：Phase 10
- 目标：监控线程检测 config.txt 变更 → swap 串联 → 升余弦过渡；修改文件实时生效且无爆音（对齐 v6.9 R1-R4）
- 影响模块：`config/watcher.rs`、`config/parser.rs`（filter_spec 产出，v7.9）、`object/apo.rs`（hot_reload/APOProcess）
- 规范落点：`config 6.1`（filter_spec 契约 + parse_file_with_spec + 128KB 逐文件闸门 + 单冒号校验 + 白名单三段式，v7.9/v7.11/v7.12）、`config 6.2`（目录级事件驱动 + DirectoryChanged + 外部驱动模型，v7.8/v7.9/v7.10）、`object 7.1.8`（watcher 生命周期随锁定周期，v7.9）、`object 7.1.9`（末尾启动 watcher + active_spec 基线 + start_watcher 定义，v7.9/v7.10）、`object 7.1.10`（stop_watcher 定义，v7.10）、`object 7.1.18`（spec 短路 + 保留旧链，v7.9）、`pipeline 4.19/4.20`（Convolution strict + VST 特判，v7.11/v7.12）、`intent.md`（产品意图 + 语法严格性 + 诊断日志归属）
- 依赖：P0-3（已 Done ✅）
- DoD：☑ 规范定稿（v7.3/v7.8/v7.9/v7.10/v7.11/v7.12）☐ 实现（部分：config/parser/watcher 已实装；对象层接线 + v7.11/v7.12 严格化待补）☐ 测试（含手动听感验证）

> **定稿说明（v7.3，v7.8/v7.9 修订）**：watcher 能力由**轮询（2000ms + 500ms 去重）**升级为
> **Win32 事件驱动**（`FindFirstChangeNotificationW` + `WaitForMultipleObjects` + 10ms 去重 +
> shutdown_event 退出）——对齐 EAPO `notificationThread`。
> 行为链：目录变更 → 128KB 闸门 → 重新解析 + spec 比对（与 active_spec）→ 相同幂等跳过 / 不同建新链过渡；
> watcher 生命周期随锁定周期（Lock 末尾启动、Unlock 停止）；解析失败保留旧链；过渡缓冲预分配。
>
> **实现验收**：修改 `Documents\VxAPO\{GUID}\config.txt` → 音频变化无爆音；10ms 内生效；
> 解析出错时旧 EQ 保持；无关文件变更/内容未变不触发过渡（spec 短路）。
>
> **反馈记录**（全部已修订）：
> - `feedback.md #P0-4-1`（v7.9 目录级语义矛盾）、`#P0-4-2`（v7.10 watcher 线程模型）、
>   `#P0-4-3`（v7.11 语法严格化）、`#P0-4-4`（v7.12 白名单三段式）
>
> **执行端待办**（补做后回归，DoD 方全勾）：
> - ① **对象层接线**（v7.10 确认）：ApoObject 加 `watcher_thread`/`watcher_shutdown_event` 字段；
>   `LockForProcess` 末尾调 `start_watcher()`（spawn 循环 wait_and_handle → hot_reload）；
>   `UnlockForProcess` 调 `stop_watcher()`（SetEvent + join + close）；回归测试。
> - ② **v7.11 严格化**：`split_command_value` 单冒号校验；`_` 分支 Unmatched → SyntaxError（删裸命令兜底）；
>   `parse_convolution_params` 改 `Result`；测试断言更新；错误日志写入 `log/`。
> - ③ **v7.12 白名单**：`_` 分支补白名单校验（is_known_dsp_command + VSTPlugin 特判 + Unmatched 文案收窄）；
>   测试 `spec_with_unknown_command_reports_error`。
> - ④ 真实音频引擎验证（无爆音）留手动验收。

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

### P0-6  子 APO 委托实现（object/child.rs 落地）
- 状态：Backlog
- 优先级：P0 ｜ 关联 Phase：Phase 10T
- 目标：`object/child.rs` 规范已完备（三接口类型化持有 + 委托），实现缺——补 `ApoObject.child_apo` 字段 + CoCreateInstance + Initialize/LockForProcess/UnlockForProcess/APOProcess 完整委托（对齐 EAPO childAPO/childRT/childCfg）
- 影响模块：`object/child.rs`、`object/apo.rs`、`install/device/slots.rs`（子 APO GUID 读取）
- 规范落点：（定稿时回填；`object 7.1.3` child_apo 字段 + `object 7.2` child.rs 方法）
- 依赖：P0-4、P0-5
- DoD：☐ 规范定稿 ☐ 实现 ☐ 测试

> **说明（v8.0 重编号 P1-5 → P0-6）**：子 APO 委托属**驱动层基础能力**——`object 7.1.8`
> Initialize 步骤 4 本就要「创建 ChildApo」（失败降级为无子 APO），`object 7.1.3` child_apo 字段、
> `object 7.2` 委托方法均已规范——补实现是 P0 链路的完整性收尾，非 P1 核心功能扩展。
> EAPO 在 Initialize 中 `CoCreateInstance(子 APO GUID)` → QI 三接口 → 委托全部方法
> （v7.8 EAPO 源码二次检查确认）；VxAPO 规范 object 7.2 已有定义，实现尚缺。

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
- 反馈闭环：3 点实现反馈 → v7.4 规范修订（见 `feedback.md #P0-2-1/2/3`）

### P0-3  per-device 配置路径 — Done @ v7.6
- 规范落点：`sys 3.6`（known_folder）、`sys 3.3.1b`（APOInitSystemEffects，v7.6 实测路径）、`object 7.1.8`（per-device 路径 + 二级兜底）、`object 7.1.11`（过渡完成 reloading 修正）
- 实现：`src/sys/known_folder.rs`（新建）、`src/sys/com/apo_types.rs`（re-export）、`src/object/apo.rs`（extract_endpoint_guid/resolve_config_path/Initialize 重写 + 过渡修正）、`src/sys.rs`
- 验收：441 passed；路径/目录/文件/`_default` 兜底 4 项；真实端点 GUID 提取留手动验收
- 反馈闭环：APOInit 实测结构（v7.6）+ 过渡 3 点修正（v7.6 二次检查对齐）+ watcher 接线留 P0-4（v7.3 已定义）