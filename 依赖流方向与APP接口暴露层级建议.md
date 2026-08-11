# 依赖流方向与后续 APP 引用接口暴露层级建议

- 版本：v0.1（建议稿，2026-08-11）
- 依据：68/68 源文件审查（`模块引用规范/代码审查报告/`）+ `模块引用规范（无详细模块版）.md` v9.16 及四份子规范
- 状态：**v9.17 部分采纳**——三（依赖流方向：3.2 总表修订 + 3.3 铁律）已落地
  （见主规范第十一节 + `vxapo-driver/scripts/check_deps.ps1`）；四（api 薄层/
  可见性收敛/CLI 迁移）暂缓至 APP 需要时再定；五（优先级）中 registry/install
  修复随 v9.17 审查整改一并落地。

---

## 一、背景与结论先行

全项目审查发现：**分层总则（sys → pipeline → config/install → object 胶水）在代码中总体成立**，但存在三类系统性问题：

1. **规范总表滞后于实现**：`object/apo/*`、`install` 各行的允许依赖清单多处缺项，父总表、子规范、实际代码三者不一致（详见二.2 表）。
2. **可见性没有分层**：`install/device/*`、`install/selector/*`、`config/parser` 等内部实现全部 `pub`，未来 APP 将无从分辨“稳定接口”与“内部细节”。
3. **交互逻辑与纯逻辑混层**：`install/selector/select.rs` 混有 stdin/stdout 交互，APP 无法复用；`dll_exports::register_apo_with_path` 这种正确入口却藏在 COM 导出文件里。

建议分两步收敛：
- **短中期（1–2 个版本）**：修订规范总表 + 收敛可见性 + 新增 `api`（或 `app`）薄转发层，作为 APP/CLI 唯一引用面。
- **长期（可选）**：拆 crate（`driver-core` rlib + `driver-api` rlib + `vxapo` cdylib），从编译期强制分层。

---

## 二、当前依赖流方向盘点

### 2.1 规范声明的方向

```
utils/          （零内部依赖，仅 core / windows-core / 第三方纯工具）
sys/            （只依赖 windows-rs，不知道任何业务）
pipeline/       （依赖 sys + utils + 自身；不知道 install/config/object）
config/         （依赖 sys + utils + pipeline/dsp/{model,filter,factory}；不知道具体 Filter）
install/        （依赖 sys + utils + object/vx_reg_props；不知道 pipeline/config）
object/         （胶水层，依赖所有模块；被 Windows 直接调用）
telemetry/      （依赖 pipeline/realtime/ring）
```

### 2.2 实际依赖与规范不一致点（审查汇总）

| 文件 | 父总表允许 | 实际额外依赖 | 子规范是否授权 | 建议动作 |
|------|-----------|--------------|----------------|----------|
| `install/selector/operation.rs` | 未列 `install/audiodg` | `install/audiodg::{disable,restart,stop}` | Note 47 Step 0a 明文要求 | **补录总表** |
| `object/apo/process.rs` | 未列 `install/audiodg`、`object/vx_reg_props`、`sys/audio_defs`、`sys/com/prelude`、`sys/com/apo_interfaces` | 全部使用 | object 7.1 聚合清单大部分授权 | **补录总表**（process 行） |
| `object/apo/init.rs` | 未列 `install/selector/operation`、`install/device/sysfx` | 自愈调用 | v9.4 设计明文要求 | **补录总表** |
| `object/apo/config.rs` | 未列 `config/watcher`、`pipeline/dsp/transition`、`sys/com/*`、`super/inner` | 全部使用 | 无子规范行 | **补录/新增行** |
| `object/apo/inner.rs`、`aggregate.rs` | 无独立行 | 被各子模块共享 | 无 | **新增行或聚合授权** |
| `object/factory.rs` | 列 `object/apo.rs` 入口 | 用 `object/apo/aggregate` | 无 | **补录 aggregate** |
| `object/vx_reg_props.rs` | 列 `sys/com/prelude`、`apo_types` | 用 `sys/com/apo_interfaces::IID_IAPO` | 无 | **补录 apo_interfaces** |
| `install/device/endpoint.rs` | 列 `sys/registry`、`utils/error` | 用 `sys/com/prelude`、`utils/guid` | 模块级允许 | **补录** |
| `install/device/slots.rs` | 与实现一致 | — | — | 无（样板） |
| `install/device/format.rs` | 与实现一致 | — | — | 无（样板） |
| `pipeline/format.rs` | 列 `apo_interfaces`、`apo_types` | 用 `sys/com/prelude`、`sys/audio_defs`、`utils/vx_error` | 模块级允许 | **补录** |
| `pipeline/dsp/fir.rs`、`peq_hybrid.rs` | 未列第三方 | 用 `rustfft` | 无 | **登记第三方依赖** |
| `utils/vx_error.rs` | 列 `windows-core` | 用 `thiserror` | 无 | **登记第三方依赖** |
| `telemetry/logger.rs` | 与实现一致 | — | — | 无（样板） |

