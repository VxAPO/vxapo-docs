# 代码审查报告：install/audiodg.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\install\audiodg.rs`
- 审查模块：`install/audiodg`（DisableProtectedAudioDG 检查/修复、AudioSrv 停止/重启）
- 参照规范：`install 模块规范.md` 5.6、`模块引用规范（无详细模块版）.md` 第十一节
- 总体评价：⚠️ 需修改

---

## 审查摘要

- 关键风险：
  1. `is_disabled()` 把“读注册表失败/权限错误”与“值不存在”一律吞成 `Ok(false)`，错误被掩盖（27–38 行）。
  2. `ensure_can_load()` 的进程内 `AtomicBool` 缓存一旦置 true 终身有效，外部把值改回/删除后本进程仍认为已禁用（75–89 行）。
  3. 两个服务函数各自重复定义 `ScmGuard`/`SvcGuard`，且使用 `SC_MANAGER_ALL_ACCESS`/`SERVICE_ALL_ACCESS` 最高权限，超出最小权限原则。

优点：句柄全部 RAII 关闭、非 RT 控制线程、注释详实、30s 轮询超时合理。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 97–149、166–241 | `stop_audio_service` 与 `restart_audio_service` 前半段（打开 SCM/服务、等待 STOPPED）几乎完全重复 | 提取 `fn stop_service_handle(svc) -> Result<()>` 公共逻辑 |
| 110–116、186–193、122–128、200–207 | 两处同名 `ScmGuard`/`SvcGuard` 重复定义 | 提为模块级 `struct ScmHandleGuard` / `SvcHandleGuard` |
| 248 | `is_disabled_at` 仅测试使用却放在生产代码并 `#[allow(dead_code)]` | 移入 `#[cfg(test)]` 或标注 `#[cfg(any(test, ...))]` |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 27–38 | `RegKey::open`/`read_dword_value` 的所有 `Err` 都按“未禁用”处理，权限/IO 错误被掩盖，`disable()` 可能因此反复尝试写 HKLM | 区分“值缺失”（`Ok(false)`）与“查询失败”（`Err` 上抛） |
| 75–89 | 静态缓存无失效机制；若安装/卸载流程在同一进程内改回注册表，`ensure_can_load` 仍返回 `Ok` | 至少提供 `pub(crate) fn invalidate_cache()` 供 `restore()`/卸载调用，或注释明确“仅限 audiodg 进程生命周期” |
| 134–145 | `ControlService` 若收到 `ERROR_SERVICE_ALREADY_RUNNING/STOP_PENDING` 等竞态会直接 `Err` 失败 | 对 `SERVICE_STOP_PENDING` 也进入轮询等待，而非直接报错 |
| 216–232 | 同上：服务处于 `STOP_PENDING` 时跳过停止直接 `StartServiceW` | 统一状态分支处理 |
| 236–237 | 启动后不等待 `SERVICE_RUNNING`，注释称“失败不阻塞”，但 `StartServiceW` 失败仍返回 `Err`（与注释语义不一致） | 明确 best-effort：失败仅日志降级，或等待运行态后再返回 |
| 137、222 | 30 秒超时后直接报错，未尝试停止时先发 `SERVICE_CONTROL_PRESHUTDOWN` 或至少提示残余状态 | 可选：超时错误附带当前 `dwCurrentState` |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 106–108、119 | `OpenSCManagerW` 的 SAFETY 注释写的是“非 RT 线程”，未说明句柄/字符串有效性前提；`OpenServiceW` 处无独立 SAFETY 注释 | 按块补“PCWSTR 有效、句柄非空、失败不持有”前提 |
| 107、180 | `SC_MANAGER_ALL_ACCESS` / `SERVICE_ALL_ACCESS` 权限过大 | 用 `SC_MANAGER_CONNECT` + `SERVICE_QUERY_STATUS|SERVICE_STOP|SERVICE_START` 最小权限 |
| 110–116、122–128 | 两个 RAII guard 的 `Drop` 内 `CloseServiceHandle` 无 SAFETY 注释（113、125 行） | 补“句柄由 Open 创建且未被转移”前提 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 7–8 | 实际依赖 `sys/registry` + `utils/vx_error` + `windows`（Services/Registry）；规范 5.6 与父总表仅列 `sys/registry` | 规范侧修订 `install/audiodg.rs` 行为 `sys/registry + utils/vx_error + windows Win32_System_Services`；实现本身未触 `pipeline/config/object`，无实质越界 |
| 97–241 | 服务控制逻辑直接散在函数内，未来 CLI/APP 复用需复制 | 抽 `sys/registry` 之外的 `utils/service.rs` 或本模块私有辅助 |
| 12–16 | 常量集中且有注释 ✓ | 无 |
| 264–321 | `cfg(test)` 隔离 ✓，测试用 HKCU 注入路径设计良好 | 无 |

### 依赖违规检查

- 本文件所在模块：`install/audiodg.rs`
- 规范允许依赖：`sys/registry`（详细表）；模块级总则允许 `sys/`、`utils/`、`object/vx_reg_props.rs`
- 规范禁止依赖：`pipeline/`、`config/`、`object/`（除 vx_reg_props）
- 实际依赖：`sys/registry`、`utils/vx_error`、`windows`（Registry/Services）
- 违规项：⚠️ `utils/vx_error` 未在 5.6 行列出（模块级总则允许）→ 规范表述不一致，建议修订总表；无 pipeline/config/object 违规。

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 26–39 | 两层 `match` 吞错 | `RegKey::open(...).and_then(|k| k.read_dword_value(...))`，用 `Option`/`is_err` 区分缺失与失败 |
| 97–149 | 完整服务停等逻辑手写 | `fn stop_service(svc: SC_HANDLE) -> Result<()>` + 统一轮询 helper |
| 110–128 | 局部 struct guard | 模块级 `ScmHandle(SC_HANDLE)` 实现 `Drop`，两函数共用 |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无 RT/unsafe/依赖红线违规（本模块为控制路径） |
| 🟡 建议级 | 4 | 错误吞没；进程内缓存无失效；服务竞态状态未处理；权限过大 |
| 🟢 优化级 | 4 | 公共停服逻辑抽取；guard 去重；`is_disabled_at` 移入测试；错误区分缺失/失败 |
