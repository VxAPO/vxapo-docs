# 代码审查报告：install/selector/operation.rs

- 审查日期：2026-08-11
- 被审文件：`D:\APO_Project\VxAPO\vxapo-driver\src\install\selector\operation.rs`
- 审查模块：`install/selector/operation`（安装/卸载执行 + 事务回滚）
- 参照规范：`install 模块规范.md` 5.5.2（Note 47）、`模块引用规范（无详细模块版）.md` 第十一节
- 总体评价：🚫 严重违规（事务回滚核心失效 + 规范/实现漂移）

---

## 审查摘要

- 关键风险：
  1. **事务回滚 DeleteKey 是静默空操作**：`Transaction::Drop` 中 `RegKey::open(HKLM, path)` 打开目标键后，再对同一个 key 调 `delete_sub_key(path)`（内部 `RegDeleteTreeW(handle, 全路径)`）——全路径相对该键自身不存在，错误被 `is_not_found` 吞掉（126–127 行）。安装中途失败时新建的 FxProperties 永远不会被删除。
  2. **新建的安装信息区无回滚记录**：`write_child_apo_config` 用 `RegKey::create` 创建 `HKLM\SOFTWARE\VxAPO\Child APOs\{guid}`，但从未 `tx.record(DeleteKey)`（641–643 行）→ 失败残留信息区，下次安装被误判为“非全量路径”。
  3. **E3.4 自检与规范漂移**：规范 5.5.2（v8.3 修正）要求 `verify=true` 时激活 IAudioClient 做音频管线自检；实现仍是 `CoCreateInstance` 验证 DLL 可实例化（263–301 行），且 `select.rs` 恒传 `false`，该参数形同虚设。

优点：REG_SZ 槽位写入/self-preserve 过滤/“槽位空才恢复”/capture 特例/EAPO 保留槽位等实证修正都非常到位，卸载“只碰 VxAPO”语义清晰，测试有回归价值。

---

## 可读性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 145–155 vs 217–253 | 函数注释步骤号（1–7）与代码执行顺序（Step 2→3→1→5→6→7→8）错位 | 按真实执行顺序重排注释，或显式标注“执行序 vs Note 序” |
| 163 | 参数 `verify` 文档写 CoCreateInstance，与规范 v8.3 的 IAudioClient 语义不符 | 与规范统一 |
| 676–685 | `split_hklm_path` 与 `slots.rs::split_path` 重复实现 | 合并到 `utils/guid` 或 `slots` 一处导出 |

---

## 健壮性问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 123–135 | DeleteKey 回滚空操作（见摘要 1） | 改用 `delete_tree(HKEY_LOCAL_MACHINE, path)`（本文件 393 行已示范正确用法），并加回滚单元测试 |
| 641–643 | 信息区创建未记回滚（见摘要 2） | `tx.record(RollbackAction::DeleteKey(info_key.clone()))` |
| 349 | `stop_audio_service()` 错误被 `let _ =` 忽略，若停服失败则后续删槽位必然失败且提示不准确 | 记录日志并把失败合并进后续错误上下文 |
| 391–394 | 卸载删除信息区结果被 `let _ =` 吞掉——删除失败时返回 Ok，下次安装误判非全量路径 | 失败返回 `Err` 或至少 `log::error!` |
| 304 | `restart_audio_service()?` 在 commit 之后返回 Err——注册表已写入，用户看到“安装失败”但实际已生效 | 与 audiodg 文档“best-effort 仅日志”统一：失败 `log::warn!` 不返回 Err |
| 271–283 | `CoInitializeEx` 成功后无 `CoUninitialize`（进程级 CLI 可接受，但需注释） | 补注释或 RAII 配对 |
| 284–300 | verify 未按 `install_premix/postmix` 过滤，只装单侧时也实例化两个 CLSID | 按配置过滤；并按规范改为 IAudioClient 管线自检 |
| 522–534、641 | `device_guid` 未校验即拼注册表路径 | 复用 `parse_guid_string` 校验后再拼接（与 slots.rs 同款建议） |
| 733 | `delete_other_mode_slots` 删除失败全静默，可能残留旧槽位 | 收集失败并 `log::warn!`（不阻断安装） |
| 548–552 | `ensure_fx_properties` 对同一路径打开两次 | 一次 `open` 后复用句柄判断 `version` |

