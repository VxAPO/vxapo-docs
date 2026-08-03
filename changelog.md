# Changelog

## v8.1 — 2026-08-03

变更类型：`结构重构`（P0-6 子 APO 实现对齐——EAPO 源码精读 + 主规范新增「EAPO 对齐度与差异化」章节）

- **P0-6 开放决策收敛（EAPO 源码对齐）**：实读 EAPO `EqualizerAPO.cpp`/`FilterEngine`/`FilterConfiguration`——
  ① Unlock 失败语义（默认 VxAPO 容错）；② 双链过渡下 childRT 委托**前置每帧一次**（双链共享同一份 child 输出；child 不在任一链内）；③ 通道约束**方案 A**（child 输出==父链输入==最终输出；Lock 校验三方一致）——**对应章节**：`roadmap P0-6`
- **主规范新增「十八、EAPO 对齐度与差异化」**：对齐度 A1-A5（创建/QI/Initialize/降级/APOProcess 时序/GetLatency/就地缓冲）；
  差异化 D1-D5（父内双链 vs EAPO 单链配置级过渡 / realChannelCount 维度切换 vs VxAPO 显式约束 / child 输出通道数不可知的约定显式化 /
  Unlock 失败语义差异 / IsInputFormatSupported 委托失败回落）+ 差异化带来的设计影响（RT 成本叠加 3 次 DSP / 维度稳定前提 / 校验前置）——**对应章节**：`主规范 十八`

> *v8.1 补正说明：撤销「显式约束/显式优于 EAPO 隐式信任」表述（语义保留于 roadmap P0-6、主规范 18.2/18.3，已同步重写为「等价立场」）——经推演，EAPO 靠「协商期锁死 in==out + 协商委托 child + mono→stereo 补系统默认」构成完整通道语义；VxAPO 与之**等价**，唯一区别是不引入 realChannelCount 维度切换机制（协商后冗余/死路径），属实现简化非更严谨。*
>
> 对应 commit：`8e5c54f`

## v8.0 — 2026-08-03

变更类型：`结构重构`（治理机制 major 升级——反馈外置 feedback.md + feedback-rule）

- **反馈外置（roadmap 精简）**：执行端反馈统一记录于根目录 `feedback.md`（11 条迁移：P0-2-1/2/3、P0-3-1/2、P0-4-0/1/2/3/4）；
  `roadmap.md` 删除全部反馈/修订记录正文，条目只留状态/DoD/规范落点 + 引用（`> 反馈记录：feedback.md #PX-X`）——
  **对应章节**：`roadmap.md`、`feedback.md`（新建）
- **feedback-rule（新建规则）**：`.clinerules/feedback-rule.md`——反馈触发条件 / 记录格式（PX-X + 影响版本 + 问题 + 规范侧判定 + 修订记录 + 状态）/ 生命周期 / 与 roadmap-rule、主规范十七、changelog-rule 衔接——**对应章节**：`.clinerules/feedback-rule.md`
- **roadmap-rule 同步**：补「反馈外置（v8.0）」节——反馈只留引用，状态机/DoD 不受影响——**对应章节**：`.clinerules/roadmap-rule.md`
- **主规范十七更新**：反馈闭环语义不变（执行端不改状态 + 零容忍绕过 + 规范侧修订），反馈落点由 roadmap → feedback.md——**对应章节**：`主规范 十七`
- **P0-4 执行端待办汇总保留**：对象层接线 + v7.11/v7.12 严格化（roadmap P0-4 条目）——DoD 未全勾，状态保持 Spec-Finalized

> 对应 commit：`106729f`

## v7.12 — 2026-08-03

变更类型：`缺陷修复`（P0-4 二次反馈——「未知命令」判定失效 → 命令关键字白名单三段式）

- **命令关键字白名单（三段式）**：`split_command_value`（冒号数量）→ **命令名 ∈ 白名单**（静态命令 / REW `Filter N:` 前缀 / `registry.factory_names()`）→ `try_create`——
  修复 v7.11「Unmatched → SyntaxError」失效：`BogusCommand: x` 有冒号被 Convolution 宽容解析（任意非空=IR 路径）接管 → 永远到不了 Unmatched → 「未知命令」永不触发——
  **对应章节**：`config 6.1`（白名单小节）
