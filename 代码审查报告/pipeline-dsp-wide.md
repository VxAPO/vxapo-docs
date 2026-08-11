# 代码审查报告：pipeline/dsp/wide.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\pipeline\dsp\wide.rs`
- 审查模块：`pipeline/dsp/wide`（Wide 立体声加宽器）
- 参照规范：`pipeline 模块规范.md` 4.22（v2.4 设计）、主规范 RT 约束
- 总体评价：✅ 通过

---

## 审查摘要

- 线性相位 FIR 分频 + 完美重建（测试逐样本验证 `lp+hp == 延迟输入`）、M/S 宽度处理曲线有明确公式与参数表、tanh 软限幅有界、Intensity=0 位精确直通；RT 路径零分配（SIMD dot）；测试质量全项目顶级（重建/群延迟/谐波/反相有界/确定性/跨采样率）。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 1–18 | v2.4 设计说明完整 ✓ | 无 |
| 37–59 | `parse_wide_params` 疑似旧命令解析遗留（同 aural） | grep 确认调用点 |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 111–139 | 环形分段点积：`oldest <= start` 连续路径与两段回绕路径正确 ✓ | 无 |
| 137 | `delayed_x` 索引 `(*pos + delay_len - 1 - center) & mask` —— 中心对齐正确 ✓ | 无 |
| 225–230 | L/R 槽位越界与 `frame_count.min(len)` 防御 ✓ | 无 |
| 251–252 | `ll + tanh(proc)` 低频原样相加——反相极端下输出 ≤1.1 由测试限定 ✓ | 无 |
| 193 | `new` 时 `FirSplit::new(vec![1.0], 0)` 占位，`initialize` 才建真实 FIR——未初始化即 process 会走 `active=false` 直通 ✓ | 无 |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| — | 无 unsafe/FFI（SIMD 封装在 fir.rs，已审） | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 20、96、127–133 | 依赖 `dsp/filter`、`dsp/fir` —— 符合 dsp/\*.rs 约束 ✓ | 无 |
| 62–73 | 常量集中、带设计公式注释 ✓ | 无 |

### 依赖违规检查

- 本文件所在模块：`pipeline/dsp/wide.rs`
- 规范允许依赖：`dsp/filter`、`dsp/model`、`dsp/biquad`、`utils`
- 规范禁止依赖：config/install/object
- 实际依赖：`dsp/filter`、`dsp/fir`
- 违规项：无 ✅

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 126–135 | 分段 dot 两分支 | 已最简；可抽 `fn dot_segment` helper |
| 244–252 | M/S 处理算术清晰 ✓ | 无 |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无 |
| 🟡 建议级 | 1 | parse_wide_params 死代码确认 |
| 🟢 优化级 | 1 | dot_segment helper |
