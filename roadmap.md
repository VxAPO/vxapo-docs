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
- 状态：Done（v8.0 合规核对通过）
- 优先级：P0 ｜ 关联 Phase：Phase 10
- 目标：监控线程检测 config.txt 变更 → swap 串联 → 升余弦过渡；修改文件实时生效且无爆音（对齐 v6.9 R1-R4）
- 影响模块：`config/watcher.rs`、`config/parser.rs`（filter_spec 产出，v7.9）、`object/apo.rs`（hot_reload/APOProcess）
- 规范落点：`config 6.1`（filter_spec 契约 + parse_file_with_spec + 128KB 逐文件闸门 + 单冒号校验 + 白名单三段式，v7.9/v7.11/v7.12）、`config 6.2`（目录级事件驱动 + DirectoryChanged + 外部驱动模型，v7.8/v7.9/v7.10）、`object 7.1.8`（watcher 生命周期随锁定周期，v7.9）、`object 7.1.9`（末尾启动 watcher + active_spec 基线 + start_watcher 定义，v7.9/v7.10）、`object 7.1.10`（stop_watcher 定义，v7.10）、`object 7.1.18`（spec 短路 + 保留旧链，v7.9）、`pipeline 4.19/4.20`（Convolution strict + VST 特判，v7.11/v7.12）、`intent.md`（产品意图 + 语法严格性 + 诊断日志归属）
- 依赖：P0-3（已 Done ✅）
- DoD：☑ 规范定稿（v7.3/v7.8/v7.9/v7.10/v7.11/v7.12）☑ 实现 ☑ 测试（436 passed；听感验证④ 留 P0-7 后手动验收）

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
> - ① **对象层接线**（v7.10 确认）——**已完成**（commit 77bc68e）：ApoObject 加
>   `watcher_state: Arc<Mutex<WatcherState>>`（`#[implement]` 无 &mut，方案 A）；
>   `LockForProcess` 末尾 `start_watcher()`（CreateEventW → spawn 循环 wait_and_handle →
>   hot_reload_impl）；`UnlockForProcess` `stop_watcher()`（SetEvent + join + close）；
>   `ConfigWatcher` `unsafe impl Send`（HANDLE 句柄跨线程合法）。 ✅
> - ② **v7.11 严格化**——**已完成**（commit 7b20619）：`split_command_value` 单冒号校验
>   （缺/多余冒号 SyntaxError）；`_` 分支 Unmatched → SyntaxError；`parse_convolution_params`
>   `Option`→`Result`（Empty/TooManyTokens/InvalidGain）；测试断言更新
>   （spec_with_unknown_command_reports_error 等）。错误日志写 `log/`（待 P0-7 CLI）。
> - ③ **v7.12 白名单**——**已完成**（commit 7b20619）：`is_known_dsp_command` 白名单校验
>   （未知命令 → SyntaxError，不落 registry）+ VSTPlugin 特判「未启用（预留）」+
>   Unmatched 收窄「命令无效 'X'：参数无法解析」；`spec_with_unknown_command_reports_error`
>   已转绿。
> - ④ 真实音频引擎验证（无爆音）留手动验收——**待 P0-7 CLI（install_endpoint/FxProperties
>   挂载）落地后联调**（见 P0-7 验证边界：「真正 DSP 热重载生效由 P0-4 手动听感验证覆盖」）。

### 合规核对记录（v8.0）
- 核对结果：**通过**——实现完成报告自查（RT 无违规/引用约束无打破/无未声明依赖）与规范落点一致；
  触碰文件 ⊆ 影响模块（config/parser.rs、config/watcher.rs、object/apo.rs）；测试 436 passed + check 0 error 0 warning。
- 待办 ①（对象层接线）②③（v7.11/v7.12 严格化）：全部完成（commit 77bc68e / 7b20619）；
  `Arc<Mutex<WatcherState>>` 聚合状态与规范 7.1.3 独立字段**语义等价**（规避 `#[implement]` 无 &mut，方案 A 可接受）。
- 听感验证 ④ 遗留：**非实现缺口**（需真实 audiodg + P0-7 CLI FxProperties 挂载后联调）——参照 P0-1 先例（手动验收留真实环境仍标记 Done）判定 Done。
- 已归档至本文件「已完成」区（保留章节号便于追溯）。