- **Unmatched 语义收窄**：`try_create` Unmatched 仅表示「已知命令的参数无效」（命令已过白名单）——文案 `命令无效 'X'：参数无法解析`（原「未知命令」由白名单分支判定）——**对应章节**：`config 6.1`
- **VSTPlugin 特判**：白名单命中但功能未启用 → 不过 try_create，直接 `SyntaxError「命令无效 'VSTPlugin'：该命令当前未启用（预留）」`（诚实且准确定位）——**对应章节**：`config 6.1`、`pipeline 4.20`

> 对应 commit：`b26a324`

## v7.11 — 2026-08-03

变更类型：`结构重构`（P0-4 潜在问题反馈② → config 语法严格化 + 错误报告方案定稿）

- **config 语法严格化（用户产品决策）**：每行必须 `命令关键字: 参数`——无冒号行拒绝（SyntaxError「缺少冒号」）、
  一行仅一个冒号（SyntaxError「多余冒号」）；**v7.9 裸命令可达性修正反转**（无冒号不再 `try_create(cmd)`）——
  `BogusCommand` 不再被 Convolution 宽容语义误接——**对应章节**：`config 6.1`、`intent.md「config.txt 语法严格性」`
- **Unmatched → SyntaxError**：`registry.try_create` 返回 Unmatched（未知命令）→ 不再 `log::warn` 跳过，
  改为 `SyntaxError「未知命令」`整体失败（保留旧链）；「配置写错必有反馈」——**对应章节**：`config 6.1`
- **Convolution 参数严格化**：`parse_convolution_params` 从 `Option` → `Result`——≥3 tokens / 第 2 个非数值
  → `ParseError`（`ir.wav -6 abc` 不再静默忽略 `abc`）——**对应章节**：`pipeline 4.19`
- **VST 静默 NoMatch 对齐**：`VSTPlugin:` v7.11 起与未知命令同样落 SyntaxError（诚实反馈，不再静默跳过）——
  **对应章节**：`pipeline 4.20`
- **诊断日志归属（intent 固化）**：config 解析错误摘要写入**软件安装根目录 `log/`**（非 Documents\VxAPO 设备目录），
  由应用层读取呈现；DLL 只写日志（纯落盘，非状态回传通道），watcher 天然不监控 log/——**对应章节**：
  `intent.md「诊断日志归属」`

> 对应 commit：`b71f1f8`

## v7.10 — 2026-08-03

变更类型：`实现对齐`（P0-4 实现完成报告反馈①——watcher 线程模型澄清 + 对象层接线缺口确认）

- **ConfigWatcher 外部驱动模型澄清（执行端反馈①）**：`config 6.2` 原「new 启动 watcher 线程」与 `wait_and_handle`
  外部驱动矛盾——统一为 **`ConfigWatcher` 不自启线程**（new 只建句柄；线程由调用方 `object/apo.rs::start_watcher`
  创建并循环驱动），`shutdown` 不 join（join 由 `stop_watcher` 负责：SetEvent → join → close）——
  **对应章节**：`config 6.2`、`object 7.1.9`（start_watcher 定义）、`object 7.1.10`（stop_watcher 定义）、`object 7.1.3`（字段补 shutdown_event/watcher_thread）
- **P0-4 对象层接线缺口确认（执行端遗留 1）**：`apo.rs` 尚未创建 watcher 线程（LockForProcess 末尾 start_watcher
  + UnlockForProcess stop_watcher）——**属执行端 P0-4 剩余项**，规范本文件已完备，执行端补做后回归；
  P0-4 **不标记 Done**（热重载链路未实际接通）——**对应章节**：`object 7.1.9/7.1.10`、`roadmap P0-4`

> 对应 commit：`f860c10`

## v7.9 — 2026-08-02

变更类型：`结构重构`（P0-4 配置变更检测方案定稿——filter_spec 指纹 + 目录级事件驱动语义落定）

