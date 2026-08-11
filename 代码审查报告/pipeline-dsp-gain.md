# 代码审查报告：pipeline/dsp/gain.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\pipeline\dsp\gain.rs`
- 审查模块：`pipeline/dsp/gain`（Preamp 增益 + 比例平滑）
- 参照规范：`pipeline 模块规范.md` 4.15（Note 15）、主规范 RT 约束
- 总体评价：⚠️ 需修改（轻微）

---

## 审查摘要

- 平滑算法正确：比例平滑等效对数域、步进 clamp ±5%、snap 阈值、极深衰减恢复起点保护；RT 路径零分配、`get_mut` 防越界。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 1 | 文件头旧路径 `dsp/filters/gain.rs` | 改为 `pipeline/dsp/gain.rs` |
| 84–113 | `advance` 注释详尽 ✓ | 无 |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 99–101 | `current_gain.abs() < snap` 时置 `current = target * 1e-4`——若 target 为 0 则保持 0，随后 `ideal = 0/0 = NaN` 分支不会执行（current==target 提前返回） | 可加 `debug_assert!` 防未来改动破坏 |
| 92–97 | snap 判定 `threshold * max(1.0)` 对 target 极小值行为正确 ✓ | 无 |
| 131–135 | `samples.get_mut(ch)` 越界静默跳过——通道索引来自 Chain 校验，可加 debug_assert | 无 |
| 103–110 | `remaining.max(GAIN_SMOOTH_RERATE)` 保证 ≥32 步 ✓ | 无 |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| — | 无 unsafe/FFI；RT 纯算术 | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 13–18 | 依赖 `dsp/filter`、`dsp/math` —— 符合 dsp/\*.rs 约束 ✓ | 无 |
| 144–152 | `db_to_linear`/`linear_to_db` 薄转发保留公开 API ✓ | 无 |

### 依赖违规检查

- 本文件所在模块：`pipeline/dsp/gain.rs`
- 规范允许依赖：`dsp/filter`、`dsp/model`、`dsp/biquad`、`utils`
- 规范禁止依赖：config/install/object
- 实际依赖：`dsp/filter`、`dsp/math`
- 违规项：无 ✅

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 91–113 | `advance` 状态机清晰 ✓ | 无 |
| 124–137 | 双层循环 | 已最简（RT 手写循环） |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无 |
| 🟡 建议级 | 0 | 无 |
| 🟢 优化级 | 2 | 文件头路径；NaN 路径 debug_assert |