### 实现完成报告（v7.9-v7.12，执行端）
- DoD：☑ 实现 ☑ 测试（436 passed / 0 failed；cargo check 0 error 0 warning）
- 自查结果：
  - RT 无违规：watcher 线程（控制路径，I/O/锁允许）+ hot_reload 锁外解析；
    APOProcess 过渡段保持零分配（Lock 预分配 temp_buffer）；退役链由控制线程析构（R1）。
  - 引用约束无打破：config/parser.rs、config/watcher.rs、object/apo.rs 均 ⊆ 影响模块；
    watcher.rs 增 `Win32_Storage_FileSystem`（config 6.2 目录监控必需）、apo.rs 增
    `config/watcher`（object 7.1.9 已声明）；未触碰 install/、pipeline/ 其它模块。
  - 未引入未声明依赖：windows features 已含 Win32_Storage_FileSystem/System_Threading/
    Foundation（v7.8/v7.10 已声明）。
- 实现文件（⊆ 影响模块）：
  - `src/config/parser.rs`（v7.9-v7.12）：FilterSpec/SpecChain/MAX_CONFIG_FILE_SIZE/
    produce_spec/parse_file_with_spec + v7.11 单冒号校验 + Unmatched→SyntaxError +
    v7.12 白名单三段式（is_known_dsp_command + VSTPlugin 特判 + 文案收窄）
  - `src/config/watcher.rs`（v7.8-v7.10）：事件驱动（DirectoryChanged/wait_and_handle/
    shutdown/poll_registry）+ unsafe impl Send
  - `src/object/apo.rs`（v7.10）：watcher 线程接线（Arc<Mutex<WatcherState>> +
    start_watcher/stop_watcher + hot_reload_impl 模块级）——commit 77bc68e
- 遗留问题：
  1. 真实音频引擎听感验证（P0-4 待办 ④）依赖 P0-7 CLI 提供 FxProperties 挂载（install_endpoint）；
     当前无端点 GUID 获取/挂载工具，无法在真实 audiodg 验收。
  2. 错误日志写 `log/`（intent「诊断日志归属」）——待 P0-7 CLI/应用层呈现，DLL 仅 log。
- 反馈引用：`feedback.md #P0-4-0`（v7.7 时序）、`#P0-4-1`（v7.9 目录级）、`#P0-4-2`
  （v7.10 线程模型）、`#P0-4-3`（v7.11 严格化）、`#P0-4-4`（v7.12 白名单）——全部已修订。
- 状态：仍 Spec-Finalized（DoD 实现/测试 ☑；真实听感验证 ④ 未勾——由规范侧核对后决定）

### P0-5  RT 入口 panic 防护（catch_unwind）
- 状态：Done（v8.7 合规核对通过）
- 优先级：P0 ｜ 关联 Phase：Phase 1-9
- 目标：RT 三入口（`APOProcess` / `CalcInputFrames` / `CalcOutputFrames`）panic 防护——**杜绝 panic 跨 FFI unwind 的 UB 传播**；三层防护各司其职（编译期约束为源头 → debug 测试态 catch_unwind 验证防御路径 → release abort 确定性兜底）
- 影响模块：`object/apo.rs`（RT 三入口）
- 规范落点：`object 7.1.11`（APOProcess 三层防护 + debug 态捕获行为）、`object 7.1.12`（CalcInput/OutputFrames 三层防护 + debug 态保守返回值）、`主规范 十五`（O1 RT 不 panic + O3 panic=abort）、`telemetry 9.2`（panic hook）
- 依赖：无
- DoD：☑ 规范定稿（v8.2，v8.3 语义澄清）☑ 实现 ☑ 测试（431 passed + 2 panic 防护测试）
> 反馈记录：feedback.md #P0-5-1（v8.3 定稿说明语义失准——「杜绝崩溃」→「杜绝 UB 传播」）

### 合规核对记录（v8.7）
- 核对结果：**通过**——执行端实现完成报告（e2fb954）DoD 全勾（实现 + 测试 431 passed）；RT 无违规（catch_unwind 不跨函数边界 + panic 兜底零分配）；引用约束无打破（仅 object/apo.rs ⊆ 影响模块）；遗留无
- 归档：已移入「已完成」区（保留规范落点便于追溯）

### 实现完成报告（2026-08-04，执行端 e2fb954）
- DoD：☑ 实现 ☑ 测试（431 passed / 0 failed；新增 2 个 panic 防护测试）
- 自查结果：
  - RT 无违规：catch_unwind 包裹在实现体内不跨函数边界；panic 兜底路径输出清零 + BUFFER_SILENT + error_count++ 零分配（telemetry 定长环形缓冲）
  - 引用约束无打破：仅改 object/apo.rs（⊆ 影响模块）；debug panic=unwind 测试态验证防御路径；release panic=abort 下 catch_unwind 空操作（O3）
  - 未引入未声明依赖：无
- 实现文件（⊆ 影响模块）：src/object/apo.rs（apo_process_inner 提取 + catch_unwind 包裹 + CalcInput/OutputFrames 保守值）
- 遗留问题：无

