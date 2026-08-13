# VxAPO UI 设计规范 — 05 WinUI 3 与 driver 集成定调

> 状态：设计定稿（v1，2026-08-13）｜ 技术栈：Windows App SDK + C#（.NET 8）+ WinUI 3
> 交互细则顺延至 `06`；UI 图形 / 布局 / 令牌见 `01–04`。

---

## 一、总纲：文件系统解耦

- **UI（WinUI 3）唯一职责**：把用户调好的参数写入
  `C:\ProgramData\VxAPO\{device-guid}\config.toml`。
- **driver 唯一职责**：初始化或收到音频引擎重载事件时读取对应配置（热重载已具备）。
- 除安装 / 卸载 / 状态查询外，UI 与 driver **无其他耦合**：
  - 不把 APO DLL 加载进 APP 进程；
  - 不做 FFI bridge DLL、named pipe 或后台服务；
  - 调音链路不经过 CLI，纯文件写入。

---

## 二、数据流

| 场景 | 路径 |
|---|---|
| 调音 | UI 内存模型 → TOML（`01` 契约）→ 写文件 → driver 热重载 |
| 读回 | UI 自行解析 `config.toml`（TOML 模型按 `01`），driver 不回写 |
| 安装 / 卸载 | UI 提权（`runas`）调 `vxapo-cli install/uninstall --json` |
| 状态 | UI 调 `vxapo-cli list --json`（名称 / GUID / 版本 / 模式 / 槽位 / 失守） |

---

## 三、CLI 契约（v0.3.0，`--json`）

> Driver 模式子命令本就是无色文本；`--json` 是为了稳定结构化输出，供 C# 反序列化。

| 命令 | 输出 |
|---|---|
| `vxapo-cli list --json` / `status --json` | JSON 数组：`index / name / guid / installed_version / install_mode / slots{LFX..EFX} / eapo? / lost_slot?` |
| `vxapo-cli install -d <device> [--mode …] [--no-child] --json` | `{"ok":true,"device":"…","mode":"…","message":"已安装"}` |
| `vxapo-cli uninstall -d <device> --json` | `{"ok":true,"device":"…","message":"已卸载"}` |
| 任一失败 | `{"ok":false,"error":"…"}`，退出码 1 |

- **提权**：install / uninstall 需管理员（APP 以 `ProcessStartInfo.Verb = "runas"` 启动）；list / status 普通权限。
- 安装为同步事务，APP 显示步骤文案 + 等待，成功 / 失败解析 JSON 收尾。

---

## 四、WinUI 3 技术约定

- Windows App SDK + C#（.NET 8）；XAML 实现 `01–04` 的视图 / 布局 / 图形令牌（胶囊 = 决策元素、圆角 = 信息布局、品牌青 `#33CCCC / #009AA2` 点缀）。
- config.toml 写入：`File.WriteAllText`（UTF-8 无 BOM）；保存后 UI 标记「已保存 / 已应用」——driver 热重载无需确认。
- 设备列表：优先 `list --json`；如需插拔实时性，用 `Windows.Devices.Enumeration` 补端点枚举。
- 状态刷新：关键操作后刷新即可，或低频轮询（≈5s）。

---

## 五、非目标

- 不做 FFI bridge DLL、named pipe、后台服务；
- 不把 APO DLL 加载进 APP 进程；
- 不迁移 driver 逻辑进 APP。

---

## 六、后续

- `06 交互细则`（原 `05` 交互细则顺延）：拖拽成组 / 拆组、命名弹窗、菜单、通道管理、安装页交互、on→off 合并确认。
- CLI `--json` 已随本定调落地（v0.3.0）。
- WinUI 工程脚手架另立任务。
