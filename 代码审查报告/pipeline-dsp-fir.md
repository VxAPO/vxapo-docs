# 代码审查报告：pipeline/dsp/fir.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\pipeline\dsp\fir.rs`
- 审查模块：`pipeline/dsp/fir`（SIMD dot + 分块 FFT 卷积）
- 参照规范：`pipeline 模块规范.md`（v9.11 FIR 基础设施）、主规范 RT 约束
- 总体评价：⚠️ 需修改（RT 边界）

---

## 审查摘要

- 关键风险：
  1. **RT 路径依赖 rustfft 运行时行为**：`process_block`（224–261 行）在 RT 调用 `fft.process_with_scratch`/`ifft.process_with_scratch`——rustfft 的 in-place scratch API 不分配，但依赖第三方保证；建议在文档/测试中固化“RT 无分配”验证。
  2. `dot` 对 `a`/`b` 长度不等时静默截断到 `min`（33–46 行），调用方需保证等长。
  3. `latency()` 返回 `block_len`（128），而实际算法延迟为 `block_len - 1`（测试 304 行自证）——对外口径偏大 1。

优点：AVX2+FMA 运行时探测 + 8 路 FMA + 标量尾正确、分块 overlap-add 语义（跨调用累积、Reset 清空）与 EAPO 对齐、测试含朴素卷积对照/变帧数不变量/Reset。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 1 | 文件头无版本注释 | 可补 v9.11 来源 |
| 217–220 | `latency` 注释“块大小”与实现 `block_len` 一致但实际延迟差 1 | 修正为 `block_len - 1` 或注释口径 |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 33–46 | dot 长度不匹配静默截断 | `debug_assert_eq!(a.len(), b.len())` |
| 170–202 | `in_buf[in_len]` 写索引：状态机保证 `in_len < block_len`（块处理后即复位） | 加 `debug_assert!(ch.in_len < block_len)` |
| 247 | `&ch.x_ring[((slot + blocks - b) % blocks) * fft_len..]` 切片只取起始，长度依赖循环内 `0..fft_len` 不越界（slot 回绕保证） | 可改为双切片显式边界 |
| 116 | `ir.len().div_ceil(block_len).max(1)` ✓ | 无 |
| 113–164 | `new` 控制路径分配 ✓ | 无 |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 36–38 | AVX2 分支有 SAFETY 注释 + target_feature 门控 ✓ | 无 |
| 57–58 | `_mm256_loadu_ps` 非对齐加载 ✓ | 无 |
| 70 | `get_unchecked` 尾路径依赖 `i < n` 不变式 ✓ | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 9–14 | 依赖 `rustfft`（第三方）+ `dsp/math` —— 符合 dsp 层约束；rustfft 属新增第三方依赖，规范未列 | 规范侧补录 `rustfft` 为允许第三方依赖 |
| 89–97 | 手写 `Debug` 避免打印巨大缓冲 ✓ | 无 |

### 依赖违规检查

- 本文件所在模块：`pipeline/dsp/fir.rs`
- 规范允许依赖：`dsp/filter`、`dsp/model`、`dsp/biquad`、`utils`
- 规范禁止依赖：config/install/object
- 实际依赖：`dsp/math`、`rustfft`（第三方）
- 违规项：无 ✅（rustfft 建议登记）

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 235–240 | 手工复制 + 补零 | `x_work[..block_len].copy_from_slice(&in_buf); x_work[block_len..].fill(0)` |
| 246–252 | 分块频域累加循环 | 已最简；可 `xs.iter().zip(hs).zip(acc).for_each(...)` 但性能优先保留 |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无（rustfft scratch API 无分配，RT 边界靠第三方契约） |
| 🟡 建议级 | 2 | RT 无分配契约文档化；latency 口径修正 |
| 🟢 优化级 | 3 | debug_assert 等长/写索引；复制循环简化；规范登记 rustfft |