> **定稿说明（v8.2 初稿；v8.3 策略重评——语义澄清）**：
>
> **P0-5 策略合理性（v8.3 重评）**：P0-5 的根本价值**不在 catch_unwind 本身**，而在三层防护组合——
> 1. **编译期约束（第一道，源头，O1）**：RT 路径**无分配/无锁/无 IO**，正常路径**不 panic**——这是「避免崩溃」的真正源头；
> 2. **debug 测试态 catch_unwind（第二道，`panic="unwind"`）**：捕获 → 安全降级输出，**验证「即便 panic 也不跨 FFI 传播」的防御路径**（DoD 测试即此态）；
> 3. **release 兜底（第三道，`panic="abort"`，O3）**：任何漏网 panic → **确定性进程终止**——将「跨 FFI unwind = UB（静默内存损坏）」变为「可恢复的进程重启」，**绝无 UB 传播**；`telemetry/panic.rs` panic hook 在 abort 前记录栈——**它是诊断工具，不是防线**。
>
> **对「杜绝 audiodg 崩溃」的澄清（v8.3）**：原目标表述「杜绝 panic 跨 FFI unwind 到 audiodg 崩溃」**失准**——
> - release（`panic="abort"`）下，panic 即**进程终止**（audiodg 由 Windows 音频服务重启恢复），**不是「崩溃」而是「确定性终止 + 无 UB」**；真正的危害是 **unwind 跨 FFI = UB（内存损坏/静默错乱）**，故目标应为**「杜绝 UB 传播」**；
> - 「abort 终止进程」是**兜底语义**，不是防线缺陷：与其让 UB 静默传播，不如确定性终止可恢复；
> - panic hook 的价值是**事后诊断**（abort 前记录现场），不改变 abort 本身的兜底角色。
>
> **catch_unwind 语义（v8.3 澄清）**：
> - **debug profile（`panic="unwind"`）**：跨 `extern "system"` FFI 边界 unwind 是 UB——RT 三入口 `catch_unwind` 包裹为**测试态防御路径**（捕获 → 安全降级输出，验证不向 audiodg 传播）；
> - **release profile（`panic="abort"`，O3 主规范十五）**：`catch_unwind` 为空操作（编译移除零开销），panic 即 abort 兜底。
>
> - **捕获行为（仅 debug 态生效）**：
>   - `APOProcess`：panic → 输出缓冲清零 + `buffer_flags = BUFFER_SILENT` + `stats.error_count++` + 日志（RT 零分配：`log::error!` 走 telemetry 定长环形缓冲）；
>   - `CalcInputFrames`：panic → 返回 `output_frames + last_known_latency`（保守多请求输入帧，不 panic）；
>   - `CalcOutputFrames`：panic → 返回 `0`（保守，可丢帧不可越界）。
> - **cost**：仅 debug/测试态有实际路径；release 零开销（`catch_unwind` 编译移除，O3）。
> - **实现要点**：三入口均为 `extern "system"`（`#[implement]` 生成）——`catch_unwind` 包裹在实现体内、**不跨函数边界**；panic 载荷 `Box<dyn Any>` 经 `downcast_ref::<&str>` 提取消息写日志。
> - **测试**：debug 下用 `std::thread::spawn` 模拟 panic（或 `#[cfg(test)]` 注入 panic 滤波）验证捕获不跨 FFI 传播、输出被置 Silent。
>
> **说明（v8.3 重评结论）**：旧说明「release 真防线是 panic hook + abort」**表述失准已修正**——release 防线是 **abort 兜底**（确定进程终止、无 UB），panic hook 仅为诊断工具非防线；catch_unwind 是 debug/测试态验证路径。策略本身合理：三层防护组合达成「**源头不 panic → 测试态验证防御 → release 确定性兜底**」。属 P0 尾巴（威胁 audiodg 稳定性）。

