# 代码审查报告：pipeline/dsp/loudness.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\pipeline\dsp\loudness.rs`
- 审查模块：`pipeline/dsp/loudness`（ISO 226 等响近似）
- 参照规范：`pipeline 模块规范.md` 4.21、`模块引用规范（无详细模块版）.md` 第十一节
- 总体评价：⚠️ 需修改（轻微）

---

## 审查摘要

- 关键风险：
  1. **占位实现与规范成熟度不符**：`iso_226_approx` 是启发式近似（127–144 行），注释自称“Phase 8+ 补全”——若当前版本即交付，音质验收需按近似处理。
  2. `parse_loudness_params`（149–169 行）疑似旧命令解析遗留，v9.11 后 config 走 model 层，需确认是否死代码。
  3. `initialize` 对 `phon - reference` 无上限约束（diff 可 ±120），增益经 clamp 兜底 ✓。

优点：biquad 级联在控制路径构建、RT 零分配、开关/参考级直通、Q=1.414 有注释、测试覆盖开关与静音。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 1 | 文件头旧路径 `dsp/filters/loudness.rs` | 改为 `pipeline/dsp/loudness.rs` |
| 13–14 | “占位实现”注释与“已实现”矛盾 | 更新为“近似实现，Phase 8 查表升级” |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 78–82 | `diff.abs() < 0.1` 直通阈值 ✓ | 无 |
| 87–102 | 29 个频率点逐点建 biquad，`gain.abs() > 0.05` 过滤 ✓ | 无 |
| 149–169 | 参数解析疑似死代码（v9.11 model 化） | grep 确认后删除或移入测试 |
| 127–144 | 近似曲线无文献数值表对照 | 补充参考数据或标注“启发式” |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| — | 无 unsafe/FFI；RT 仅 biquad 级联 | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 16–18 | 依赖 `dsp/filter`、`dsp/biquad`、`dsp/math` —— 与规范“loudness 内部用 biquad”一致 ✓ | 无 |
| 36–41 | ISO 频率表常量集中 ✓ | 无 |

### 依赖违规检查

- 本文件所在模块：`pipeline/dsp/loudness.rs`
- 规范允许依赖：`dsp/filter`、`dsp/model`、`dsp/biquad`、`utils`
- 规范禁止依赖：config/install/object
- 实际依赖：`dsp/filter`、`dsp/biquad`、`dsp/math`
- 违规项：无 ✅

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 87–102 | `for (_i, &freq) in ISO_FREQUENCIES.iter().enumerate()` 的 `_i` 未用 | 直接 `.iter().copied()` |
| 107–111 | RT 级联循环 ✓ | 无 |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无 |
| 🟡 建议级 | 2 | 近似实现交付声明；死代码确认 |
| 🟢 优化级 | 2 | 文件头路径；`_i` 清理 |
