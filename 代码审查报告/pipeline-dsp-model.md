# 代码审查报告：pipeline/dsp/model.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\pipeline\dsp\model.rs`
- 审查模块：`pipeline/dsp/model`（ChainModel / EffectType / 参数结构）
- 参照规范：`pipeline 模块规范.md`（v9.11 模型）、`模块引用规范（无详细模块版）.md` 第十一节
- 总体评价：✅ 通过

---

## 审查摘要

- 参数模型纯数据、无 Windows 依赖；`spec()` 指纹稳定（`{:.6}` 精度 + 有序拼接），供热重载短路；`EffectType::from_str/as_str` 双向映射完整；依赖方向 config → dsp 正确。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 76–121 | `spec()` 用 `format!` 拼接长字符串，各分支结构相似 | 可抽 `fn push_param(s, name, v)` helper；控制路径可接受 |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 85–89 | 指纹精度 `{:.6}`——参数变化 < 1e-6 时热重载不触发 | 可接受（可闻差异远大于 1e-6）；注释说明 |
| 48–59 | `from_str` 大小写不敏感 ✓ | 无 |
| 21–23 | `effects: Vec<EffectConfig>` 空链即 passthrough ✓ | 无 |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| — | 无 unsafe/FFI/RT 路径 | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 7–10 | 依赖 `dsp/{aural,maximizer,reverb,wide}`（参数结构）—— 与规范 model 相关行一致 ✓ | 无 |
| 12–17 | 常量集中 + v9.16 来源注释 ✓ | 无 |
| 35–45 | `EffectType` 7 变体与 `EffectParams` 7 变体一一对应，靠 `debug_assert` 维持 | 可加 `#[cfg(test)]` 全组合测试 |

### 依赖违规检查

- 本文件所在模块：`pipeline/dsp/model.rs`
- 规范允许依赖：`dsp/filter`、`dsp/model`、`dsp/biquad`、`utils`（dsp/\*.rs 通用行）
- 规范禁止依赖：config/install/object
- 实际依赖：`dsp/{aural,maximizer,reverb,wide}`（仅参数结构）
- 违规项：无 ✅

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 126–134 | `EffectParams` 枚举 + 具体结构 ✓ | 无 |
| 47–72 | `from_str/as_str` 对称 ✓ | 无 |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无 |
| 🟡 建议级 | 0 | 无 |
| 🟢 优化级 | 2 | spec 精度注释；变体一致性测试 |