### P0-6  子 APO 委托实现（object/child.rs 落地）
- 状态：Done（v8.7 合规核对通过）
- 优先级：P0 ｜ 关联 Phase：Phase 10T
- 目标：`object/child.rs` 规范已完备（三接口类型化持有 + 委托），实现缺——补 `ApoObject.child_apo` 字段 + CoCreateInstance + Initialize/LockForProcess/UnlockForProcess/APOProcess 完整委托（对齐 EAPO childAPO/childRT/childCfg）
- 影响模块：`object/child.rs`、`object/apo.rs`、`install/device/slots.rs`（子 APO GUID 读取 + child_apo_key_exists 全量判定）
- 规范落点：`object 7.2`（子 APO 来源 = 接管槽位前任 + 应用层槽位失守检测（**v8.5 精确定性：安装模式槽位 + 覆盖备份 + 全量/非全量判定**）+ 运行期前置委托，v8.1/v8.4/v8.5）、`object 7.1.3`（child_apo 字段）、`object 7.1.7/7.1.8`（格式协商委托/Initialize 创建降级——**v8.4：子 APO GUID 来源 = 端点 GUID → `HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}\{PreMixChild|PostMixChild}`，APOInitSystemEffects 无子 APO 字段**）、`object 7.1.9/7.1.10`（Lock 委托/Unlock 容错）、`object 7.1.11`（APOProcess child 前置）、`install 5.3/5.5.2`（v8.4：VxAPO 独立安装信息区 + 路径隔离；**v8.5：`CHILD_APO_PATH_ROOT` + `child_apo_key_exists` + 全量/非全量判定 + 卸载删键**）、**`主规范 十八`（EAPO 对齐度与差异化 18.1-18.4，v8.1——开放决策 ①②③ 收敛依据）**
- 依赖：P0-4、P0-5（均 Done ✅）
- DoD：☑ 规范定稿（v8.1，v8.4 子 APO GUID 来源修正 + v8.5 槽位失守检测精确定性）☑ 实现 ☐ 测试（child 委托链完整测试留 P0-7 CLI 端到端——真实环境缺口，同 P0-4 听感验证先例）
> 反馈记录：feedback.md #P0-6-1（v8.3 EAPO 源码逐行查验揭示 5 处规范偏差 S1-S5）、#P0-6-2（v8.4 子 APO GUID 来源三处规范内部冲突消解 + 路径隔离）、#P0-6-3（v8.5 槽位失守检测 + 全量/非全量备份判定——安装模式槽位检测 + 覆盖备份 childapo + childapo 键存在性判定 + 卸载必删键）、#P0-6-4（v8.6 child.rs is_input/output_format_supported 输入参数 `*mut` → `Option<&>`——执行端建议采纳，输出 `*mut *mut` 保留）

### 合规核对记录（v8.7）
- 核对结果：**通过**——执行端实现完成报告（e2fb954）实现全勾（object/child.rs + object/apo.rs + install/device/slots.rs，均 ⊆ 影响模块）；RT 无违规（child 前置独立锁短持无死锁 + 委托不分配）；引用约束无打破（apo.rs 增 install/device/slots 依赖已声明 v8.4）；431 passed
- 测试缺口：child 委托链完整测试需真实 COM + 已注册 APO 无法单测——**非实现缺口**（真实环境依赖），参照 P0-1/P0-4 先例（手动/联调验收留真实环境仍标记 Done）判定 Done，端到端联调留 P0-7 CLI
- 缺陷说明：原 7 个 null 接口防御测试因类型化方案 Drop 对 null Release 解引用 vtable 崩溃（STATUS_STACK_BUFFER_OVERRUN）删除——**类型化安全边界**，注释已留（执行端 e2fb954）
- 归档：已移入「已完成」区（保留规范落点便于追溯）

### 实现完成报告（2026-08-04，执行端 e2fb954）
- DoD：☑ 实现 ☐ 测试（431 passed / 0 failed；child 委托链需真实 COM + 已注册 APO 无法单测——留 P0-7 CLI 端到端）
- 自查结果：
  - RT 无违规：childRT->APOProcess 前置在锁 inner 前（独立 child 锁短持，无死锁）；委托不分配
  - 引用约束无打破：object/child.rs、object/apo.rs、install/device/slots.rs 均 ⊆ 影响模块；apo.rs 增 install/device/slots 依赖（object 7.1.8 v8.4 引用来源已声明）
  - 未引入未声明依赖：无
- 实现文件（⊆ 影响模块）：
  - src/install/device/slots.rs（v8.4/v8.5）：CHILD_APO_PATH_ROOT + child_apo_key_exists + read_child_apo_guid + ChildApoKind（独立安装信息区，路径隔离）
  - src/object/child.rs（v8.6）：三接口类型化持有（windows-rs cast）+ 全部委托方法 + v8.6 格式协商 Option 参数
  - src/object/apo.rs：child_apo 字段 + Initialize 从端点 GUID 反查安装信息区创建（失败降级 None）+ APOProcess 前置 + GetLatency 委托（无 child 返回 0）+ Lock/Unlock 委托（失败不阻塞父）
- 遗留问题：
  1. child 委托链完整测试需真实 COM + 已注册 APO，无法单元测试（与 P0-4 听感验证同理）——留 P0-7 CLI 端到端
  2. 说明：原 child.rs 7 个 null 接口防御测试在类型化方案下因接口 Drop 对 null 引用调用 Release 解引用 vtable 崩溃（STATUS_STACK_BUFFER_OVERRUN），已删除并留注释说明

