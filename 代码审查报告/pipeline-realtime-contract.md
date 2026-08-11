# 代码审查报告：pipeline/realtime/contract.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\pipeline\realtime\contract.rs`
- 审查模块：`pipeline/realtime/contract`（RT-safety 契约 + 见证 + 断言宏）
- 参照规范：`pipeline 模块规范.md` 4.7（Note 12/58）、主规范 O1
- 总体评价：⚠️ 需修改（轻微）

---

## 审查摘要

- 关键风险：
  1. **嵌套 RtGuard 语义缺陷**：`RtGuard` 无深度计数，内层 Drop 即把全局 `RT_ACTIVE` 清 false，外层仍在 RT 上下文（117–135 行）——嵌套时保护失效。
  2. `enter/exit_rt_context` 用 `SeqCst` 原子写，debug 下每帧进/出 RT 有额外开销（78–92 行）。
  3. `rt_index`/`rt_index_mut` release 模式直接 `get_unchecked`，越界即 UB（debug_assert 只在 debug 生效）——这是设计选择，但调用点必须证明边界。

优点：`RtSafe`/`RtCopy` 不安全 trait 带完整 Safety 契约、`RealtimeContext` 零尺寸私有构造（O1 落地）、宏在 release 零开销、测试覆盖嵌套/断言/越界。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 113–135 | `RtGuard` 无“不支持嵌套”或“深度计数”说明 | 补注释：当前实现嵌套时内层 Drop 会提前退出 RT 上下文 |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 124–135 | 嵌套 Drop 语义错误（见摘要 1） | 用 `Cell<usize>`/原子深度计数，或 `debug_assert!(!RT_ACTIVE.load())` 拒绝嵌套 |
| 81、90、101 | `SeqCst` 过强 | `Relaxed` 即可（仅调试标志，无跨线程数据传递） |
| 200–224 | release `get_unchecked` 依赖调用方证明边界 | 文档补充“调用方必须保证 index < len，否则 release UB” |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 22、29 | `unsafe trait RtSafe/RtCopy` 有 Safety 文档 ✓ | 无 |
| 200–224 | `rt_index*` 有 Safety 注释 ✓ | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 3–4 | 依赖仅 `core`/std atomic —— 与规范行 `core` **一致** ✓ | 无 |
| 144–188 | 宏 `#[macro_export]` 导出到 crate 根，调用路径 `$crate::pipeline::realtime::contract::...` 正确 ✓ | 无 |

### 依赖违规检查

- 本文件所在模块：`pipeline/realtime/contract.rs`
- 规范允许依赖：`core`
- 规范禁止依赖：其他
- 实际依赖：`core`（+ debug 下 std::sync::atomic）
- 违规项：无 ✅

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 117–135 | 手动 enter/exit | `Cell<usize>` 深度计数（debug-only） |
| 145–188 | 宏定义清晰 ✓ | 无 |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无（debug-only 保护缺陷，不影响 release） |
| 🟡 建议级 | 2 | 嵌套 RtGuard 语义；release 越界 UB 文档 |
| 🟢 优化级 | 2 | SeqCst→Relaxed；深度计数 |