> 结论：**没有真正的反向依赖违规**（即没有 config 依赖 install、pipeline 依赖 config 之类）；全部问题都是“总表行过窄/缺行”。治理重点应放在表格维护机制，而非重构代码。

### 2.3 守得好的禁止链（应保持并加 CI 保护）

- `config/parser.rs` 未触碰任何具体 Filter 类型（v9.11 静态工厂分派）✅
- `pipeline/chain.rs` 未依赖 `dsp/transition.rs` ✅
- `pipeline/context.rs` 与 `pipeline/dsp/filter.rs` 互不引用 ✅
- `utils/*` 零内部依赖 ✅
- `sys/com/*` 纯重导出、无业务 ✅

---

## 三、依赖流方向建议

### 3.1 单一事实源

**以 `模块引用规范（无详细模块版）.md` 第十一节「引用约束总表」为唯一基线**，子规范（`object 模块规范.md` 等）只写职责与设计，不再各自维护引用清单（或反过来：子规范为准、总表自动生成——二选一，避免双源漂移）。

### 3.2 总表修订清单（可直接执行）

1. `install/selector/operation.rs` 行：补 `install/audiodg`。
2. `object/apo/*` 补独立行：
   - `object/apo/inner.rs`：`pipeline/{chain,context,dsp/filter,dsp/transition}`、`sys/audio_defs`
   - `object/apo/aggregate.rs`：`object/apo`（入口）、`sys/com/prelude`、`sys/com/apo_interfaces`（IID）
   - `object/apo/config.rs`：`config/{parser,watcher}`、`pipeline/{chain,dsp/transition}`、`sys/com/{apo_types,prelude}`、`super/inner`
   - `object/apo/init.rs`：补 `install/device/{slots,sysfx}`、`install/selector/operation`（find_endpoint_path）
   - `object/apo/process.rs`：补 `install/audiodg`、`object/vx_reg_props`、`sys/audio_defs`、`sys/com/{apo_interfaces,prelude}`
3. `object/factory.rs`：补 `object/apo/aggregate`。
4. `object/vx_reg_props.rs`：补 `sys/com/apo_interfaces`（仅 `IID_IAPO`）。
5. `install/device/endpoint.rs`：补 `sys/com/prelude`、`utils/guid`。
6. `pipeline/format.rs`：补 `sys/com/prelude`、`sys/audio_defs`、`utils/vx_error`。
7. 第三方依赖登记：`rustfft`（pipeline/dsp）、`thiserror`（utils/vx_error）、`once_cell`（全局）、`serde/toml`（config）。
8. 总表补一条总注：`pipeline/dsp/*.rs` 具体滤波器允许依赖 `dsp/math`、`dsp/fir` 等兄弟模块（现行“允许清单”未覆盖此类内部协作）。

### 3.3 铁律（写入规范并做自动校验）

| # | 铁律 | 现状 | 校验方式 |
|---|------|------|----------|
| D1 | `sys/` 除 windows-rs/core 外零依赖 | ✅ | 脚本 |
| D2 | `utils/` 零内部依赖 | ✅ | 脚本 |
| D3 | `pipeline/` 不得依赖 `config/install/object` | ✅ | 脚本 |
| D4 | `config/` 不得依赖 `install/object/pipeline/{process,chain,context}` 及具体 Filter | ✅ | 脚本 |
| D5 | `install/` 不得依赖 `pipeline/config`（除 `object/vx_reg_props`） | ✅ | 脚本 |
| D6 | `object/` 是唯一允许依赖所有模块的胶水层；但 `object/apo/*` 子模块禁止互相回调（聚合壳除外） | ⚠️ 需细化 | 人工 + 脚本 |
| D7 | `telemetry/` 只依赖 `pipeline/realtime/ring` | ✅ | 脚本 |
| D8 | RT 路径类型（`Filter/Chain/DspContext/RealtimeContext`）不得出现在公共 API 面 | 待落地 | 编译期 + review |

校验工具建议：先在 `cargo test` 前跑一个轻量脚本（解析 `use crate::...` 与 `crate::...` 全限定引用，按模块白名单断言），后续可换成 `cargo module-graph` 或 rustdoc 内部 lint。此脚本本身也按三文件联动登记。

---

## 四、后续 APP 引用接口暴露层级建议

### 4.1 目标形态

```
┌─────────────────────────────────────────────┐
│  APP / CLI（未来）                           │
│  只允许引用 L1（driver-api）                  │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│ L1  driver-api（薄转发 + DTO）                │
│  install / config / registry / 版本 / 自检    │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│ L0  现有内部模块（可见性收敛为 pub(crate)）     │
│  sys / pipeline / config / install / object  │
│  / telemetry / utils                         │
└─────────────────────────────────────────────┘
```

### 4.2 新增 `api` 模块（推荐放 driver 内，命名 `api`，避免与 `app` 应用名混淆）

职责：**只做转发与 DTO 转换，不含业务逻辑**。所有内部模块改为 `pub(crate)`，`api` 是唯一 `pub` 出口。