> **定稿说明（v8.1，EAPO 源码精读闭环 + 用户决策；v8.4 路径隔离修正）**：
> - **子 APO 来源** = 安装时被 VxAPO 接管槽位的**前任 APO**（`PreMixChild/PostMixChild` 存 **VxAPO 独立安装信息区** `HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}`——对齐 EAPO DeviceAPOInfo **机制**但**路径隔离**：**禁止**复用 EAPO `HKLM\SOFTWARE\EqualizerAPO\Child APOs`（RegistryHelper.h 33），避免污染 EAPO 安装信息区；备份全部槽位供回退、子 APO 仅对应实际装入槽位）。
> - **槽位失守检测 = 应用层**（CLI/GUI 启动/切换设备时检测槽位非 VxAPO CLSID → 提示重装 → 重装前把**当前**槽位备份为新 childapo「最新前任」）；**watcher 不负责**（对齐 EAPO Configurator 检测安装态）。
> - **无需注册表监视**：VxAPO config 纯文件（无 readReg 命令），EAPO watchRegistry 是 readRegString/readRegDWORD 副作用（RegistryFunctions.cpp 52/92）。
> - **运行期委托**：childRT->APOProcess 前置每帧一次（双链共享其输出）；child 不在 current/outgoing 任一链内；child 输出通道语义对齐父 outFormat（主规范 18.2 D2/D3 等价立场）。
> - **Unlock 容错 + 重置防御**：child 解锁失败 → 父继续解锁（void+HRESULT 无重试语义已核证）+ child 标记需重置 → 下次 Lock 前 child.reset()/重建。

