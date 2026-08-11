# 代码审查报告：config/watcher.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\config\watcher.rs`
- 审查模块：`config/watcher`（目录变更监控）
- 参照规范：`config 模块规范.md` 6.2（v7.8/v7.9）、`模块引用规范（无详细模块版）.md` 第十一节
- 总体评价：⚠️ 需修改（轻微）

---

## 审查摘要

- 事件驱动 + 目录级语义正确（FindFirstChangeNotificationW 不提供文件名，靠 spec 指纹幂等短路）、10ms 去重窗口、`FindNextChangeNotification` 失败自愈重建（v9.4 防自旋）、句柄生命周期（Send + Drop 兜底）安全注释完整。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 15–16 | “new 注释的模板残留”说明已写入文件头 ✓ | 无 |
| 68–71 | doc 列表缩进错乱（`///` 首行缩进异常） | rustfmt/doc 整理 |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 158–170 | `poll_registry`/`WatchEvent::RegistryChanged` 标注“保留”——VxAPO 无注册表监视 | grep 确认生产无调用则删除或移入测试 |
| 120–156 | `WAIT_FAILED` 与 `WAIT_TIMEOUT` 一律返回 false（永久停监控）——保守但无重试 | 可对 WAIT_FAILED 尝试 `recreate_notify` 一次 |
| 139 | 去重窗口用第二段 `WaitForMultipleObjects` 忽略结果 ✓ | 无 |
| 176–187 | `shutdown()` 与 Drop 双路径关闭，幂等 ✓ | 无 |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 77–88、127–129、141–142、177–183、193–203 | unsafe 块均有 SAFETY 注释 ✓ | 无 |
| 212–215 | `unsafe impl Send` 有完整论证 ✓ | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 20–25 | 依赖仅 `windows`（Foundation/FileSystem/Threading）——规范 watcher 行列 `config/error、utils` 属过列（实现更严） | 规范可精简；无越界 |
| 232–278 | 测试覆盖哈希/无效目录/空 shutdown ✓ | 无 |

### 依赖违规检查

- 本文件所在模块：`config/watcher.rs`
- 规范允许依赖：`config/error`、`utils`
- 规范禁止依赖：install/object
- 实际依赖：`windows` 相关模块
- 违规项：无 ✅（比规范更少依赖）

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 120–156 | 事件分发 match | 已清晰；可抽 `fn on_notify(&mut self)` |
| 189–209 | `recreate_notify` 与 `new` 的句柄创建重复 | 抽 `fn create_notify(watch_dir: &Path) -> HANDLE` |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 0 | 无 |
| 🟡 建议级 | 2 | poll_registry 死代码确认；WAIT_FAILED 重试 |
| 🟢 优化级 | 2 | doc 缩进；create_notify helper |