---

## 安全问题

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 126–127 | 回滚删除使用“打开后再删全路径”的语义错误调用，虽不会误删，但使回滚失效（见健壮性 1） | 修正确认后删除范围仅限已记录的键路径 |
| 271–293 | `CoInitializeEx`/`CoCreateInstance` 的 SAFETY 注释较简略 | 补充“进程内 COM 状态、失败不释放指针、IUnknown 由 windows-rs 管理”前提 |
| — | 无 RT/unsafe 红线问题（全控制路径） | 无 |

---

## 可维护性问题（含依赖违规）

| 行号 | 问题描述 | 建议 |
|------|----------|------|
| 12–26 | 依赖 `slots`、`sysfx`、`object/vx_reg_props`、`object/dll_exports`、`sys/registry`、`sys/com/prelude`、`utils/error` —— 基本与规范 5.5.2 行一致；**`install/audiodg` 实际使用但未列入该行**（Note 47 Step 0a 已明文要求） | 规范侧把 `install/audiodg` 补入 operation 引用来源 |
| 174、304 | `crate::install::audiodg::{disable, restart_audio_service}` 全限定调用 | 顶部 `use` 收口 |
| 520–522 | `find_endpoint_path` 暴露为 `pub(crate)` 供 `object/apo/init.rs` 自愈调用——object 规范 7.1.8 引用清单未登记 `install/selector/operation` | 规范侧补登记（object 为胶水层，允许反向依赖 install） |
| 814–943 | `cfg(test)` 隔离 ✓；`read_slot_value_parses_eapo_reg_sz_guid` 测试复制了 GUID 解析实现，属于“实现复读”型测试 | 改为直接调用 `parse_guid_string` 断言，避免假阳性 |

### 依赖违规检查

- 本文件所在模块：`install/selector/operation.rs`
- 规范允许依赖：`install/device/slots`、`install/device/sysfx`、`install/device/format`、`sys/registry`、`object/vx_reg_props`、`object/dll_exports`、`sys/com/prelude`、`utils/error`
- 规范禁止依赖：`pipeline/`、`config/`
- 实际依赖：`install/device/slots`、`install/device/sysfx`、`install/audiodg`、`object/vx_reg_props`、`object/dll_exports`、`sys/registry`、`sys/com/prelude`、`utils/error`（含 windows COM/KS）
- 违规项：⚠️ `install/audiodg` 未在 5.5.2 行列出（Note 47/卸载流程文本已要求）→ 规范行补录即可；无 pipeline/config 越界。

---

## 优雅性建议

| 行号 | 当前写法 | 更 Rustacean 的写法 |
|------|----------|---------------------|
| 97–137 | `Transaction` 用 `Vec<RollbackAction>` 线性扫描 | 保持现状可接受（动作少）；Drop 回滚可改为 `actions.drain(..).rev()` 便于未来加日志 |
| 589–612 | `read_original_apo_guids` 两段对称 if | 抽 `fn keep_original(slot_value: SlotValue) -> Option<GUID>`（含 self-preserve 过滤） |
| 755–791 | `write_default_processmode` 闭包 `write(pid)` | 可改 `(5..=7).filter(...)` 但当前 match 更直观，保留 |
| 793–808 | 文件名清洗手写 replace 列表 | 可复用 `sanitize_filename` crate 或抽 `utils/fs_name`；当前已正确 |

---

## 优先级汇总

| 优先级 | 数量 | 说明 |
|--------|------|------|
| 🔴 阻断级 | 2 | ① DeleteKey 回滚空操作（126–127）；② 信息区创建无回滚记录（641–643）——两者共同破坏“任何步骤失败自动回滚”核心承诺 |
| 🟡 建议级 | 6 | E3.4 自检与规范漂移；信息区删除失败被吞；服务重启失败在 commit 后报错；stop 失败被忽略；GUID 未校验拼路径；audiodg 依赖未登记 |
| 🟢 优化级 | 4 | 重复 split helper 合并；双重 open 消除；verify 按配置过滤；测试改为直调 parse_guid_string |