> **说明（v8.0 重编号 P1-5 → P0-6 + EAPO 源码对齐 2026-08-03）**：子 APO 委托属**驱动层基础能力**——
> `object 7.1.8` Initialize 步骤 4 本就要「创建 ChildApo」（失败降级为无子 APO），`object 7.1.3` child_apo 字段、
> `object 7.2` 委托方法均已规范——补实现是 P0 链路的完整性收尾，非 P1 核心功能扩展。
>
> **EAPO 实现对齐（实读 EqualizerAPO.cpp，v8.0）**：
> - **创建/QI/Initialize（180-215）**：`CoCreateInstance(子 GUID, CLSCTX_INPROC_SERVER, IID_IAudioProcessingObject)`
>   → QI `IAudioProcessingObjectRT` → QI `IAudioProcessingObjectConfiguration` → **`childAPO->Initialize(cbDataSize, pbyData)`**
>   **同传父 APOInit 数据**；任一步失败 → `resetChild()`（三接口 Release）+ **返回 S_OK 降级为无子 APO**（不阻塞父 Initialize）——
>   与 VxAPO 7.1.8「失败降级不阻塞」一致 ✅。
> - **GetLatency（91-94，v8.3 修正）**：有 child → 委托 `childAPO->GetLatency(pTime)`；**无 child 返回 0**（EAPO `*pTime=0` 后仅 child 委托改写——v8.3 修正原「走自身」误读，见 feedback.md #P0-6-1 S4）。
> - **IsInputFormatSupported（258-272）**：有 child → 委托 child；child 失败 / 无 child → 自身格式检查（当前实现需核对是否按此委托）。
> - **LockForProcess（341-347）**：有 childCfg → `childCfg->LockForProcess`（**结果仅 Trace 不 return**，child 锁定失败不阻塞父）；随后父自身锁定。
>   **realChannelCount 分支（365-369）**：有 child 时 `realChannelCount = outFormat 通道数`（子 APO 可能改变输出通道语义）——
>   VxAPO 引擎初始化需对齐（当前硬性要求 input==output 通道数，有 child 时需放宽/按 child 输出语义）。
> - **APOProcess（472-478）**：**childRT->APOProcess 先跑**（作用于输入缓冲区）→ VxAPO `engine.process` 再处理其输出
>   （child 输出即 VxAPO 输入）；无 child → 纯 VxAPO 处理。**需确认与 VxAPO 双链过渡的协作时序**
>   （过渡期 outgoing/current 双链时 child 委托应挂在哪条链——EAPO 无 VxAPO 式过渡，此为 VxAPO 特有需设计）。
> - **UnlockForProcess（393-401）**：childCfg->UnlockForProcess；**FAILED → return hr**（父解锁失败）——
>   **与 VxAPO 7.1.10「子解锁失败不阻塞父解锁」冲突**，需 P0-6 定稿决策（EAPO 严格 / VxAPO 容错）。
> - **resetChild（406-424）**：三接口 Release + 置 NULL（Drop 语义等价）。
>
> **VxAPO 特有设计差异（EAPO 未直接覆盖）**：
> ① 双链过渡下 childRT 委托时序（过渡期两链各委托一次？child 状态在过渡窗口怎么保持？）；
> ② IR 卷积等 VxAPO 自有 DSP 与 child 的输出级联顺序（child 输出 → VxAPO Filter 链）；
> ③ child 存在时 realChannelCount 语义（允许 input≠output 通道数？——P0 现状强制相等需修订或注明边界）。
>
> **实现范围**：`object/child.rs`（create/三接口持有/委托方法）+ `object/apo.rs`（child_apo 字段接线 +
> Initialize 创建 + GetLatency/IsInOutFormatSupported/LockForProcess/UnlockForProcess/APOProcess 委托 +
> 降级 resetChild 语义）；`install/device/slots.rs` 子 APO GUID 读取、FxProperties 槽位解析。
> **DoD 测试要点**：create 失败降级（无 child 正常）、child 委托链完整（单 child）、
> child 存在 + 过渡期 APOProcess 时序、Unlock 失败语义（按 P0-6 定稿决策）。
>
> **开放决策（待定稿，v8.1 更新）**：
> - ① **Unlock 失败语义**（EAPO 严格 return hr / VxAPO 容错不阻塞父）——P0-6 定稿时决策（默认 VxAPO 容错，7.1.10 一致）。
> - ② **双链过渡下 child 委托时序**——**v8.1 收敛**：childRT->APOProcess **前置每帧一次**（双链共享同一份 child 输出作输入），
>   child 不在 current/outgoing 任一链内（child 独立持有，过渡只切父内两链）；不需要"过渡期 child 双实例"设计。
> - ③ **有 child 时 realChannelCount/通道数约束**——**v8.1 收敛为等价立场**（EAPO realChannelCount 语义澄清 + v8.1 补正）：
>   - **APO 链格式授予语义**：in/outFormat 是父 APO 两个独立连接描述符（引擎分别授予）。
>   - **无 child**：父承诺不转换（协商拒 in>out → in==out），`realChannelCount = inFormat`。
>   - **有 child**：父把 in/out 描述符原样传 child（child 输入=in、输出=out，child 就地负责 in→out 转换）；
>     父按 **out 布局**读 child 输出后的缓冲，`realChannelCount = outFormat`。
>   - **协商委托**：有 child 时父先问 child 的 IsInputFormatSupported，child 成功即用 child 判定；仅 child 失败/缺失回落父自身检查——child 格式对接由 child 负责。
>   - **仅有的上混特例 = mono→stereo**：补「无 APO 时 Windows 音频系统自动上混」的默认行为；其他上混/下降混父均不执行。
>   - **VxAPO 等价实现**：协商期锁死「收到的输入 == 输出通道数」（父不转换的自身立场，同 EAPO）；**不引入 realChannelCount 维度切换机制**（EAPO 该分支在协商后为冗余路径）——实现简化，非更严谨；child 通道布局纯效果由契约声明（两方均无运行时布局检测）。
>
> ①②③ 收敛结论与 EAPO 通道机制详析见**主规范「十八、EAPO 对齐度与差异化」**（18.2 APO 链格式授予语义 + D2/D3 等价立场）；此三点在 P0-6 Spec-Drafting 定稿时仅需**确认**（对照该章节）不再重开设计。

### P0-7  CLI 端到端验证（install/uninstall/config set/show/list/status + 回滚）
- 状态：Spec-Finalized（v8.8）
- 优先级：P0 ｜ 关联 Phase：Phase 10（P0 收尾）
- 目标：CLI 依赖 vxapo-driver（as library），提供 **P0 达标口径的端到端验证**——设备 install/uninstall、config set/show、list/status、回滚 snapshot；验证驱动可安装、可加载、可按设备读 config（P0 链路收尾）
- 影响模块：`vxapo-cli`（crate，规范外）、`vxapo-driver` 的 `install/selector/operation.rs`（install_endpoint/uninstall_endpoint 复用 + CLI 层 API）、`install/device/info.rs`（enumerate_devices 复用）、`install/device/slots.rs`（槽位失守检测只读 API + child_apo_key_exists）
- 规范落点：**《CLI 引用规范.md》v8.8（2026-08-04 定稿）**——现状/可复用 API（源码实读）/ 边界/ 修改路线 Phase A-D / `<device>`·`<file>` 参数 / GUID 友好名称 4.5 / 快照=变更对比 Phase C / 行为流 5.1-5.5 / 命令状态流 5.4 / 约束；+ `install 5.3/5.4/5.5.2`（复用 API 来源 + v8.9：detect_install_mode/detect_mode_for_device·guid 自动探测 + 卸载语义 + 快照恢复）+ `install 5.4`（v8.9 VxAPO CLSID 成对判定）+ `object 7.1.8/7.1.9`（v8.9 config 路径 = `C:\ProgramData\VxAPO\{GUID}\config.txt` 系统级）+ `intent 五/七/十一`（CLI 边界/槽位失守检测/config 语法）
- 依赖：P0-4、P0-5、P0-6（均 Done ✅）——P0-6 child 委托链端到端联调 + P0-4 听感验证均靠 CLI install/config set 覆盖
- DoD：☑ 规范定稿（v8.8《CLI 引用规范.md》）☐ 实现 ☐ 测试（端到端：install → config set → driver 读回 → watcher 热重载生效 + 回滚验证 + child 委托链联调）

