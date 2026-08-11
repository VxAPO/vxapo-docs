# 代码审查报告：object/factory.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\object\factory.rs`
- 审查模块：`object/factory`（IClassFactory + LOCK_COUNT）
- 参照规范：`object 模块规范.md` 7.5（Note 2/3/4/5）、`模块引用规范（无详细模块版）.md` 第十一节
- 总体评价：🚫 严重违规（CreateInstance 空指针先解引用 + lock_decrement 下溢）

---

## 审查摘要

- 关键风险：
  1. **空指针先解引用**：80 行 `unsafe { *ppvobject = null }` 在 83 行 `ppvobject.is_null()` 校验**之前**执行——调用方传 null 时直接 UB（应“先校验后写入”）。
  2. **LOCK_COUNT 下溢**：`lock_decrement`（34–37 行）用 `fetch_sub` + `saturating_sub`，`prev == 0` 时 `fetch_sub` 回绕成 `u32::MAX`，`saturating_sub(1)` 后计数停在 4294967294，`DllCanUnloadNow` 永久 `S_FALSE`（ref_count.rs 已用 CAS 保护，此处未同步）。
  3. 87–111 行注释仍称“完整 NonDelegating 委托需手写 vtable……本阶段先锁定行为”，而 aggregate.rs 已实现完整委托——注释严重过期。

优点：聚合路径（NApo）接线正确、QI 后释放工厂临时引用的计数推演正确、mock outer 的端到端测试质量高。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 87–111 | 过期注释（见摘要 3） | 重写为“聚合由 aggregate.rs 的 NApo 完整实现” |
| 19–20 | `#[allow(unused_imports)] use ...ApoObject` 仅测试使用 | 移入 `#[cfg(test)]` |
| 128 | `type QIFn2` 命名可读性 | 顶部统一定义 `type QiFn` |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 80–85 | 空指针先解引用（见摘要 1） | 先 `if riid.is_null() \|\| ppvobject.is_null() { return E_INVALIDARG; }` 再写 `*ppvobject` |
| 34–37 | `lock_decrement` 下溢（见摘要 2） | 复用 ref_count.rs 的 CAS 零值保护模式 |
| 75 | debug 探针在参数校验前解引用 `*riid` | 移到校验之后 |
| 99–105 | 聚合时 `riid != IUnknown::IID` 返回 E_NOINTERFACE——按 COM 规范正确；注释可补“引擎实际只请求 IUnknown” | 无 |
| 141 | `release_aggregate(na)` 释放工厂临时引用——若 `qi2` 返回成功但调用方随后不使用（COM 契约保证会使用），计数仍平衡 ✓ | 无 |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 80 | `*ppvobject` 在未校验时解引用（R3/UB 边界） | 修复顺序即消除 |
| 127–131 | `transmute` QI 函数指针有 SAFETY 注释（130 行）✓ | 无 |
| 112–119 | `create_aggregate` 的 SAFETY 注释 ✓ | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 20–22 | 依赖 `object/apo`（仅测试用 import 声明）、`object/vx_reg_props`、`sys/com/prelude` —— 规范 factory 行：`sys/com/prelude`、`sys/com/apo_interfaces`、`object/apo.rs`、`object/vx_reg_props.rs`、`object/ref_count.rs` | 运行期未引用 apo_interfaces（可修订规范）；无越界 |
| 47–50 | `lock_reset_for_test` cfg(test) ✓ | 无 |

### 依赖违规检查

- 本文件所在模块：`object/factory.rs`
- 规范允许依赖：`sys/com/prelude`、`sys/com/apo_interfaces`、`object/apo.rs`、`object/vx_reg_props.rs`、`object/ref_count.rs`
- 实际依赖：`object/apo::aggregate`（运行期）、`object/vx_reg_props`、`sys/com/prelude`
- 违规项：⚠️ `object/apo/aggregate` 属 object/apo 子模块（规范 factory 行写 `object/apo.rs` 入口）→ 建议规范明确“factory 可依赖 aggregate”；无实质越界。

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 114–119 | `punkouter.as_ref().map(...).unwrap_or(null)` | 可保留（组合链已清晰） |
| 127–131 | 局部 `QIFn2` + transmute | 复用 aggregate.rs 导出的 `QiFn` 类型（若 pub(crate)） |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 2 | ① 80 行空指针先解引用（UB）；② `lock_decrement` 下溢导致 DllCanUnloadNow 永久 S_FALSE |
| 🟡 建议级 | 2 | 过期注释重写；debug 探针移到校验后 |
| 🟢 优化级 | 2 | unused import 收口；QiFn 类型复用 |
