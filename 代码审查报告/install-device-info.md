# 代码审查报告：install/device/info.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\install\device\info.rs`
- 审查模块：`install/device/info`（组合查询层 + 设备枚举唯一入口）
- 参照规范：`install 模块规范.md` 5.4、`模块引用规范（无详细模块版）.md` 第十一节
- 总体评价：⚠️ 需修改

---

## 审查摘要

- 关键风险：
  1. `enumerate_devices` 中单个端点 `query_device_info(...)?` 失败会中止**整轮枚举**，一个坏端点导致全部设备列表不可得（178 行）；两个根路径共用同一 `?` 风格，Render 枚举出错即跳过 Capture（172 行）。
  2. 半安装态判定歧义：LFX 单独为 VxAPO（版本值存在但槽位不成对）时 `is_installed()` 返回 false，与“版本非空即已安装”的直觉不符（51–60 行）。
  3. `read_format_for_endpoint` 内联硬编码两个 PKEY 值名且无来源注释（309–310 行）。

优点：“只认 VxAPO CLSID 成对”的安装判定与 v8.9 实证一致、E3.2 零 I/O 谓词实现干净、端点 GUID 回填正确、依赖与规范行完全一致、测试覆盖好。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 309–310 | 两个 PKEY 值名内联在函数中，无常量名/无来源 | 提为 `const DEVICE_FORMAT_PKEY` / `const CHANNEL_MASK_PKEY` 并注释来源（如 mmdeviceapi.h） |
| 89 | `has_changes` 文档“是否有未应用的更改”与实现“非默认模式”语义跨度大 | 改名 `is_non_default_mode()` 或补充用途说明 |
| 6–15 | 依赖引用清晰 ✓ | 无 |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 172 | `root.enum_sub_keys()?` 失败直接中止整个枚举 | 单根失败 `continue` + 日志，另一根继续 |
| 178 | 单个端点查询失败中止全部枚举 | `match` 后 `continue` + `log::warn!`，坏端点跳过 |
| 51–60 | 半安装态（版本存在但槽位不成对）被判定为未安装 | 明确语义：`version 非空` 单独成字段/方法（`has_install_mark()`），避免与“成对安装”混淆 |
| 319–329 | `version` 只接受 `RegValue::Sz`，若旧版写 DWORD 则永远“未安装” | 兼容 DWORD 读取，或与写入端确认类型 |
| 78–85 | `is_enhancements_disabled` 吞掉所有错误返回 false | 与 endpoint.rs 相同的错误区分建议 |
| 493–496 | `enumerate_devices_runs_without_error` 是真实注册表访问的单测，CI/无权限环境可能不稳定 | 加 `#[ignore]` 或 mock 注入层 |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| — | 无 unsafe、无 FFI、无 RT 路径；只读注册表操作 | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 6–15 | 依赖 `device/endpoint`、`device/format`、`device/slots`、`sys/registry`、`object/vx_reg_props`、`utils/error` —— 与规范 5.4 行（info）**一致** ✓ | 无 |
| 20–26 | 路径常量集中 ✓ | 无 |
| 129–157 | `query_device_info` 恒返回 `Ok(Some(...))`，`Option` 形同虚设 | 简化返回 `DeviceInfo`，或保留并让 `query_endpoint` 的可空语义真正落地 |

### 依赖违规检查

- 本文件所在模块：`install/device/info.rs`
- 规范允许依赖：`device/endpoint`、`device/format`、`device/slots`、`sys/registry`、`object/vx_reg_props`、`utils/error`
- 规范禁止依赖：`pipeline/`、`config/`
- 实际依赖：`install/device/{endpoint,format,slots}`、`sys/registry`（含 `is_windows_version_at_least`）、`object/vx_reg_props`、`utils/error`
- 违规项：无 ✅（与规范 5.4 行完全一致）

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 268–285 | 三连 if 判定成对模式 | 定义 `[(ApoSlot, ApoSlot, InstallMode); 3]` 常量表 + `find_map`，或保持显式 if（可读性已可接受） |
| 255–260 | `is_vxapo_slot` + `is_vxapo_pre/post` 三函数重叠 | `SlotValue::Guid(g)` 与 `[PRE, POST].contains(g)` 合一，或 `fn is_vxapo(g: &GUID) -> bool` |
| 222–232 | `detect_mode_for_device` 组合清晰 ✓ | 无 |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无 RT/unsafe/依赖红线违规 |
| 🟡 建议级 | 3 | 单端点失败中止全枚举；半安装态判定歧义；版本值类型兼容 |
| 🟢 优化级 | 4 | PKEY 常量化；`Option` 简化；判重函数合并；真实注册表单测隔离 |
