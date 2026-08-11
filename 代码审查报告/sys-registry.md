# 代码审查报告：sys/registry.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\sys\registry.rs`
- 审查模块：`sys/registry`（注册表 RAII 封装 + .reg 备份导出）
- 参照规范：`sys 模块规范.md` 3.4、`模块引用规范（无详细模块版）.md` 第十一节
- 总体评价：🚫 严重违规（R3 + 备份格式错误）

---

## 审查摘要

- 关键风险：
  1. **R3**：45–46 行 `unsafe impl Send/Sync for RegKey` 无 SAFETY 注释。
  2. **`.reg` 备份导出 MULTI_SZ/QWORD 格式错误**（589–599、575–580 行）：MultiSz 直接写转义文本而非 UTF-16LE hex 字节，并以字面 `\0` 结尾——生成的 .reg 无效；QWORD 只导出低 16 位。FxProperties 备份含 ProcessingModes（REG_MULTI_SZ），恢复不可靠。
  3. **`delete_sub_key` 语义陷阱**：`RegDeleteTreeW(handle, name)` 的 `name` 是相对句柄的子键名，但调用方（operation.rs 事务回滚、本文件测试清理）都传完整路径——删除静默空操作。

优点：最小写权限 `open_for_write` 与 MMDevices ACL 实证注释、`read_value` 类型自动识别、UTF-16 解析正确、`win32_ok` 错误映射统一、`write_multi_value` 双 null 布局正确。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 406–415 | `delete_sub_key` 参数名 `name` 易误解为“完整路径” | 改名 `relative_child` 并加 doc：相对当前句柄的子键名 |
| 455–474 | `split_key` 支持 5 个根 ✓ | 无 |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 240–262、273–297 | 512 字符缓冲区：超长名返回 ERROR_MORE_DATA(234) 直接报错中断枚举 | 捕获 234 后扩容重试（上限 32767） |
| 151–166 | DWORD/QWORD 缓冲不足静默返回 0 | 改为 `Err`（数据损坏不可静默） |
| 222–224 | `value_exists` 把 ERROR_ACCESS_DENIED 也当 false | 区分“不存在”（false）与“访问失败”（Err） |
| 575–580 | QWORD 导出仅 2 字节 | 8 字节完整导出 `hex(b):lo,...,hi` |
| 589–599 | MULTI_SZ 导出无效格式（见摘要 2） | 按 UTF-16LE hex 逐项编码 + `00,00` 双终止 |
| 557 | 导出固定写 `HKEY_LOCAL_MACHINE` 头，HKCU 等根会生成错误 .reg | 按 root 参数输出对应根头 |
| 602 | 值读取失败静默跳过——备份不完整 | 收集警告日志 |
| 627–645 | 测试清理用 `open(...).delete_sub_key(TEST_KEY)` 同样是空操作，测试键残留 | 用 `delete_tree(TEST_ROOT, TEST_KEY)` |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 45–46 | `unsafe impl Send/Sync` 无 Safety 注释（R3） | 补：HKEY 句柄为值语义，Windows 句柄跨线程合法；内部无借用状态 |
| 50–52、61、76–88、100、121–143、222、242–253、276–294、329–391、398–413、493 | 各 unsafe 块均无逐块 SAFETY 注释（靠函数上下文） | 至少为句柄有效性/缓冲区长度补行内注释（对标其余模块标准） |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 3–10 | 依赖 `windows`（Registry/Foundation）+ `sys/com/prelude`（guid_to_string）—— 与规范 registry 行**一致** ✓ | 无 |
| 12–17 | SAM 常量带 ACL 实证注释 ✓ | 无 |

### 依赖违规检查

- 本文件所在模块：`sys/registry.rs`
- 规范允许依赖：windows-rs、`sys/com/prelude`（guid_to_string）
- 规范禁止依赖：其他
- 实际依赖：windows-rs、`sys/com/prelude`
- 违规项：无 ✅

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 116–172 | `read_value` 两阶段查询 ✓ | 无 |
| 545–615 | `dump_key_recursive` 手写 .reg 序列化 | 抽 `fn reg_escape(s) -> String` 与各类型编码函数 |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 1 | R3：unsafe impl Send/Sync 无 Safety 注释（45–46） |
| 🟡 建议级 | 4 | .reg MULTI_SZ/QWORD 导出格式错误；delete_sub_key 语义陷阱（连带 operation.rs 回滚失效）；枚举超长名；value_exists 错误掩盖 |
| 🟢 优化级 | 3 | DWORD/QWORD 截断改 Err；导出根头参数化；测试清理修正 |