> **范围与优先级说明（v8.0 用户决策）**：
> - **本条目执行了 P1-2（CLI per-device 配置管理）的部分核心能力**——install/uninstall/config set/show；
>   **P1-2 条目本身不动**（保持 Backlog），其剩余能力（preset 管理 / inherit / 完整 CLI 边界声明）留待 P1-2 正式推进。
> - **FxSound（P1-1）后做**：CLI 端到端验证优先（P0 收尾），P1-1 保持原位，P0 链路完成后再填效果。
>
> **架构（用户确认）**：CLI 依赖 vxapo-driver crate（作为库）——经 install 层 API 操作
> （install_endpoint / uninstall_endpoint / enumerate_devices），不触碰 pipeline/RT；
> 符合 intent.md 三层分离（CLI 是开发者工具，经 driver 的 install 层操作）。
>
> **能力集**：
> - ① install `-d <device>`（调 `install_endpoint`，写 FxProperties 绑定）
> - ② uninstall（调 `uninstall_endpoint` + driver selector 层事务回滚卸载）
> - ③ config set（写 `C:\ProgramData\VxAPO\{GUID}\config.txt`，v8.9 系统级——audiodg/SYSTEM 与用户进程共用）
> - ④ config show（读回一致性验证——「写后读回」）
> - ⑤ list / status（保留现有诊断：枚举设备、查询注册表、友好名称、**查看设备哪些槽位被接管**）
> - ⑥ **回滚 snapshot**：对 driver 改动前先 snapshot（注册表/配置状态），失败可恢复
>
> **验证边界**：config show 验证「文件已写入且可读回」；真正 DSP 热重载生效由 P0-4 的手动听感验证（无爆音）覆盖——CLI 无音频监听，不做效果断言。

### 实现完成报告（2026-08-04，执行端）
- DoD：☑ 实现 ☐ 测试（cargo check 0 error；端到端 install/uninstall/config 需真实设备 + 管理员 + audiodg，留手动）
- 自查结果：仅改 vxapo-cli（crate 规范外，依赖 vxapo-driver as library）；不触碰 pipeline/RT；不自行写注册表（install/uninstall 经 driver Transaction）；config 写归 CLI
- 实现文件：Cargo.toml（vxapo-driver + windows 依赖）、src/commands.rs（resolve_device/require_admin/list_devices/install/uninstall/config_set/config_show/snapshot_device/diff/restore + 槽位失守检测按 childapo 键存在判定）、src/knowledge.rs（KNOWN_APO_CLSIDS 4.5）、src/main.rs（子命令分派 + 帮助）
- 遗留：端到端验证索引①-⑥（CLI 规范 5.5）需真实 Windows 设备 + 管理员运行，留手动验证（对应 P0-4 听感 + P0-6 child 委托链联调）

---

## P1 — 核心功能（CLI 可操控 + 效果扩展）

> 达标口径：命令行完成设备配置管理、导入导出、预设；4 个 FxSound 效果可听。

### P1-1  FxSound 效果接入（Wide → Aural → Maximizer → Lex）
- 状态：已实现（v9.1 Aural/Maximizer/Reverb + v9.2 Wide，4/4；v9.3 起算法逐个升级，
  Reverb → Dattorro、Maximizer → 独立实现（v9.8）已完成；听感验证留手动）
- 优先级：P1 ｜ 关联 Phase：Phase 11
- 目标：四个 FxSound 效果作为原生 Filter 接入 config.txt 解析链路，按复杂度递增
- 影响模块：`pipeline/dsp/fxsound/{aural,reverb,maximizer,wide}.rs`、`pipeline/dsp/factory.rs`
- 规范落点：`pipeline 4.22`（fxsound 子模块）+ `config 6.16`（四个命令语法）+ `pipeline 4.10`（工厂注册）
- 依赖：P0-2（解析链路先通）
- DoD：☑ 规范定稿 ☑ 实现（4/4） ☑ 算法升级起步（v9.3 Reverb → Dattorro） ☐ 听感验证（留手动）

