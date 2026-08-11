# 代码审查报告：config/model.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\config\model.rs`
- 审查模块：`config/model`（FileModel → ChainModel 转换与校验）
- 参照规范：`config 模块规范.md` 6.0（v9.11 TOML 模型）、`模块引用规范（无详细模块版）.md` 第十一节
- 总体评价：✅ 通过

---

## 审查摘要

- 校验严格且完整：未知键（flatten extra）、不适用字段、必填字段、范围/有限性、重复通道、dither 枚举、单块 1–31 段 + 全局 31 段预算（v9.16）全部落地；依赖方向 config → dsp 正确。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 241–278 | `set_fields()` 手工 28 字段布尔表冗长 | 可宏或 `serde` field 探测；当前可接受 |
| 381–396 | `into_reverb` 的 `unit` 闭包内嵌套 match 取默认值，可读性一般 | 抽 `fn unit_or_default(v, name, d) -> Result<f32>` |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 137–159 | 全局 PEQ 预算校验 ✓ | 无 |
| 280–308 | 通道去重/非空校验 ✓（未与设备真实通道名比对——Chain 层职责） | 无 |
| 497–512 | `finite_range` 拒绝 NaN/越界 ✓ | 无 |
| 427–441 | dither 大小写不敏感 ✓ | 无 |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| — | 无 unsafe/FFI；控制路径 | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 12–20 | 依赖 `config/error` + `pipeline/dsp/{aural,maximizer,model,reverb,wide}` —— 与规范 config/model.rs 行**一致** ✓ | 无 |
| 119–121 | `#[serde(flatten)] extra` 严格校验设计 ✓ | 无 |

### 依赖违规检查

- 本文件所在模块：`config/model.rs`
- 规范允许依赖：`config/error`、`pipeline/dsp/model`、`pipeline/dsp/{aural,maximizer,reverb,wide}`（参数结构）
- 规范禁止依赖：install/object、pipeline/process/chain
- 实际依赖：`config/error`、`pipeline/dsp/{aural,maximizer,model,reverb,wide}`
- 违规项：无 ✅

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 172–188 | 7 变体 match 分派到 `into_*` ✓ | 无 |
| 241–278 | 手工字段表 | `macro_rules!` 生成或按效果器拆分（可维护性次要） |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无 |
| 🟡 建议级 | 0 | 无 |
| 🟢 优化级 | 2 | set_fields 宏化；unit 闭包抽取 |