建议接口清单（签名级草案）：

| 接口 | 转发目标 | 说明 |
|------|----------|------|
| `api::list_devices() -> Result<Vec<DeviceSummary>>` | `install/device/info::enumerate_devices` | 过滤 Active；**默认设备判定留给 APP 用 COM**（E3.1） |
| `api::install(guid, req: InstallRequest, verify: bool) -> Result<InstallReport>` | `install/selector/operation::install_endpoint` | `InstallRequest` 与 `InstallConfig` 字段一一对应 |
| `api::uninstall(guid) -> Result<()>` | `operation::uninstall_endpoint` | — |
| `api::detect_mode(guid) -> InstallMode` | `device/info::detect_mode_for_guid` | — |
| `api::query_install_state(guid) -> InstallState` | `device/info::query_device_info` | 只读汇总 |
| `api::validate_config(path, ctx?) -> Result<ConfigReport>` | `config/parser::parse_file` | `ctx` 用默认 48k/2ch 或调用方显式传入 `ConfigContext`（DTO） |
| `api::read_config(path) -> Result<String>` | `config/parser::read_config_file` | 供 APP 预览/编辑 |
| `api::register_apo(dll_path) -> Result<()>` | `object/dll_exports::register_apo_with_path` | 从 COM 文件提升为公共 API |
| `api::ensure_third_party_allowed() -> Result<bool>` | `install/audiodg::ensure_can_load` | — |
| `api::version() -> &'static str` | 常量 | 与 changelog 联动 |

DTO 建议（均 `#[non_exhaustive]` + `Debug/Clone/PartialEq`）：
- `DeviceSummary { guid, friendly_name, state, flow, install_state, mode }`
- `InstallRequest { install_premix, install_postmix, mode, use_original_apo_premix, use_original_apo_postmix, allow_silent_buffer, auto_adjust }`
- `InstallReport { ok, message, changed_slots, sysfx_takeover_count }`
- `ConfigReport { effects: usize, peq_bands: usize, spec: Vec<String>, warnings }`

### 4.3 明确不暴露（红线）

| 禁止暴露 | 原因 |
|----------|------|
| `Filter` / `Chain` / `DspContext` / `RealtimeContext` | RT 内部类型，生命周期与引擎绑定 |
| `ApoObject` / `aggregate::create_aggregate` / `ChildApo` | COM 运行时对象，只存在于 audiodg 进程；APP 不得实例化 |
| `ApoObjectInner` / 任何 `Mutex<...>` 字段 | 内部可变状态 |
| `stdin/stdout` 交互（`select.rs` 的 `print_device_list`/`prompt_user`） | 属于 CLI 宿主，移出 driver |
| `telemetry/logger` 的 ring 内部 | APP 需要日志时走 L1 `api::drain_logs()` 包装 |
| 注册表路径常量与直接 `RegKey` | APP 一律经 L1，防路径漂移 |

### 4.4 迁移步骤

1. `install/selector/select.rs` 的交互函数移入未来 CLI 宿主（driver 只留 `list_devices` 纯逻辑）。
2. 内部模块可见性收敛：`pub mod` → `pub(crate) mod`（`lib.rs` 顶层模块声明同步改），一次一个模块、跑测试确认。
3. 新增 `api.rs`（或 `api/` 目录）并按 4.2 清单实现薄转发；补 `api` 的单元测试（DTO 转换 + 转发桩）。
4. 在 `Cargo.toml` 增加 `[[bin]]` 或独立 `cli` crate 承接交互逻辑（若当前 CLI 在同仓库，从 select.rs 迁移）。
5. 规范侧：总表新增 `api` 行（允许依赖：全部内部模块；禁止依赖：无——但禁止导出 RT 类型），roadmap/changelog 三文件联动登记。

---

## 五、落地优先级

| 优先级 | 动作 | 影响面 |
|--------|------|--------|
| P0 | 按 3.2 清单修订总表（纯文档，无代码风险） | 规范 |
| P0 | `sys/registry.rs` 修复 `.reg` 导出格式 + `delete_sub_key` 语义（连带 install 事务回滚修复） | sys/install |
| P1 | 新增依赖校验脚本（D1–D7 白名单） | 工程化 |
| P1 | 内部模块可见性收敛 + `api` 薄层 | 全模块 |
| P2 | 交互逻辑迁出 driver、telemetry 接线（log + panic hook） | CLI/telemetry |
| P3 | 拆 crate（core/api/cdylib） | 长期 |

---

## 六、预期收益

- APP 得到一个**稳定、无 RT 泄漏、无 COM 运行时依赖**的引用面，版本化友好。
- 依赖治理从“文档对照”变为“脚本断言”，杜绝总表再次滞后。
- 交互/运行时/安装三件事物理解耦：CLI 管交互、audiodg 管运行时、driver-api 管安装与配置。
- 为未来“APP 需监听配置变更/读取安装状态”提供明确入口（`api::validate_config`、`api::query_install_state`），不需要向 APP 开放 watcher 或注册表句柄。