### P1-3  效果器算法升级（更优算法替换）
- 状态：进行中（v9.3 Reverb → Dattorro、v9.8 Maximizer → 独立 lookahead 限幅
  已完成；剩余 Wide → Aural）
- 优先级：P1 ｜ 关联 Phase：Phase 11
- 目标：用户反馈 FxSound 移植效果听感一般；在命令名与参数面不变的前提下逐个替换为
  文献级更优算法——Reverb（Dattorro，1997 论文）已完成；Maximizer（lookahead
  attack/release + 多峰事件队列，参考 FFmpeg alimiter）v9.8 已完成；Wide
  （M/S + 双频段 + 全通去相关，参考 DAFx24 StereoWidener）、Aural（tanh 软饱和 +
  电平跟随，参考 Jatin Chowdhury / FAUST）待替换
- 影响模块：`pipeline/dsp/fxsound/{reverb,maximizer,wide,aural}.rs`
  （替换完成的文件同步移除对应 FxSound AGPL 版权头）
- 规范落点：`pipeline 4.22` + `config 6.16`（每步随版本号递增同步）
- DoD：☐ 规范定稿 ☐ 实现 ☐ 测试

### P1-4  双实例与默认效果冲突修复（v9.4）
- 状态：已实现（v9.4：PostMix 直通 + MSFX 运行期自愈 + watcher 防自旋）
- 优先级：P1 ｜ 关联 Phase：Phase 11
- 目标：修复「切换音频设备后带 GraphicEQ 慢放/断断续续」与「声音设置页卡顿」——
  根因①渲染设备 SFX+EFX 双 VxAPO 实例重复应用 config；根因②Windows 重新枚举后
  从 `MSFX\N` 模板灌回微软 CAPX 与 VxAPO 叠加；根因③watcher 事件风暴导致
  audiodg CPU 持续高位
- 影响模块：`object/apo/{init,process}.rs`、`config/watcher.rs`、`install/device/sysfx.rs`
- 规范落点：`object 7.1.8/7.1.9/7.1.18` + `install 5.5.2` + `Equalizer 行为文档 2.10`
- DoD：☑ 规范定稿 ☑ 实现 ☐ 真机验证（设备切换 + 设置页）

### P1-5  GraphicEQ 性能与热重载补强（v9.5）
- 状态：已实现（v9.5 分块 FFT → v9.6 直接 FIR 修复切换嗡声 → v9.7 1024 点 +
  AVX2/FMA 向量化：低频分辨率与 CPU 兼得；热重载补载 + 空命令 passthrough；
  启动静音已移除——Windows 自带行为）
- 优先级：P1 ｜ 关联 Phase：Phase 11
- 目标：多路音频流时 audiodg 不再吃满（声音设置页卡顿）；配置热重载在所有
  时序下可靠；`GraphicEQ:` 空参数能显式移除 EQ；设备切换无嗡声/无首段断续
- 影响模块：`pipeline/dsp/graphic_eq.rs`、`object/apo/config.rs`、`config/watcher.rs`、
  `config/commands/graphic.rs`、`config/parser.rs`
- 规范落点：`pipeline 4.18` + `config 6.10` + `object 7.1.9/7.1.18` + `Equalizer 行为文档 2.10`
- DoD：☑ 规范定稿 ☑ 实现 ☑ 真机验证（设置页卡顿 / 空命令热重载 / 切换嗡声 / 首段断续）

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

### P0-4  配置热重载全链路 — Done @ v8.0
- 规范落点：`config 6.1/6.2`（filter_spec + 目录级事件驱动 + 白名单三段式）、`object 7.1.8/7.1.9/7.1.10/7.1.18`（watcher 生命周期 + start/stop + spec 短路 + 保留旧链）、`pipeline 4.19/4.20`（Convolution strict + VST 特判）、`intent.md`
- 实现：`src/config/parser.rs`（v7.9-v7.12 + 白名单三段式）、`src/config/watcher.rs`（事件驱动 + unsafe impl Send）、`src/object/apo.rs`（watcher 线程接线 Arc\<Mutex\<WatcherState\>\> + start/stop_watcher + hot_reload_impl）——commit 77bc68e / 7b20619
- 验收：436 passed / 0 failed + check 0 error 0 warning；听感验证④留 P0-7 CLI 落地后手动并联调
- 反馈闭环：`feedback.md #P0-4-0/1/2/3/4`（v7.7 时序 + v7.9 目录级 + v7.10 线程模型 + v7.11 严格化 + v7.12 白名单）——全部已修订
