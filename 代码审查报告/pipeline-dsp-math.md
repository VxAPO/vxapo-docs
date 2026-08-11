# 代码审查报告：pipeline/dsp/math.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\pipeline\dsp\math.rs`
- 审查模块：`pipeline/dsp/math`（dB 换算/极点稳定性/RT 线程 FTZ-DAZ）
- 参照规范：`pipeline 模块规范.md` 4.16 相关、主规范 O1
- 总体评价：✅ 通过

---

## 审查摘要

- 数值策略完善：NaN/±inf 归一、f64 中间计算、极点半径判据带余量、FTZ/DAZ 用 thread_local 幂等 + 内联汇编带 SAFETY 注释。RT 入口每帧调用零开销。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 36–70 | 常量集中且带来源注释 ✓ | 无 |
| 85–88 | `MAX_FRAME_COUNT = 8192` 与 `object/apo/process.rs::MAX_APO_LATENCY_SAMPLES = 8192` 语义不同却同值 | 注释区分或统一常量来源 |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 73–77 | `clamp_gain_db` 先判 NaN 再判 ±inf ✓ | 无 |
| 99–114 | `is_stable_biquad` 用 f64 精度判定 ✓ | 无 |
| 149–178 | `init_audio_thread` 首次调用执行两条 asm，幂等 ✓ | 无 |
| 185–195 | `warn_rate_limited` 控制路径锁 + HashMap 分配 ✓（已注释非 RT） | 无 |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 164–177 | 内联汇编有 SAFETY 注释 + `nostack, preserves_flags` ✓ | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| — | 内部依赖无（std/once_cell）—— 符合 dsp/*.rs 约束（允许 utils；本文件零内部依赖更严） | 无 |
| 34 | `docs/dsp-math-optimization-plan.md` 引用是否存在待确认 | 确认文档路径 |

### 依赖违规检查

- 本文件所在模块：`pipeline/dsp/math.rs`
- 规范允许依赖：`dsp/filter`、`dsp/model`、`dsp/biquad`、`utils`
- 规范禁止依赖：config/install/object
- 实际依赖：无内部模块依赖
- 违规项：无 ✅

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 145–178 | `thread_local!` + Cell 幂等 ✓ | 无 |
| 73–97 | 三个数值转换函数简洁 ✓ | 无 |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无 |
| 🟡 建议级 | 0 | 无 |
| 🟢 优化级 | 2 | 8192 常量语义注释；文档引用确认 |
