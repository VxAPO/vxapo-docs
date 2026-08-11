# 代码审查报告：pipeline/dsp/peq_hybrid.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\pipeline\dsp\peq_hybrid.rs`
- 审查模块：`pipeline/dsp/peq_hybrid`（混合式 PEQ：IIR 低频 + 最小相位 FIR 高频）
- 参照规范：`pipeline 模块规范.md`（v9.11 混合 PEQ）、主规范 RT 约束
- 总体评价：⚠️ 需修改（轻微）

---

## 审查摘要

- DSP 质量高：T−L 差分合成 + cepstrum 最小相位 FIR、宽 Q 段“影响跨分频点”判据（0.25 dB 阈值）有完整设计注释、静音恢复 8ms 淡入 + 64 帧保持防过零误判、不稳定段直通降级；RT 路径零分配（Direct dot / Partitioned FFT 均预分配）。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 174–177 | `sr × 0.0213` 魔法常数无来源说明 | 补注释（FIR 时长≈21.3ms？与分辨率/延迟权衡） |
| 363–374 | `in_iir_band` 的判据注释非常详细 ✓ | 无 |
| 457–462 | `peaking_response_db` 的“同一计算路径”说明有价值 ✓ | 无 |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 240–250 | `frames` 由各通道长度取 min + clamp——防御正确 ✓ | 无 |
| 252–286 | 静音检测：峰值阈值 1e-4（约 -80 dBFS）与 64 帧保持 ✓ | 无 |
| 276–286 | `input_active=false` 时输出恒 0 且状态照常更新 ✓ | 无 |
| 299–324 | Direct FIR 环形分段点积边界（delay_len == fir_len）有注释 ✓ | 无 |
| 330–338 | `latency()` 保守取 fir_len/4——不上报引擎，仅记账 ✓ | 无 |
| 185–189 | 空段直通（模型层已限制 1–31 段）✓ | 无 |
| 199–207 | 不稳定段 `warn_rate_limited` 限频 ✓ | 无 |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| — | 无 unsafe/FFI（SIMD/FFT 封装在 fir.rs/rustfft） | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 11–17 | 依赖 `rustfft`、`dsp/filter`、`dsp/fir`、`dsp/math`、`dsp/model` —— 符合 dsp/\*.rs 约束 ✓ | rustfft 建议规范登记（同 fir.rs） |
| 36–44 | 内部 `Biquad` 与 `dsp/biquad.rs` 重复实现（RBJ peaking）——两处漂移风险 | 与 biquad.rs 统一（或说明为什么自包含：稳定判据/DF2T 差异） |

### 依赖违规检查

- 本文件所在模块：`pipeline/dsp/peq_hybrid.rs`
- 规范允许依赖：`dsp/filter`、`dsp/model`、`dsp/biquad`、`utils`
- 规范禁止依赖：config/install/object
- 实际依赖：`dsp/filter`、`dsp/fir`、`dsp/math`、`dsp/model`、`rustfft`
- 违规项：无 ✅

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 385–455 | cepstrum 流水线步骤注释编号清晰 ✓ | 无 |
| 240–328 | process 单函数较长（约 90 行） | 可拆 `fn fade_factor`/`fn process_fir`，当前可接受 |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无 |
| 🟡 建议级 | 2 | 0.0213 常数来源注释；内部 Biquad 与 biquad.rs 去重/说明 |
| 🟢 优化级 | 1 | process 拆分可选 |
