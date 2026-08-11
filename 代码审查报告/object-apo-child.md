# 代码审查报告：object/apo/child.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\object\apo\child.rs`
- 审查模块：`object/apo/child`（子 APO COM 生命周期与委托）
- 参照规范：`object 模块规范.md` 7.2、主规范 18.1（A1–A5）、`模块引用规范（无详细模块版）.md` 第十一节
- 总体评价：⚠️ 需修改（轻微）

---

## 审查摘要

- 关键风险：
  1. `apo_process_inner` 中 `child.calc_input_frames(0)` 的返回值被丢弃（process.rs 574 行）——委托帧数计算形同虚设，属遗留死调用。
  2. `unsafe impl Send/Sync for ChildApo` 的论证基于“引擎保证线程亲和”，但子 APO 实际可能被父的 RT/控制两线程同时触碰（`apo_process` vs `lock_for_process`），依赖约定需更强注释或互斥保障。
  3. 303–308 行两个 `from_raw_parts` 无内联 SAFETY 注释（函数级 Safety 已覆盖，属可读性收尾）。

优点：引用计数生命周期推演清晰（ref 1→4→3→0）、`resolve_supported` 用 `ManuallyDrop` 移交所有权正确、失败降级语义（Note 57）与 EAPO A2 对齐、null 防御齐全、依赖与规范行完全一致。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 1 | 文件头仍写旧路径 `host/instance/apo_child.rs` | 改为 `object/apo/child.rs` |
| 303–308 | 两处 `from_raw_parts` 无内联 Safety 注释 | 补“描述符数组由引擎分配，num 参数校验非零” |
| 329–336 | 测试删除原因说明详实 ✓ | 无 |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 255–257（及 process.rs 574） | `calc_input_frames(0)` 结果被丢弃 | 用返回值参与父的帧计算，或删除该调用并注释“child 帧数由父统一计算” |
| 110–113 | `get_latency` 失败返回 0（保守）✓ | 无 |
| 118–123 | `reset` 失败仍返回 HRESULT 供上层判定 ✓ | 无 |
| 99–103 | `is_valid` 仅检查非空，不验证引用有效性（cast 成功即有效，属防御冗余） | 可保留；注释已说明 |
| 292–312 | `num_input==0` 直接 E_POINTER，与 APOERR_NUM_CONNECTIONS_INVALID 语义略异 | 可保持（子 APO 层防御），建议注释 |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 59–63 | `unsafe impl Send/Sync` 有 SAFETY 注释，但“线程亲和”论证偏弱（见摘要 2） | 补充：`child_apo` 以 `Mutex<Option<ChildApo>>` 保护，RT/控制路径均先取锁 |
| 74–93 | `create` 的 SAFETY 注释覆盖 COM 初始化与 CLSID 有效性 ✓ | 无 |
| 130–145、152–166、177–204、208–236、272–280、292–312 | 各 unsafe 函数均有函数级 Safety 文档 ✓ | 无 |
| 230–231 | `ManuallyDrop` 所有权移交逻辑正确（调用方 Release）✓ | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 26–35 | 依赖 `sys/com/apo_interfaces`、`sys/com/apo_types`、`sys/com/prelude` —— 与规范 object/child.rs 行**完全一致** ✓ | 无 |
| 329–336 | 无 `cfg(test)` 测试（真实 COM 无法单测）——已在文件头说明原因 | 可补充 `#[cfg(test)]` 内 `ManuallyDrop` 移交的纯逻辑测试（不触 COM） |

### 依赖违规检查

- 本文件所在模块：`object/apo/child.rs`
- 规范允许依赖：`sys/com/prelude`、`sys/com/apo_interfaces`、`sys/com/apo_types`
- 规范禁止依赖：`object/apo`（禁止循环）
- 实际依赖：`sys/com/apo_interfaces`、`sys/com/apo_types`、`sys/com/prelude`
- 违规项：无 ✅

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 118–123、318–322 | `map(\|_\| S_OK).unwrap_or_else(\|e\| e.into())` 重复 | 抽 `fn hr<T>(r: Result<T>) -> HRESULT` 统一转换 |
| 303–308 | 两次 `from_raw_parts` 类型转换重复 | 抽 `fn descriptor_slices(inputs, outputs, ni, no) -> Option<(&[...], &[...])>` |
| 74–93 | 三步创建 + drop 注释清晰 ✓ | 无 |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无 RT 分配/unsafe unsound/依赖红线 |
| 🟡 建议级 | 2 | `calc_input_frames(0)` 死调用；Send/Sync 论证需补互斥说明 |
| 🟢 优化级 | 3 | 文件头路径修正；内联 Safety 注释；HRESULT 转换 helper |