- **目录级语义澄清（执行端反馈 → 方案定型）**：`FindFirstChangeNotificationW` 是**目录级通知**、不提供具体文件名——
  旧 6.2「校验文件名 == config.txt」无法实现；统一为 `DirectoryChanged(watch_dir)`，hot_reload 内 spec 指纹比对决定是否真正切换——
  **对应章节**：`config 6.2`、`object 7.1.8`
- **filter_spec 配置指纹（用户方案）**：parser 分发层统一产出 `produce_spec(cmd, value)`（命令名小写 + `\x1F` 分隔 + token 级规范化）；
  `FilterSpec = String`（单一规范化字符串，含无冒号裸命令整行处理）；`parse_file_with_spec` 双返回 `(滤波器列表, SpecChain)`——
  **对应章节**：`config 6.1`
- **配置变更检测行为链**：目录变更 → 128KB 逐文件闸门 → 重新解析 + spec 比对（与 active_spec）→ 相同幂等跳过 / 不同构建新链 + 双链过渡；
  **Include 失败 = 整体解析失败**（不更新 active_spec）；active_spec 构建成功即更新——
  **对应章节**：`object 7.1.18`、`object 7.1.3`、`object 7.1.9`
- **watcher 生命周期随锁定周期**：Initialize 不再启动（未锁定无可放新链）；`LockForProcess` 末尾启动、`UnlockForProcess` 停止（对齐 EAPO startMonitorThread）——
  **对应章节**：`object 7.1.8/7.1.9/7.1.10`
- **裸命令可达性修正**：无冒号行（`PK Fc 1000`）当前 value="" → try_create 收空串 → Unmatched 到不了工厂；值空且非配置关键字时 `try_create(cmd)`（整行作参数）——
  **对应章节**：`config 6.1`
- **产品意图文档**：新建 `intent.md`（产品定位 / 用户行为模型——正常路径 CLI/UI 管理、手动改文件为边缘降级 / 驱动-应用层边界 / 治理地位：规范上位）——
  **对应章节**：`intent.md`（根目录新文档）

> 对应 commit：`f042966`

## v7.8 — 2026-08-02

变更类型：`外部借鉴`（EqualizerAPO 源码二次检查 → P0-4 热重载事件驱动 + 容错对齐）

- **ConfigWatcher 事件驱动**：`FindFirstChangeNotificationW` + `WaitForMultipleObjects` 替代 2000ms 轮询（延迟 10ms 级、无空闲 CPU）；**监控目录**（非文件——文件删除重建时句柄失效）；去重窗口 500ms→10ms（对齐 EAPO + Note 72）；shutdown_event 退出 + join——**对应章节**：`config 6.2`、`object 7.1.8`
- **热重载失败保留旧链**：解析失败不再变空链直出（EQ 消失），保留 current_chain + log::warn（对齐 EAPO loadConfig 失败不替换）——**对应章节**：`object 7.1.18`
- **过渡缓冲预分配**：LockForProcess 按 `max_frame_count × max_ch` 预分配 temp_buffer_old/new，杜绝 RT 首次过渡 `resize()` 扩容——**对应章节**：`object 7.1.9`
- **EAPO 对比澄清**：R1（退役链零析构）与 EAPO `previousConfig` 为**等价对齐**（EAPO 也在控制线程 loadConfig 析构，非我此前误述的"我们更优"）——**对应章节**：`object 7.1.3`（R1 注）

> 对应 commit：`2c304e4`

## v7.7 — 2026-08-02

变更类型：`缺陷修复`（实现缺陷反馈：IsInputFormatSupported 时序错误）

- **补关键时序约束（object 7.1.16）**：`IsInputFormatSupported`/`IsOutputFormatSupported` 在 `LockForProcess` 之前被引擎调用，此时 `pipeline_context` 为全零 `PipelineContext::new()`——**禁止依赖 pipeline_context 做等值比较**（真实格式 vs 全零永远不等 → 拒绝所有格式、APO 无法协商）。正确做法是对请求格式做**独立属性检查**（浮点格式 + 44.1k~192k + 1~8 通道），属性在协商时已确定、与锁定后上下文无关——**对应章节**：`object 7.1.16`

