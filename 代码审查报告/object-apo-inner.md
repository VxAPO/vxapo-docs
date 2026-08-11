# 代码审查报告：object/apo/inner.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\object\apo\inner.rs`
- 审查模块：`object/apo/inner`（双链过渡状态 + 启动淡入）
- 参照规范：`object 模块规范.md` 7.1、`模块引用规范（无详细模块版）.md` 第十一节
- 总体评价：⚠️ 需修改（轻微）

---

## 审查摘要

- 关键风险：
  1. `apply_startup_fade` 文档写“默认 500ms”，常量实际是 `STARTUP_FADE_HOLD_MS = 100`（86–87 行 vs 90–91 行）——文档与实现不一致。
  2. `build_dsp_context` 与 `object/apo/process.rs` 241–253 行手工构造 `DspContext` 重复——同一数据构造逻辑两处漂移。
  3. `apply_startup_fade` 是 RT 路径被调函数（纯算术、零分配 ✓），但缺失“RT 安全”注释，容易被后续维护误加 I/O。

优点：字段全部有语义注释、`retired_chain`/`pending_reload` 等过渡状态与规范 7.1.3 对齐、`apply_startup_fade` 有 `out_ch==0`/`out.len()/out_ch` 防御。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 86–87 vs 90–91 | “默认 500ms”与常量 100ms 不符 | 统一为 100ms 或改常量 |
| 94–98 | `STARTUP_FADE_RAMP_MS = 0` 时 `ramp = max(1)` 且 `hold == total`，淡入恒为静音后瞬跳——机制实际停用 | 补注释说明 v9.7 停用现状，避免误读为启用态 |
| 113–114 | `apply_startup_fade` 无 RT 安全标记 | 加 `// RT：纯算术、无分配、无锁、无 I/O` |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 117–127 | `hold = total * 100 / 100`，`ramp = max(total-hold, 1)`——若未来调整比例需小心整数除法；当前 0 淡入下逻辑可读性差 | 用命名中间量并加注释 |
| 122 | `frames.min(out.len() / out_ch)` 防御 ✓ | 无 |
| 142–159 | `build_dsp_context` 与 process.rs 重复构造 `DspContext`，若 `ProcessingStage`/`DeviceType` 语义变更会漏改一处 | 统一调用 `inner::build_dsp_context` |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| — | 无 unsafe/FFI；`apply_startup_fade` 为 RT 纯计算 | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 4–8 | 依赖 `pipeline/{chain,context,dsp/filter,dsp/transition}`、`sys/audio_defs` —— object 层允许；但父总表没有 `object/apo/inner.rs` 行 | 规范侧为 `object/apo/inner` 补登记（或并入 object/apo 聚合行） |
| 28–41 | `last_calls: [(u64, u32, u32, f32, u32, f32); 8]` 魔法 8 仅字段注释说明 | 提 `const RT_CALL_HISTORY: usize = 8;` |

### 依赖违规检查

- 本文件所在模块：`object/apo/inner.rs`
- 规范允许依赖：object 层总则“依赖所有模块”；父表无 inner 行
- 实际依赖：`pipeline/chain`、`pipeline/context`、`pipeline/dsp/filter`、`pipeline/dsp/transition`、`sys/audio_defs`
- 违规项：无实质违规；⚠️ 规范缺 inner.rs 行，建议补录。

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 117–135 | 逐采样双层循环手写淡入 | 可接受（RT 手写循环更可控）；可抽 `fn fade_factor(elapsed, hold, ramp) -> f32` 纯函数 |
| 142–159 | 重复的 `DspContext` 构造 | 收敛为唯一 `build_dsp_context` 调用点 |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无 RT 分配/unsafe/依赖红线违规 |
| 🟡 建议级 | 2 | 文档/常量不一致；DspContext 构造重复 |
| 🟢 优化级 | 2 | RT 安全注释；魔法 8 常量化 |
