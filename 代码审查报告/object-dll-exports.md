# 代码审查报告：object/dll_exports.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\object\dll_exports.rs`
- 审查模块：`object/dll_exports`（DllMain/DllGetClassObject/DllRegisterServer/DllUnregisterServer/DllCanUnloadNow）
- 参照规范：`object 模块规范.md`（Note 28/29/30/59）、`模块引用规范（无详细模块版）.md` 第十一节
- 总体评价：⚠️ 需修改

---

## 审查摘要

- 关键风险：
  1. **R3 边界**：`DllGetClassObject` 中 `transmute_copy(&factory)` + vtable 槽 `transmute`（151–157 行）无独立 SAFETY 注释块，依赖零散行注释。
  2. `register_apo_with_path` 的失败回滚只覆盖**已成功条目**（`j < i`），当前条目若已写 CLSID 父键后 InprocServer32 写失败，会遗留半成品注册（247–254 行）。
  3. `get_dll_path` 固定 1024 缓冲区，路径恰好满 1024 字符时 `GetModuleFileNameW` 截断且无法区分（298–306 行）。

优点：DllMain 严格极简（Loader Lock 铁律注释完整）、DllGetClassObject 参数校验顺序正确、AudioEngine 11 字段注册与 EAPO 对齐、注销幂等、测试覆盖全导出函数。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 1、23 | 文件头旧路径 `host/installation/exports.rs`/`host/installation/install.rs` | 改为 `object/dll_exports.rs` |
| 130–138、170–179 | debug 探针残留 | 删除或 feature 门控 |
| 149–150 | “Phase 5: 升级 windows-rs 后可移除”注释 | 若确认当前绑定无法避免，标注版本号与替代方案 |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 247–254 | 当前条目半成品不参与回滚（见摘要 2） | 单条目注册也记录“已创建键”，失败时删除本条目已写键 |
| 298–306 | 1024 缓冲可能截断 DLL 路径 | 返回 `ERROR_INSUFFICIENT_BUFFER`（len == 1024）时扩容重试 |
| 292–295 | `MODULE_HANDLE` 为空返回 None → DllRegisterServer 返回 SELFREG_E_CLASS ✓ | 无 |
| 325、334–362 | 大量 `map_err(|e| e.code())` 重复 | 抽 `fn wr<T>(r: Result<T>) -> Result<T, HRESULT>` helper |
| 279 | 注销失败继续（best-effort）✓ | 无 |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 62–67、70–74 | `DllMain` unsafe 有函数级 Safety 文档 ✓；实现极简符合 Loader Lock 约束 ✓ | 无 |
| 117–128 | `DllGetClassObject` 空指针校验先于解引用 ✓ | 无 |
| 151–157 | `transmute_copy`/`transmute` 缺独立 SAFETY 注释（R3） | 补：factory 为 `#[implement]` 有效 COM 对象，vtable 槽 0 为 QI（windows-interface 布局保证） |
| 159 | `qi(raw_ptr, ...)` 在 `drop(factory)` 之前调用 ✓ | 无 |
| 125–128 | `*ppv = null` 先置空再路由 ✓ | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 32–36 | 依赖 `object/factory`、`object/ref_count`、`object/vx_reg_props`、`sys/com/prelude`、`sys/registry` —— 与规范 dll_exports 行一致（telemetry 未用，可精简规范） | 无 |
| 203–211 | AudioEngine 常量集中、带实证注释 ✓ | 无 |
| 388–600 | 测试覆盖全部导出函数 ✓；`release_com_ptr` 有 Safety 注释 ✓ | 无 |

### 依赖违规检查

- 本文件所在模块：`object/dll_exports.rs`
- 规范允许依赖：`object/*`、`sys/com/prelude`、`sys/registry`（CLSID 键写入）、`utils`、`telemetry`
- 实际依赖：`object/factory`、`object/ref_count`、`object/vx_reg_props`、`sys/com/prelude`、`sys/registry`
- 违规项：无 ✅

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 243–258 | 手动 for + 逆序回滚 | 可保留（清晰）；或 `try_for_each` + 已处理列表 |
| 318–366 | 注册函数顺序执行 + 每步 map_err | 抽 `fn write_sz/kw` 包装，减少重复 |
| 291–307 | `get_dll_path` 线性 | 循环扩容 `buf.resize(buf.len()*2, 0)` 直到成功 |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无 RT/unsafe unsound/依赖红线（R3 属注释缺失非 unsound） |
| 🟡 建议级 | 3 | 当前条目半成品不回滚；DLL 路径截断；R3 注释补全 |
| 🟢 优化级 | 3 | 探针清理；错误映射 helper；路径缓冲扩容 |
