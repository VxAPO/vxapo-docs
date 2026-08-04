# CLI 引用规范

> **目的**：定义 VxAPO CLI（`vxapo-cli`）的规范边界、现有能力、修改路线与执行约束。
> **定位**：CLI 是**开发者工具**（intent.md「三层分离」五节），依赖 vxapo-driver（as library）经
> **安装层 API** 操作设备，不触碰 pipeline/RT；面向调试自动化与 P0 端到端验证，不面向终端用户。
> **依据**：源码实读 `D:\APO_Project\VxAPO\vxapo-cli`（v0.2.0）+ `D:\APO_Project\VxAPO\vxapo-driver`（2026-08-04）。

---

## 一、现有 CLI 现状（源码实读）

### 1.1 文件结构（vxapo-cli/src，8 文件）

```
vxapo-cli/
├── Cargo.toml          # 仅依赖 winreg = "0.52"（当前**未依赖 vxapo-driver**）
├── src/
│   ├── main.rs         # 入口：交互式主循环（枚举端点 → 选择 → 查看注册表）
│   ├── app.rs          # App 状态（endpoints 列表 + refresh）
│   ├── display.rs      # 端点列表/详情打印
│   ├── endpoint.rs     # 端点查询（独立 winreg 枚举）
│   ├── knowledge.rs    # 已知 GUID/CLSID 知识库
│   ├── probe.rs        # 探测辅助
│   ├── reg.rs          # 注册表读取辅助
│   └── regdump.rs      # 端点注册表转储（[x] 命令）
```

### 1.2 现有命令能力

| 命令 | 行为 | 现状 |
|------|------|------|
| 启动 | 枚举音频端点（winreg 遍历 MMDevices） | 有 |
| `[0-N]` 选择端点 | 进入端点详情子菜单 | 有 |
| `[x]` 查看注册表 | 转储端点注册表键 | 有 |
| `[r]` 刷新 / `[q]` 退出 | 重新枚举 / 退出 | 有 |

**缺口**：CLI **完全没有** install/uninstall/config set/preset/状态检测能力——纯交互式诊断工具。

---

## 二、driver 可复用 API（源码实读确认）

`vxapo-driver` `lib.rs` 全模块 pub（config/install/object/pipeline/sys/telemetry/utils），
CLI 依赖后可直接调用：

### 2.1 安装/卸载（install/selector/operation.rs）

```rust
pub struct InstallConfig {
    pub install_premix: bool,
    pub install_postmix: bool,
    pub install_mode: InstallMode,       // SfxEfx 默认
    pub use_original_apo_premix: bool,   // 保留前任 PreMix 为子 APO
    pub use_original_apo_postmix: bool,  // 保留前任 PostMix 为子 APO
    pub allow_silent_buffer: bool,
    pub auto_adjust: bool,               // 默认 false
}
// default_config(): SfxEfx, 双向安装, use_original 均 false, allow_silent=true, auto_adjust=false

// Note 47 完整 7 步安装（Transaction 保护，失败自动回滚）+ v8.5 全量/非全量判定
pub fn install_endpoint(device_guid: &str, device_name: &str, connection_name: &str,
                        config: &InstallConfig, verify: bool) -> Result<()>;
pub fn uninstall_endpoint(device_guid: &str) -> Result<()>;
```

> **v8.5 全量判定内建**：`install_endpoint` 开头检查 `HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}`
> 键存在性——不存在=全量备份路径、存在=非全量（失守重装槽位覆盖 childapo）。

### 2.2 设备查询（install/device/info.rs + slots.rs）

```rust
pub struct DeviceInfo {
    pub endpoint: Option<EndpointInfo>,
    pub install_mode: InstallMode,
    pub slots: [SlotValue; 5],
    pub format: Option<AudioFormat>,
    pub installed_version: String,
}
// is_installed() / can_be_upgraded() / is_disabled() / is_unplugged()

pub fn enumerate_devices() -> Result<Vec<DeviceInfo>>;   // 设备枚举唯一入口

// 槽位 + 安装信息区（install/device/slots.rs）
pub const CHILD_APO_PATH_ROOT: &str = r"HKLM\SOFTWARE\VxAPO\Child APOs";
pub enum ChildApoKind { PreMix, PostMix }
pub fn child_apo_key_exists(device_guid: &str) -> bool;                    // 全量/非全量判定依据
pub fn read_child_apo_guid(device_guid: &str, kind: ChildApoKind) -> Option<windows::core::GUID>;
pub fn read_all_slots(endpoint_key: &RegKey) -> [SlotValue; 5];
pub fn get_original_pre_mix(slots: &[SlotValue; 5], mode: InstallMode) -> String;
pub fn get_original_post_mix(slots: &[SlotValue; 5], mode: InstallMode) -> String;
```