> 对应 commit：`5da8b60`

## v7.6 — 2026-08-02

变更类型：`实现对齐`（P0-3 实现反馈闭环 → APOInitSystemEffects 提取路径实测化）

- **APOInitSystemEffects 提取路径修订**：`pSystemEffectsProperties->pEndpointGuid` → `pAPOSystemEffectsProperties`（`IPropertyStore`）取 `PKEY_AudioEndpoint_GUID`（PROPVARIANT VT_CLSID 的 `puuid`）——windows-rs 0.62.2 实测结构（P0-3 实现反馈①）——**对应章节**：`object 7.1.8`、`sys 3.3.1b`
- **无 GUID 兜底描述同步**：`PKEY_AudioEndpoint_GUID` 提取失败/为空时回退 `_default`（原 `pEndpointGuid` 残留修正）——**对应章节**：`object 7.1.8`
- **apo.rs 二次检查对齐（规范侧实读 881 行）**：① 7.1.11 过渡完成不再先置 `reloading=true` 再调 hot_reload（会短锁拦截自身），改为不置位直接调用 + pending 残留 bypass 防御 + advance None→factor=1.0；② 7.1.8 Initialize 非法数据降级默认配置仍返回 Ok（非 E_INVALIDARG，SDK 容错）；③ 7.1.8 补 `documents_folder()` 失败→固定 `C:\ProgramData\VxAPO\config.txt` 二级兜底——**对应章节**：`object 7.1.8/7.1.11`
- **P0-3 per-device 配置路径进入 Done**：实现完成（441 passed，含 known_folder/config_path 测试）+ 合规核对通过；watcher 接线留待 P0-4（v7.3 约定，属 P0-4 职责）

> 对应 commit：`ed8557b`

## v7.5 — 2026-08-02

变更类型：`实现对齐`（正式 CLSID GUID 落定）

- **正式 GUID 替换占位值**：`CLSID_VXAPO_PRE_MIX = 41C34613-D391-459D-A039-72B2B15A1A1D`、
  `CLSID_VXAPO_POST_MIX = B4A97313-ABC0-45ED-9C33-428B20D39428`（用户确定）；
  涉及 `vxapo.def` / `sys/com/apo_types` / `install/device/slots` / `object/vx_reg_props` 同步更新——**对应章节**：`object 7.5`

> 对应 commit：`a243e7f`

## v7.4 — 2026-08-02

变更类型：`缺陷修复`（P0-1/P0-2 实现反馈闭环 → config 规范修订）

- **实现反馈闭环机制**：新增「十七、实现反馈闭环」——执行端实现中发现规范不可行/遗漏 → 追加反馈段（不改状态）→ 规范侧修订 → 合规核对后标记 Done；零容忍绕过——**对应章节**：`主规范 十七`
- **config 6.1 修订 current_file**：`ParseContext.current_file: &'a Path` → `PathBuf`（Include 子解析需独立持有子文件路径，借用跨递归层不安全，旧 `Box::leak` 致泄漏）——**对应章节**：`config 6.1`
- **config 6.1 补充 REW 动态命令名分发**：REW `Filter N:`（如 `Filter 12:`）命令关键字动态，静态 match 无法命中，补充 `starts_with("filter ")` 前缀分支——**对应章节**：`config 6.1`
- **config 6.3 注册意图澄清**：`register_all_commands` 只注册 DSP 工厂——纯配置命令由 6.1 静态分发（`FilterFactory::create_filter` 只收 value 不含命令关键字，config 命令工厂注册后永远无法命中）——**对应章节**：`config 6.3`

> 对应 commit：`bf7a08e`

## v7.3 — 2026-08-01

变更类型：`实现对齐`（P0-4 配置热重载全链路规范补齐）

