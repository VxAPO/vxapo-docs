# 代码审查报告：pipeline/dsp/biquad.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\pipeline\dsp\biquad.rs`
- 审查模块：`pipeline/dsp/biquad`（双二阶滤波器基础实现）
- 参照规范：`pipeline 模块规范.md` 4.12（Note 13/53/58）、主规范 RT 约束
- 总体评价：⚠️ 需修改（轻微）

---

## 审查摘要

- 全项目数值质量标杆：RBJ 公式 f64 中间计算、朱利稳定性 + 深切地板（-60 dB）回退、DF2T f64 状态 + FMA、立体声 SSE2/FMA 两路 SIMD 与标量逐位一致、NaN 输出清状态兜底；测试覆盖布局断言/三种结构一致性/SIMD vs 标量/槽位路由/极限参数，质量顶级。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 1 | 文件头旧路径 `dsp/filters/biquad.rs` | 改为 `pipeline/dsp/biquad.rs` |
| 331–341 | SIMD 路径注释非常详细（作用域语义/指令数/误差）✓ | 无 |
| 489 | `set_coeffs` 注释“Phase 8+”——参数平滑尚未实现 | 标注当前仅测试用途 |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 102–147 | `compute_coeffs` 回退链完备（sr=0/NaN/不稳定/深切地板）✓ | 无 |
| 303–322 | NaN 输出清状态 ✓ | 无 |
| 607–634 | SIMD 分支条件（num_ch==2、槽位不同、len≥2）完整；`split_at_mut` 借用正确 ✓ | 无 |
| 611 | `is_x86_feature_detected!("fma")` 每帧调用 | 可缓存到 `OnceLock`/static（同 fir.rs 模式） |
| 549–555 | `num_channels` 两次赋值（max 后 min）可读性一般 | 合并为单表达式 + 注释 |
| 490–492 | `set_coeffs` 直接替换系数——若 RT 线程运行中调用会撕裂状态（当前仅控制路径/测试） | 注释“非 RT 调用” |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 392–396 | `simd_df2t::process_sample` 有完整 Safety 注释 ✓ | 无 |
| 625–628 | 调用处 SAFETY 注释 ✓ | 无 |
| 349–388 | `step_sse2/step_fma` 为内部 unsafe fn，`#[target_feature]` 门控 ✓ | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 17–21 | 依赖 `dsp/filter`、`dsp/math` —— 符合 dsp/\*.rs 约束 ✓ | 无 |
| 274–283 | `BiquadState` repr(C) 16 字节布局有注释 + 测试断言 ✓ | 无 |
| 664–1136 | 测试覆盖全面 ✓ | 无 |

### 依赖违规检查

- 本文件所在模块：`pipeline/dsp/biquad.rs`
- 规范允许依赖：`dsp/filter`、`dsp/model`、`dsp/biquad`、`utils`
- 规范禁止依赖：config/install/object
- 实际依赖：`dsp/filter`、`dsp/math`
- 违规项：无 ✅

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 132–146 | 地板回退 loop | 已清晰；可抽 `fn coeffs_with_floor(...)` |
| 576–649 | process 三结构 match + SIMD 分支 | 可拆 `fn process_df2t`，当前可接受 |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无 |
| 🟡 建议级 | 2 | FMA 探测缓存；set_coeffs RT 约束注释 |
| 🟢 优化级 | 2 | 文件头路径；num_channels 表达式合并 |