### 2.3 audiodg（install/audiodg.rs）

```rust
pub fn ensure_can_load() -> Result<()>;   // DisableProtectedAudioDG 检查
```

### 2.4 config 层（config/parser.rs —— 仅供读回验证，不修改）

```rust
pub struct ConfigParser;   // parse_file / parse_string（spec + filter 链）
```

---

## 三、CLI 边界（intent 三层分离 + P0-7 定位）

| 层面 | CLI 可做 | CLI 不做 |
|------|----------|----------|
| install 层 API | 依赖 vxapo-driver 调用 `install_endpoint`/`uninstall_endpoint`/`enumerate_devices` | 不自行写注册表（经 driver 事务） |
| 槽位失守检测 | 经 `enumerate_devices` + `read_all_slots` 检测**安装模式槽位**（v8.5） | 提示重装 + 触发 `install_endpoint` 覆盖备份 childapo；不直接写 childApoPath |
| config 管理 | `config set` 写 `Documents\VxAPO\{GUID}\config.txt`；`config show` 读回验证 | 不解析 DSP 语义（读回一致性仅文件级） |
| pipeline/RT | **不触碰** | 不做实时音频处理 |
| 回滚 | 对 driver 改动前 snapshot（注册表/配置状态） | — |

**验证边界（P0-7）**：config show 验证「文件已写入可读回」；真正 DSP 热重载生效由 P0-4 手动听感验证；
child 委托链完整验证需真实 audiodg + 已注册 APO（P0-6 遗留，CLI 端到端联调覆盖）。

---

## 四、CLI 修改路线（分阶段）

### Phase A：依赖接入 + 诊断增强（P0-7 前置）

1. **Cargo.toml 增加 vxapo-driver 依赖**（path = `../vxapo-driver`）——CLI 由独立 winreg 枚举改为
   经 `install::device::info::enumerate_devices` 统一枚举（去除自实现 endpoint.rs 的重复逻辑）
2. **status/list 命令**：调用 `enumerate_devices` 展示设备列表 + `is_installed`/`is_disabled`/
   `installed_version`/安装模式/5 槽位值（**标注哪些槽位被接管**）
3. **槽位失守检测接入（v8.5 产品意图）**：CLI 启动 / 切换设备时，检测 `DeviceInfo.install_mode`
   的 premix/postmix 槽位值 ≠ VxAPO CLSID → 提示「需重新安装」→ 用户确认 → `install_endpoint`
   （内部先 `child_apo_key_exists` 判定全量/非全量 → 失守重装覆盖备份 childapo 最新前任）

### Phase B：核心命令（P0-7 端到端）

| 命令 | 行为 | driver API |
|------|------|-----------|
| `install -d <device>` | 安装（默认 SfxEfx + use_original=true 保留前任为子 APO + verify=true 管线自检） | `install_endpoint` |
| `uninstall -d <device>` | 卸载（含删除 childApoPath 键，v8.5） | `uninstall_endpoint` |
| `config set -d <device> -f <file>` | 写 `Documents\VxAPO\{GUID}\config.txt` | 文件写（driver 不提供 config 写——CLI 直接写文件） |
| `config show -d <device>` | 读回 config.txt 验证 | `ConfigParser::parse_file` 合法性验证 |

### Phase C：回滚 snapshot（P0-7） + 注册表转储保留

1. **回滚 snapshot**：`install`/`uninstall`/`config set` 前对 driver 改动快照（注册表 FxProperties +
   childApoPath 键 + config 文件状态）→ 失败可恢复（driver Transaction 已覆盖注册表回滚；
   CLI 补文件级 + 显式 snapshot 命令）
2. **保留现有 `[x]` 注册表转储 + 诊断**：regdump/reg 辅助保留（调试工具价值）

### Phase D：P1 扩展（预留，不改 P0-7 主链）

- `preset list/apply/set-intensity`（P1-2）；`config import` EAPO config.txt（P1-4）
- 引导至 GUI App 的交互式向导（可选）

---

## 四.5 GUID 友好名称（用户补充 2026-08-04）