- **watcher 启动约定**：Initialize 中按 `config_path` 父目录启动 `ConfigWatcher`（轮询 2000ms、去重 500ms，`config 6.2`）；`ConfigFileChanged/Deleted` 事件经 `hot_reload`（R2 阻塞式 + R1 退役链）处理——**对应章节**：`object 7.1.8`
- **watcher 生命周期**：与 APO 实例一致（`ApoObject.watcher` 字段持有）；`UnlockForProcess`/`Reset` 不停止 watcher（热重载跨锁定周期持续生效）——**对应章节**：`object 7.1.8`

> 对应 commit：`a2c8d6c`

## v7.2 — 2026-08-01

变更类型：`实现对齐`（P0-3 per-device 配置路径规范补齐）

- **sys 新增 known_folder**：`sys/known_folder.rs` 封装 `SHGetKnownFolderPath(FOLDERID_Documents)` 已知文件夹解析（RAII 释放 CoTaskMem 内存），只做 FFI 收窄不拼接路径——**对应章节**：`sys 3.6`
- **APOInitSystemEffects re-export**：`sys/com/apo_types` 增加 `APOInitSystemEffects`（Initialize 初始化数据，提取端点 GUID）——**对应章节**：`sys 3.3.1b`
- **Initialize 补全 per-device 路径解析**：`APOInitSystemEffects` 反查端点 GUID → `guid_to_string` 大写格式化 → `Documents\VxAPO\{GUID}\config.txt`；目录自动创建、config 缺失写默认 passthrough、无 GUID 兜底 `_default`——**对应章节**：`object 7.1.8`
- **引用约束总表同步**：主规范增加 `sys/known_folder.rs` 行、`object/apo.rs` 增 `sys/known_folder` 依赖——**对应章节**：`主规范 十一`

> 对应 commit：`8dd42e2`

## v7.1 — 2026-08-01

变更类型：`实现对齐`（P0-1 DllRegisterServer 规范补齐）

- **regsvr32 职责边界澄清**：`DllRegisterServer` 无设备参数，只做全局 COM 类注册（2 个 CLSID 的 COM 类键 + ThreadingModel）；设备挂载/FxProperties 绑定归 `install_endpoint`，经 `vxapo-cli install -d` 触发——**对应章节**：`object 7.6`
- **DllRegisterServer 完整流程**：注册顺序（PostMix→PreMix）+ 幂等覆盖 + 失败逆序回滚（`SELFREG_E_CLASS`）+ 禁止触碰 MMDevices/FxProperties——**对应章节**：`object 7.6`
- **DllUnregisterServer 幂等**：键不存在视为成功（重复 `regsvr32 /u` 安全），尽力清理——**对应章节**：`object 7.6`
- **dll_exports 依赖补全**：引用约束总表增加 `sys/registry`（CLSID 键写入）——**对应章节**：`主规范 十一`、`object 7.6`

> 对应 commit：`895d9cd`

## v7.0 — 2026-08-01

变更类型：`结构重构`（引入路线清单治理机制，v6.9 → v7.0 major 递增）

- **路线清单驱动**：新增 `roadmap.md`（"要做什么"的规划），登记 P0-1..4 / P1-1..4 首版条目；任何新功能必须先登记，禁止"规范外直接实现"——**对应章节**：`主规范 十六`
- **状态机 + 硬门禁**：`Backlog → Spec-Drafting → Spec-Finalized → Implementing → Done`；仅 `Spec-Finalized` 可进入 `Implementing`（roadmap-rule 强制）——**对应章节**：`主规范 十六`、`.clinerules/roadmap-rule.md`
- **三文件联动**：一次版本变更须同时满足 roadmap 状态更新 + 规范章节落地 + changelog 记录，changelog 版本号 == 主规范版本号 == roadmap 落点版本——**对应章节**：`主规范 十六`、`.clinerules/roadmap-rule.md`
- **配套 skill 修正**：`roadmap-add-item`（仅登记，Backlog 不写 changelog，依赖检查推迟到 Spec-Drafting 前）；`spec-finalize`（次版本默认递增、changelog 严格走 changelog-rule、补三文件一致性校验、无法合规停留 Spec-Drafting）——**对应章节**：`.clinerules/skills/roadmap-add-item/SKILL.md`、`.clinerules/skills/spec-finalize/SKILL.md`

