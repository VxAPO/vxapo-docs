# 代码审查报告：object/apo/init.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\object\apo\init.rs`
- 审查模块：`object/apo/init`（Initialize / GetRegistrationProperties）
- 参照规范：`object 模块规范.md` 7.1.8、`模块引用规范（无详细模块版）.md` 第十一节
- 总体评价：⚠️ 需修改

---

## 审查摘要

- 关键风险：
  1. `Initialize` 是 COM 入口但无 catch_unwind，内部 `child_apo.lock().unwrap()`/`config_path.lock().unwrap()` 可因 mutex 中毒 panic 跨 FFI（68、112 行）。
  2. 每次流初始化都执行 `diag_append` 文件 I/O（84–91 行），audiodg 高频实例化时控制线程写盘频繁，且该文件无大小上限。
  3. 运行期自愈依赖 `install/selector/operation::find_endpoint_path` 与 `install/device/sysfx`，object 规范 7.1.8 引用清单未登记（74–97 行）。

优点：数据非法降级不阻断加载、子 APO 失败降级（Note 57）、自愈“仅动微软 CAPX + 幂等 + 失败仅日志”语义正确、SAFETY 注释基本到位。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 23–32 | debug 探针残留（“排查完删除”） | 删除或 feature 门控 |
| 70–97 | 自愈 + 诊断日志混在“解析端点”步骤中，步骤编号 4→5b→5 跳变 | 拆出 `fn self_heal_msfx(eg: &GUID, clsid: GUID)` |
| 100–111 | 与 46–51 行重复的 `unsafe` 解引用 + `valid_init_data` 分支 | 先一次解引用存 `Option<&APOInitSystemEffects>`，两处共用 |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 68、112 | `lock().unwrap()` 在 COM 入口内 | 与 `apo_process` 一致用 `unwrap_or_else(\|e\| e.into_inner())` 或 catch_unwind |
| 84–91 | `diag_append` 每次 Initialize 写盘、无门控无上限 | debug 门控 + 限频（可复用 math.rs 的 rate-limit 模式） |
| 74–97 | `find_endpoint_path` 失败静默忽略（注释说明）——但若端点路径定位失败，自愈永久跳过 | 可接受；建议对 `Err` 记 debug 日志 |
| 36–37 | `cb_data_size >= size_of::<APOInitSystemEffects>()` 校验 ✓ | 无 |
| 61–64 | `premix.or(postmix)`：PostMix 优先缺失时用 PreMix 作为子 APO——语义是“任一保留 APO 即子 APO”，注释可更明确 | 补一句决策说明 |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 45–48 | 第一处 `unsafe { &*pby_data }` 有 Safety 注释 ✓ | 无 |
| 100–102 | 第二处相同解引用无独立 Safety 注释（R3 边界） | 补注释或复用 47 行结果 |
| 62–63 | `ChildApo::create` unsafe 有 SAFETY 注释 ✓ | 无 |
| 128–135 | `CoTaskMemAlloc` + `ptr::write` 有 Safety 注释 ✓；对齐注释 ✓ | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 8–17 | 依赖 `super::child/config/state`、`install/device/slots`、`object/vx_reg_props`、`sys/com/apo_types/prelude` —— 基本符合 object 7.1.8 | 无 |
| 77、80 | 运行期依赖 `install/selector/operation`、`install/device/sysfx` —— **规范 7.1.8 未登记** | 规范侧补录这两个依赖（object 胶水层允许） |
| 84–91 | 跨模块全限定路径调用 `aggregate::AGG_*`/`config::diag_append` | 顶部 `use` 收口 |

### 依赖违规检查

- 本文件所在模块：`object/apo/init.rs`
- 规范允许依赖（7.1.8 相关）：`sys/com/*`、`object/vx_reg_props`、`install/device/slots`（v8.4）
- 规范禁止依赖：`object/apo`（禁止循环，按父表解读）
- 实际依赖：`super::child`、`super::config`、`super::state`、`install/device/slots`、`install/selector/operation`、`install/device/sysfx`、`object/vx_reg_props`、`sys/com/apo_types`、`sys/com/prelude`
- 违规项：⚠️ `install/selector/operation`、`install/device/sysfx` 未在 7.1.8 登记（模块级允许）→ 规范补录；`super::child/config/state` 为同目录协作，父表“禁止循环”解读下建议规范明确授权。

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 46–51、100–111 | 两次重复解引用 + 两次 `resolve_config_path` 分支 | `let init = (!pby_data.is_null() && cb >= size).then(\|\| unsafe {...});` 统一一次 |
| 56–67 | `premix.or(postmix).and_then(...)` 组合链清晰 ✓ | 无 |
| 120–136 | `get_registration_properties` 简洁 ✓ | 无 |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无 RT 分配/unsafe unsound/依赖红线（依赖为登记缺失非越界） |
| 🟡 建议级 | 3 | COM 入口 unwrap panic 源；Initialize 每次写盘；自愈依赖未登记 |
| 🟢 优化级 | 3 | 探针清理；重复解引用收敛；自愈函数拆分 |
