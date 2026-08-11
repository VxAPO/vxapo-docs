# 代码审查报告：pipeline/dsp/factory.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\pipeline\dsp\factory.rs`
- 审查模块：`pipeline/dsp/factory`（模型 → Filter 静态分派）
- 参照规范：`pipeline 模块规范.md` 4.10（v9.11 静态 match）、`模块引用规范（无详细模块版）.md` 第十一节
- 总体评价：✅ 通过

---

## 审查摘要

- 静态 `match` 穷尽 7 种效果器、`enabled=false` 直通、Loudness 尊重 ctx 开关；已删除动态注册表/字符串解析（v9.11 收敛）；构造全部控制路径。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 1–6 | 模块头说明 v9.11 收敛 ✓ | 无 |
| 39–50 | `matches_kind` 用于 debug_assert，`kind` 字段在生产 match 中未被使用 | 补注释：“params 为权威，kind 仅指纹/调试” |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 19–37 | 构造“不失败”的前提是 config 层已校验——`debug_assert` 仅调试生效 | 可接受（config 层职责）；建议 release 下 `match` 不依赖 kind 一致性（当前已如此） |
| 31–35 | Loudness 由 `ctx.loudness_enabled` 覆盖 —— 与 `EffectConfig.enabled` 语义叠加清晰 ✓ | 无 |
| 25 | `HybridPeqFilter::new(p.clone())` 深拷贝 bands——控制路径 | 可接受；若可借用则省一次分配 |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| — | 无 unsafe/FFI/RT 路径 | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 8–16 | 依赖全部 7 个具体 Filter + `dsp/filter` + `dsp/model` —— 与规范 factory.rs 行**一致** ✓ | 无 |
| 52–161 | 测试覆盖全部类型 + 禁用 + Loudness 开关 ✓ | 无 |

### 依赖违规检查

- 本文件所在模块：`pipeline/dsp/factory.rs`
- 规范允许依赖：`dsp/filter`、`dsp/model`（EffectConfig/EffectType）、`dsp/*`（静态 match）、`utils`
- 规范禁止依赖：config/install/object
- 实际依赖：`dsp/{aural,filter,gain,loudness,maximizer,model,peq_hybrid,reverb,wide}`
- 违规项：无 ✅

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 19–37 | 大 match 构造 | 可让各 Filter 实现 `From<Params>`，工厂 `p.clone().into()`；当前已清晰，可选 |
| 39–50 | matches_kind 与 match 双份枚举 | 可接受（调试守卫） |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无 |
| 🟡 建议级 | 0 | 无 |
| 🟢 优化级 | 2 | kind 权威性注释；Peq clone 复用 |