CLI 的 `knowledge.rs` 需**扩充 APO CLSID → 产品名映射**，使 `list/status` 能友好显示槽位中被
EAPO/VxAPO 占用的 CLSID（当前 `SYSTEM_APO_CLSIDS` 无友好名，槽位显示只有 `{d04e05a6...}` 属性名）。

### 4.5.1 EAPO / VxAPO pre/postmix CLSID（源码确认）

| 产品 | 类型 | CLSID | 来源 |
|------|------|-------|------|
| **EAPO** | PreMix（pre_mix） | `{EACD2258-FCAC-4FF4-B36D-419E924A6D79}` | RegistryHelper.h 29（源码实读） |
| **EAPO** | PostMix（post_mix） | `{EC1CC9CE-FAED-4822-828A-82A81A6F018F}` | RegistryHelper.h 31（源码实读） |
| **VxAPO** | PreMix | `{41C34613-D391-459D-A039-72B2B15A1A1D}` | vx_reg_props.rs（v7.5 正式 GUID） |
| **VxAPO** | PostMix | `{B4A97313-ABC0-45ED-9C33-428B20D39428}` | vx_reg_props.rs（v7.5 正式 GUID） |

> **实现**：`knowledge.rs` 增加 `KNOWN_APO_CLSIDS: HashMap<&str, &str>`（CLSID → "Equalizer APO PreMix" /
> "Equalizer APO PostMix" / "VxAPO PreMix" / "VxAPO PostMix"），`friendly_name`/槽位标注优先查此表。

### 4.5.2 Premix / PostMix 具体用法（二次明确）

| 概念 | 定义 | Windows 音频链位置 | 典型槽位（SfxEfx） | VxAPO 用法 |
|------|------|-------------------|---------------------|-----------|
| **Premix（前置混合）** | 在系统**混音器之后、效果链起点**处理的 APO | 引擎先跑 Premix 再混入其他流 | **SFX(2)**（Win10+）/ LFX(0)（Legacy） | VxAPO PreMix 实例：处理**独立流/输入源**（如单应用或捕获），child 前置委托每帧一次 |
| **PostMix（后置混合）** | 在**所有流混音之后、端点输出前**处理的 APO | 引擎混完所有流后跑 PostMix | **EFX(4)**（默认）/ MFX(3)（蓝牙）/ GFX(1)（Legacy） | VxAPO PostMix 实例：处理**最终混合输出**（系统全局 EQ/效果） |

**关键区分**：
- **EAPO 语义**：`preMix` 标志经 `APOInitSystemEffects.APOInit.clsid` 判定（EqualizerAPO.cpp 122）；
  安装时 `installPostMix = !input`（渲染设备才装 PostMix，DeviceAPOInfo.cpp 254）。
- **VxAPO 对齐**：PreMix 实例（`CLSID_VXAPO_PRE_MIX`）处理混音前流；PostMix 实例（`CLSID_VXAPO_POST_MIX`）
  处理混音后最终输出；两者共用同一 config.txt（per-device），child 委托均前置。
- **CLI 显示**：`list/status` 标注槽位占用时应显示「VxAPO PreMix/PostMix」「EAPO PreMix/PostMix」，
  便于用户快速识别哪个 APO 接管了哪个槽位（配合 v8.5 槽位失守检测提示重装）。

---

## 五、修改约束（硬性）

1. **不触碰 pipeline/RT**：CLI 仅依赖 `install/` + `config/parser`（读回验证）+ `sys/`（经 driver）；
   违反 intent.md 三层分离禁止。
2. **不自行写注册表**：install/uninstall 一律走 `install_endpoint`/`uninstall_endpoint`（Transaction 保护）；
   槽位失守检测只读（`enumerate_devices` + `read_all_slots`），写动作经 driver。
3. **不重复实现**：端点枚举用 `enumerate_devices`（唯一入口，install 5.4）；废弃 CLI 自实现 winreg 枚举。
4. **config 写归 CLI**：driver 只兜底 default config（object 7.1.8）；CLI 负责 config set（intent
   「应用层写文件」）。
5. **权限**：install/uninstall 需管理员（写入 HKLM）；CLI 检测非管理员运行时应提示并拒绝操作。

---

## 六、关联

- P0-7（roadmap）：CLI 端到端验证——本规范落地后 P0-7 可进入 Implementing
- P0-6 遗留：child 委托链完整测试 + P0-4 听感验证——均靠 CLI（install/config set）端到端联调覆盖
- intent.md 五节（三层分离）/ 七节（槽位失守检测 v8.5）/ 十一（config.txt 语法）——CLI 行为上位约束