# 代码审查报告：object/vx_reg_props.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\object\vx_reg_props.rs`
- 审查模块：`object/vx_reg_props`（CLSID + 注册属性常量 + 注册顺序）
- 参照规范：`object 模块规范.md` 7.5、`模块引用规范（无详细模块版）.md` 第十一节
- 总体评价：⚠️ 需修改（轻微）

---

## 审查摘要

- 关键风险：
  1. 依赖 `sys/com/apo_interfaces::IID_IAPO`（17、95 行）超出规范行“`sys/com/prelude` + `sys/com/apo_types`”字面清单。
  2. `str_to_u16_256` 按字节转 u16（63–74 行），仅 ASCII 安全——当前名称/版权为 ASCII，未来若改中文会损坏 UTF-16。
  3. 文件头仍写旧路径 `host/instance/reg_props.rs`。

优点：CLSID 定死并附生成来源、编译期断言（Flags/接口数/名称）非常有价值、注册/注销顺序注释完整。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 1 | 文件头旧路径 | 改为 `object/vx_reg_props.rs` |
| 23–26 | GUID 来源注释 ✓ | 无 |
| 296–299 | AudioEngine 路径的 P0-7 根因注释有价值 ✓ | 无 |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 63–74 | 非 ASCII 会静默损坏（字节截断为 u16） | 改用 `encode_utf16` 真转换，或 `const fn` 限制 ASCII 并加断言 |
| 68 | 写入到 `dst < 255`，尾部始终留一个 `0` 终止符 ✓ | 无 |
| 301–306 | `registration_entries` 返回元组列表，类型无命名 | 可改 `struct RegEntry { name, value }` |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| — | 无 unsafe/FFI/RT 路径 | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 15–18 | 依赖 `sys/com/prelude`、`sys/com/apo_types`、`sys/com/apo_interfaces`（IID_IAPO） | 规范侧把 `sys/com/apo_interfaces`（仅 IID_IAPO）补入 vx_reg_props 行 |
| 272–317 | `ClsidEntry` 与 `registration_order/unregistration_order` 属注册辅助，逻辑简单 ✓ | 无 |

### 依赖违规检查

- 本文件所在模块：`object/vx_reg_props.rs`
- 规范允许依赖：`sys/com/prelude`、`sys/com/apo_types`
- 规范禁止依赖：所有其他
- 实际依赖：`sys/com/prelude`、`sys/com/apo_types`、`sys/com/apo_interfaces`（仅 `IID_IAPO`）
- 违规项：⚠️ `sys/com/apo_interfaces::IID_IAPO` 未在规范行列出（sys 层同族常量）→ 规范补录即可，无实质越界。

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 80–97 | `make_reg_props` 纯 const fn ✓ | 无 |
| 126–142 | 三个查询辅助简单清晰 ✓ | 无 |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无 RT/unsafe/依赖红线 |
| 🟡 建议级 | 1 | 依赖行补录（IID_IAPO） |
| 🟢 优化级 | 2 | 文件头路径；UTF-16 编码健壮化 |
