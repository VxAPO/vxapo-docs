# Changelog

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