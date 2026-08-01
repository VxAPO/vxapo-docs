# Changelog

## v6.9 — 2026-08-01

变更类型：`外部借鉴`（EqualizerAPO FilterEngine 过渡/重载机制深度分析）

- **R1 退役链延迟析构**：过渡完成帧 RT 线程仅 `retired_chain = outgoing_chain.take()`（零析构），控制线程锁内统一 drop——重型滤波器析构绝不留在 RT 线程
- **R2 阻塞式重载**：hot_reload 检测到过渡在途/加载中即返回不构建（EAPO `loadSemaphore` 对齐），过渡完成由 APOProcess 触发重载；`reloading` 标志防覆盖
- **R3 空链快路径**：`Chain::is_empty()` 时 `process_audio` 直接复制去交织结果（近似 memcpy）跳过链遍历
- **R4 过渡周期 10ms**：`default_smoothing_length` 由 `sample_rate/20`（50ms）→ `sample_rate/100`（10ms）——双处理窗口缩短 5 倍

> 对应 commit：`7aa6f16`

## v6.8 — 2026-08-01

变更类型：`外部借鉴`（EqualizerAPO Device 层）

- **E3.1 默认设备判定边界**：driver 层 `enumerate_devices()` 仅返回合法安装容器，不判定默认设备；默认/有效设备由用户层经 COM `GetDefaultAudioEndpoint` 取得后对应
- **E3.2 设备物理状态谓词**：`DeviceInfo::is_disabled()` / `is_unplugged()`（由已有 `EndpointState` 推导，零新增 I/O）
- **E3.3 autoAdjust 独立字段**：`InstallConfig::auto_adjust`（默认 false），Step 4 写注册表读取该字段而非硬编码
- **E3.4 安装自检**：`install_endpoint(..., verify)`——commit 后 `CoCreateInstance` 验证 DLL 可实例化；失败不自动回滚
- 修复 install 5.5.2 多余代码围栏（用户修复，`b162890`）

> 对应 commit：`bac344e`（+ 用户围栏修复 `b162890`）

## v6.7 — 2026-07-31

变更类型：`外部借鉴`（EqualizerAPO `IFilter::getInPlace`）

- **E1 就地处理声明**：`Filter::is_in_place()`（默认 true）+ `Chain::is_fully_in_place()`——全链就地时 `temp_buffers` 即最终输出，零拷贝快路径
- 与 v6.6 的 O1（RealtimeContext）/ 去交织架构完全兼容

> 对应 commit：`415ef53`

> **范围说明**：本 changelog 自 v6.7 开始记录（v6.0-v6.6 不补录）。维护规则见 `.clinerules/changelog-rule.md`。