> 对应 commit：`f59f623`

## v6.9 — 2026-08-01

变更类型：`外部借鉴`（EqualizerAPO FilterEngine 过渡/重载机制深度分析）

- **R1 退役链延迟析构**：过渡完成帧 RT 线程仅 `retired_chain = outgoing_chain.take()`（零析构），控制线程锁内统一 drop——重型滤波器析构绝不留在 RT 线程——**对应章节**：`object 7.1.3/7.1.9/7.1.10/7.1.11/7.1.13`
- **R2 阻塞式重载**：hot_reload 检测到过渡在途/加载中即返回不构建，过渡完成由 APOProcess 触发重载；`reloading` 标志防覆盖——**对应章节**：`object 7.1.3/7.1.11/7.1.18`
- **R3 空链快路径**：`Chain::is_empty()` 时 `process_audio` 直接复制去交织结果跳过链遍历——**对应章节**：`pipeline 4.5/4.6`
- **R4 过渡周期 10ms**：`default_smoothing_length` 由 `sample_rate/20`（50ms）→ `sample_rate/100`（10ms）——**对应章节**：`pipeline 4.11`

> 对应 commit：`7aa6f16`

## v6.8 — 2026-08-01

变更类型：`外部借鉴`（EqualizerAPO Device 层）

- **E3.1 默认设备判定边界**：driver 层 `enumerate_devices()` 仅返回合法安装容器，不判定默认设备；用户层经 COM `GetDefaultAudioEndpoint` 对应——**对应章节**：`install 5.4`
- **E3.2 设备物理状态谓词**：`DeviceInfo::is_disabled()` / `is_unplugged()`（由已有 `EndpointState` 推导）——**对应章节**：`install 5.4`
- **E3.3 autoAdjust 独立字段**：`InstallConfig::auto_adjust`（默认 false），Step 4 写注册表读取——**对应章节**：`install 5.5.2`
- **E3.4 安装自检**：`install_endpoint(..., verify)`——commit 后 `CoCreateInstance` 验证 DLL 可实例化——**对应章节**：`install 5.5.2`
- 修复 install 5.5.2 多余代码围栏（用户修复）——**对应章节**：`install 5.5.2`

> 对应 commit：`bac344e`（+ 用户围栏修复 `b162890`）

## v6.7 — 2026-07-31

变更类型：`外部借鉴`（EqualizerAPO `IFilter::getInPlace`）

- **E1 就地处理声明**：`Filter::is_in_place()`（默认 true）+ `Chain::is_fully_in_place()`——全链就地时 `temp_buffers` 即最终输出，零拷贝快路径——**对应章节**：`pipeline 4.9/4.5/4.6`
- 与 v6.6 的 O1（RealtimeContext）/ 去交织架构完全兼容——**对应章节**：`pipeline 4.7/4.9`

> 对应 commit：`415ef53`

## v6.6 — 2026-08-01

变更类型：`外部借鉴`（tympan-apo）

- **O1 RT 编译期见证**：`RealtimeContext` 零尺寸标记 + `DspContext::rt_marker`（`PhantomData<RealtimeContext>`）——编译期能解决的问题绝不拖到运行时——**对应章节**：`pipeline 4.7/4.9`
- **O2 StateCell 补全**：`release()`（任意态→Created）+ `TransitionError{expected, attempted, actual}` + 语义化转换——**对应章节**：`object 7.1.4`
- **O3 生产构建约束**：release 必须 `panic="abort"` + `codegen-units=1`（RT 跨 FFI unwind = UB）——**对应章节**：`主规范 十五`
- **O4 AEC 接口预留**：`feature = "aec"` 门控 3 个 AEC 接口 + IID 常量——**对应章节**：`sys 3.2`

> 对应 commit：`a6ca7b5`

> **范围说明**：本 changelog 自 v6.6 开始记录（v6.0-v6.5 不补录）。维护规则见 `.clinerules/changelog-rule.md`。