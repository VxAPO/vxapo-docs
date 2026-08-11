# 代码审查报告：object/apo/negotiate.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\object\apo\negotiate.rs`
- 审查模块：`object/apo/negotiate`（格式协商）
- 参照规范：`object 模块规范.md` 7.1.16、主规范 18.2（D2）、`模块引用规范（无详细模块版）.md` 第十一节
- 总体评价：⚠️ 需修改

---

## 审查摘要

- 关键风险：
  1. **上混拒绝不完整**：`is_input_format_supported` 只拒绝 `in > 2ch 且 in > out`；`2 → 6` 这类“立体声到多声道”上混会通过协商，但 VxAPO 只做 mono→stereo 上混，其余通道被静音填充——与规范 D2“仅 mono→stereo 上混”不符（68–72 行）。
  2. 协商入口是 COM 调用但无 catch_unwind，`expect("checked above")`（74、91 行）一旦被未来改动破坏即成跨 FFI panic 源。
  3. 43–45 行 `unsafe { is_float_format(mt_ptr) }` 无独立 Safety 注释（R3 边界）。

优点：S_FALSE 退化策略注释清晰、采样率不设限与 EAPO 对齐、降混拒绝正确、依赖与规范行一致。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 55–60 | S_FALSE 无法表达的注释很有价值 ✓ | 无 |
| 73–74 | `expect("checked above")` 依赖前面已 `as_ref()` 的隐式不变式 | 改用 `ok_or_else(|| Error::from(APOERR_INVALID_CONNECTION_FORMAT))?` |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 68–72 | `2→6`、`1→6` 等上混被接受（仅 `in>2` 才拒降混） | 对齐 D2：仅允许 `in==out` 或 `in==1 && out==2`，其余上/降混拒绝 |
| 86–90 | `is_output_format_supported` 对称问题：`6→2` 被拒 ✓，但 `2→6` 未被本函数拒绝（该方向由对端检查） | 与输入侧一起统一成“仅等通道或 mono→stereo”白名单 |
| 23–25、40–42 | 两处 `as_ref()` None → 错误 ✓ | 无 |
| 49–51 | 通道 1–8 校验 ✓ | 无 |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 26–28 | `extract_format_ref` 的 const→mut 转换有 Safety 注释 ✓ | 无 |
| 43–45 | `is_float_format(mt_ptr)` unsafe 无 Safety 注释（R3） | 补“mt_ptr 指向有效 COM 对象且调用期间有效” |
| 74、91 | `expect` 潜在 panic 跨 FFI（R4 边界） | 改为 `?` + 显式错误码 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 8–13 | 依赖 `pipeline/format`、`sys/com/apo_interfaces`、`sys/com/apo_types` —— 与父表 negotiate 行**一致** ✓ | 无 |
| 36–53 | `check_format_supported` 职责单一 ✓ | 无 |

### 依赖违规检查

- 本文件所在模块：`object/apo/negotiate.rs`
- 规范允许依赖：`sys/com/apo_interfaces`、`sys/com/apo_types`、`pipeline/format`、`utils`
- 规范禁止依赖：`object/apo`（禁止循环）
- 实际依赖：`pipeline/format`、`sys/com/apo_interfaces`、`sys/com/apo_types`
- 违规项：无 ✅

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 61–76、80–93 | 两个协商函数结构对称 | 抽 `fn validate_channel_relationship(in_ch, out_ch) -> bool`（等通道或 1→2），两处共用 |
| 43–45 | 重复的 const→mut 指针转换 | 封装 `fn media_type_ptr(r: &Ref<IAudioMediaType>) -> *mut IAudioMediaType` |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无 RT/unsafe unsound/依赖红线 |
| 🟡 建议级 | 2 | 上混白名单不完整（与 D2 不符）；expect panic 源 |
| 🟢 优化级 | 2 | Safety 注释补全；通道关系校验收敛 |
