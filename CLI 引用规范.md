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
| config 管理 | `config set` 写 `C:\ProgramData\VxAPO\{GUID}\config.toml`（v8.9 系统级根，v9.11 起 TOML）；`config show` 读回验证；`config convert` 旧 txt → TOML 一次性迁移 | 不解析 DSP 语义（读回一致性仅文件级） |
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

#### 4.4.1 命令参数定义（用户补充 2026-08-04）

**`<device>` 参数：接受两种形式，统一经 `resolve_device` 映射为 `(device_guid, device_name, connection_name)`**

| 形式 | 写法 | 说明 | 优先级 |
|------|------|------|--------|
| **GUID（规范形式，推荐）** | `{XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX}` | 稳定唯一；等于 `EndpointInfo.endpoint_guid`（注册表 `MMDevices\Audio\Render\{guid}` 键名），直接传给 `install_endpoint` 第一参数 | 优先匹配 GUID 格式（`{}` 开头） |
| **枚举序号** | 非负整数 `0..N-1` | `list/status` 输出顺序的序号（交互便捷，但随枚举顺序不稳定）；`resolve_device` 经 `enumerate_devices()[n].endpoint.endpoint_guid` 映射 | 数字命中序号 |

```
resolve_device(device_ref) -> (device_guid, device_name, connection_name)
  1. device_ref 以 '{' 开头 → 直接作 GUID（校验格式），name/connection 从 enumerate_devices 匹配（找不到则空串）
  2. device_ref 可解析数字 n → enumerate_devices()[n] → endpoint_guid / friendly_name / connection_name
  3. 均不匹配 → 报错「未知设备：<device_ref>」（附 list 提示）
```

**`<file>` 参数（仅 `config set -f`）**：**源文件**（要写入目标 config.toml 的内容来源）

- 接受**绝对路径**或**相对路径**（相对当前工作目录）；含空格的路径用引号包裹（shell 层处理）
- 文件必须存在且可读；CLI 读取内容后**原样写入** `C:\ProgramData\VxAPO\{GUID}\config.toml`
- 目标路径拼装：固定 `C:\ProgramData\VxAPO\{GUID}\config.toml`（v8.9 系统级 CONFIG_ROOT——**不用用户级
  Documents**：audiodg 是 SYSTEM 服务，它调 `documents_folder()` 拿到 SYSTEM 的 Documents，读不到
  CLI（用户进程）写入的文件，导致「改 Documents 的 config 没效果」；ProgramData 全用户共享，
  与快照目录同根）；`{GUID}` 为设备 GUID（大写花括号格式）；目录不存在则创建
- **CLI 不修改文件内容**（纯复制）；语法合法性由 driver（Lock/热重载解析）承担，CLI 只做文件级读回

#### 4.4.2 命令表

| 命令 | 行为 | driver API |
|------|------|-----------|
| `install -d <device> [--mode LfxGfx\|SfxMfx\|SfxEfx] [--no-child]` | 安装（默认 SfxEfx + use_original=true 保留前任为子 APO + verify=true 管线自检；`--no-child` 关子 APO 保留） | `install_endpoint` |
| `uninstall -d <device>` | 卸载（含删除 childApoPath 键，v8.5） | `uninstall_endpoint` |
| `config set -d <device> -f <file>` | 写 `C:\ProgramData\VxAPO\{GUID}\config.toml`（v8.9/v9.11） | 文件写（driver 不提供 config 写——CLI 直接写文件） |
| `config show -d <device>` | 读回 config.toml 内容（文件级验证） | 文件读回 |
| `config convert <old.txt> [out.toml]` | 旧 EAPO 风格 txt → config.toml（GraphicEQ/Preamp/Wide/Aural/Reverb/Maximizer/Loudness） | 文件写 |

### Phase C：快照 = 变更对比 + 基线保持（用户补充 2026-08-04）+ 注册表转储保留

**快照的真实功能 = 列出更改项目（diff）**，非纯备份。

> **范围限定（v8.8 用户指示修正）**：快照**只捕获注册表改动**——config 文件的比对**不属于 CLI 快照**，
> 由 **driver 自身**经目录监控 + filter_spec 序列对齐检测（P0-4/v7.9：「目录变更 → 重新解析 →
> spec 序列与 active_spec 逐项比较」config 6.1 + object 7.1.18，非哈希）。

1. **快照内容（`snapshot_device(guid)`）**：捕获设备**注册表**状态
   - FxProperties 键下 5 槽位值（LFX/GFX/SFX/MFX/EFX GUID）
   - childApoPath 安装信息区（`HKLM\SOFTWARE\VxAPO\Child APOs\{guid}`：PreMixChild/PostMixChild/
     AllowSilentBufferModification/DisableAutomaticAdjustment/Version）
   - FxProperties 增强设置（DisableEnhancements）
   - **不含 config 文件**（config 比对归 driver spec 序列对齐，见上注）
2. **快照持久化**：存 `%ProgramData%\VxAPO\snapshots\{guid}.json`（**首个快照 = 安装前基线**）
3. **change 列示（`snapshot diff` / `show changes` 命令）**：`diff_snapshot(baseline, current)`——
   对比当前注册表状态与基线，**红绿底色/字体**输出（CLI 支持 ANSI 时；不支持则前缀 `+`/`-`/`~`）：

   | 变更类型 | 显示 |
   |----------|------|
   | 新增（快照无 → 当前有） | 绿色 `+ 值名 = 值` |
   | 删除（快照有 → 当前无） | 红色 `- 值名 = 值` |
   | 修改（值不同） | 黄色 `~ 值名 = 旧值 → 新值` |
   | 无变化 | 灰色 `  值名 = 值`（可折叠） |

4. **基线保持语义（用户明确）**：
   - **安装前**：`snapshot_device` 建立基线（初次快照）
   - **安装后**：可 `snapshot diff` 列出本次安装产生的全部注册表更改（红绿对比，可读验证）
   - **卸载时**：对比**仍用最开始的基线**（非卸载前重照）——展示「卸载是否恢复原状/清除了什么」；
     **快照只在「下一次重新安装」时才替换**为新基线（新安装前重照）
5. **回滚**：`snapshot restore` 恢复基线状态（**仅注册表**）；driver Transaction 已保证
   失败自动回滚（注册表级）；CLI 快照补**显式恢复**（用户可手动还原）
6. **保留现有 `[x]` 注册表转储 + 诊断**：regdump/reg 辅助保留（调试工具价值）

### Phase D：P1 扩展（预留，不改 P0-7 主链）

- `preset list/apply/set-intensity`（P1-2）；EAPO config.txt 迁移由 `config convert` 承担（v9.11 已实现）
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
  处理混音后最终输出；两者共用同一 config.toml（per-device），child 委托均前置。
- **CLI 显示**：`list/status` 标注槽位占用时应显示「VxAPO PreMix/PostMix」「EAPO PreMix/PostMix」，
  便于用户快速识别哪个 APO 接管了哪个槽位（配合 v8.5 槽位失守检测提示重装）。

---

## 五、CLI 行为流（用户补充 2026-08-04——安装/验证/卸载，具体到函数、层次分明）

> **分层**：CLI 命令（参数解析/流程）→ CLI 辅助（resolve/snapshot/io）→ driver API（install 层唯一写入口）
> → driver 内部（Transaction/注册表）。行为流**只经 driver 公开 API**，不触碰 driver 内部实现。

### 5.1 安装流（`vxapo-cli install -d <device> [--mode ...] [--no-child]`）

```
main
 └─ cli::install(args)
     ├─ 1. require_admin()                                 # 非管理员 → 报错退出（约束 5）
     ├─ 2. (guid, name, conn) = resolve_device(args.device)  # 4.4.1：GUID 或枚举序号 → 三元组
     ├─ 3. config = InstallConfig::default_config()
     │      config.install_mode  = parse_mode(args.mode)     # --mode → InstallMode（缺省 SfxEfx）
     │      config.use_original_apo_premix  = !args.no_child # 默认 true：保留前任 PreMix 为子 APO
     │      config.use_original_apo_postmix = !args.no_child # 默认 true：保留前任 PostMix 为子 APO
     ├─ 4. snapshot_device(guid, replace=true)               # Phase C：**安装前建立/替换基线（只注册表）**
     │        #   FxProperties 5 槽位 + childApoPath 键 + DisableEnhancements
     │        #   持久化 %ProgramData%\VxAPO\snapshots\{guid}.json
     │        #   （config 比对不属 CLI 快照——driver 目录监控 + spec 序列对齐，见 Phase C 注）
     ├─ 5. auto_register_driver()                            # 每次安装刷新全局 APO 注册（幂等，补全 AudioEngine 键）
     ├─ 6. install_endpoint(&guid, &name, &conn, &config, true)  # driver：Note 47 七步 + v8.5 全量/非全量判定 + verify=true 管线自检
     │     └─（driver 内部，CLI 不可见）：
     │        internal: child_apo_key_exists(guid)           # → 全量备份 / 非全量（失守覆盖 childapo）路径
     │        internal: Transaction 保护（失败逆序回滚）
     │        internal: write FxProperties 槽位 + childApoPath 安装信息区 + 处理模式 GUID
     │        internal: DisableProtectedAudioDG=1            # 第三方 APO 可加载前置
     │        internal: verify=true → 激活 IAudioClient 管线自检（E3.4 v8.3：GetMixFormat + Initialize）
     │        internal: restart AudioSrv                     # 新槽位拓扑生效（EAPO 安装对齐）
     ├─ 7. 成功 → 打印「已安装 <guid>（模式 <mode>，子 APO 保留=...）」；失败 → 打印 driver Err + 建议 snapshot 恢复
     └─ 8. （可选）config set / config show 验证安装后配置（见 5.2 后续步骤）
```

### 5.2 验证流（`install` 后 → `config set` + `config show` + `status`）

```
vxapo-cli config set -d <device> -f <file>
 └─ cli::config_set(args)
     ├─ 1. (guid, ..) = resolve_device(args.device)
     ├─ 2. path = C:\ProgramData\VxAPO\{guid}\config.toml       # v8.9 系统级 CONFIG_ROOT（audiodg/SYSTEM + 用户进程共用）
     ├─ 3. fs::create_dir_all(parent) + 读源文件 args.file 内容原样写 path（纯复制，不改写）
     └─ 4. 写回后自检：`config show` 读回（文件级）；DSP 合法性由 driver 热重载承担

vxapo-cli config show -d <device>
 └─ cli::config_show(args)
     ├─ 1. resolve_device → guid
     ├─ 2. path = C:\ProgramData\VxAPO\{guid}\config.toml       # v8.9 系统级 CONFIG_ROOT
     ├─ 3. fs::read_to_string(path) → 打印内容
     └─ 4. 打印 config.toml 内容（文件级读回验证，不解析 DSP 语义）

vxapo-cli status / list
 └─ cli::status() / cli::list()
     ├─ 1. devices = enumerate_devices()                        # driver：唯一枚举入口
     └─ 2. 逐设备打印：endpoint 名 / installed_version / install_mode / 5 槽位占用（knowledge::KNOWN_APO_CLSIDS
            友好名「VxAPO PreMix/PostMix」「EAPO PreMix/PostMix」）/ is_disabled / is_unplugged
     └─ 3. 槽位失守标注（v8.5）：安装模式 premix/postmix 槽位 ≠ VxAPO CLSID → 「⚠ 已被 <友好名> 接管，需重装」
         （触发 install 重装 → install_endpoint 内建覆盖备份 childapo 最新前任）
```

### 5.3 卸载流（`vxapo-cli uninstall -d <device>`）

```
main
 └─ cli::uninstall(args)
     ├─ 1. require_admin()
     ├─ 2. (guid, ..) = resolve_device(args.device)
     ├─ 3. **不重照快照**（基线保持，用户明确）——`diff_snapshot(baseline, current)` 用**最开始的基线**
     │        # 展示「卸载是否恢复原状/清除了什么」；基线只在下次重新安装时替换（`snapshot_device(replace=true)`）
     ├─ 4. uninstall_endpoint(&guid)                            # driver：停服务 → 删除 FxProperties 槽位 VxAPO CLSID → 重启服务
     │     └─（driver 内部）停止 AudioSrv + taskkill 兜底（audiodg 句柄释放）
     │     └─（driver 内部）删除 childApoPath\{guid} 整个键（v8.5 卸载必删——再次安装回全量路径）
     │     └─（driver 内部）重启 AudioSrv（恢复输出）
     ├─ 5. 成功 → 打印「已卸载 <guid>」+ `snapshot diff`（红绿对比卸载清除项/已还原项）
     ├─ 6. 失败 → 打印 driver Err（步骤级错误 → 5.4）+ `snapshot diff` + 建议 `snapshot restore`
     └─ 7. 可选 config 清理：删除 C:\ProgramData\VxAPO\{guid}\config.toml（用户确认，v8.9）
```

### 5.4 命令状态流（用户补充 2026-08-04）

> **目标**：明确每个命令执行**需要判断什么**、命令执行把**输入窗口变成哪几种状态**、
> 每种状态**可执行的命令**、以及**状态切换逻辑**——保证每步骤**绝对严格 + 可读反馈 + 错误检验**（用户要求）。

#### 5.4.1 状态定义（CLI 设备的生命周期态）

| 状态 | 判定依据（`DeviceInfo` + 快照存在性） | 含义 |
|------|--------------------------------------|------|
| **U**nmanaged（未管理） | `!is_installed()` && 无快照 | 设备从未被 VxAPO 安装/管理 |
| **B**aselined（已基线） | `!is_installed()` && 有快照（基线） | 曾安装已卸载/仅建基线，当前槽位非 VxAPO |
| **I**nstalled（已安装） | `is_installed()` && 有快照（基线） | VxAPO 已接管槽位 + 安装前基线存在 |
| **L**ost（失守） | `is_installed()`（版本非空）但**安装模式槽位 ≠ VxAPO CLSID** | 槽位被第三方顶替（v8.5 失守） |

> 无快照 + `is_installed()` = 异常态（快照文件被删）——按 `L` 处理并提示「快照缺失，重装将重建基线」。

#### 5.4.2 命令 × 状态 矩阵（每种状态可执行的命令）

| 命令 | U | B | I | L | 前置判断（不满足 → 拒绝 + 可读错误） |
|------|---|---|---|---|--------------------------------------|
| `list` / `status` | ✅ | ✅ | ✅ | ✅ | 无（纯只读） |
| `config show` | ✅ | ✅ | ✅ | ✅ | 目标 config 存在性（不存在 → 提示「未配置」） |
| `snapshot diff` | ✅ | ✅ | ✅ | ✅ | 快照存在（无 → 「无基线，先 install」）；不存在 → 报错 |
| `install` | ✅ | ✅ | ✅ | ✅ | ① 管理员 ② resolve_device 成功 ③ `is_installed()` 时确认「重装将替换基线」 |
| `config set` | ✅* | ✅* | ✅ | ✅ | ① 目标目录可创建 ② 源文件存在可读 ③ **若非 I 态 → 警告「设备未安装，配置不会生效」仍执行**（*U/B 允许但告警） |
| `uninstall` | ❌ | ✅ | ✅ | ✅ | ① 管理员 ② 快照存在（无 → 拒绝「无基线可对比」）③ `is_installed()` 或 L 态才可卸 |
| `snapshot restore` | ❌ | ✅ | ✅ | ✅ | 快照存在（无 → 拒绝）；恢复后再 `status` 校验 |

#### 5.4.3 状态切换逻辑（命令执行后的状态转移）

```
U ── install（建立基线 + 安装）──────────► I
U ── snapshot diff（无基线）→ 拒绝（错误）→ 仍 U
B ── install（replace 基线 + 安装）──────► I
B ── uninstall（无槽位可卸）→ 拒绝 → 仍 B
I ── 槽位被第三方改写 ──────────────────► L（status 检测发现：安装模式槽位 ≠ VxAPO CLSID）
I ── uninstall（删槽位 + 删 childApoPath 键；基线保持）─► B
L ── install（失守重装：覆盖备份 childapo 最新前任 + 重装）─► I
L ── uninstall（清槽位；基线保持）──────► B
任意 ── 重新 install 且已安装 → 替换基线 + 重新安装 → I
```

**每步骤绝对严格错误检验（用户要求）**：任何命令失败不得静默——

1. driver 返回 `Err` ≠ 空 → 打印**错误码 + 错误消息 + 所属步骤**（如 `install_endpoint 步骤 4（写子 APO 配置）失败：...`）
2. 每步骤操作前**断言前置**（见 5.4.2「前置判断」列），失败 → 打印「拒绝：<原因>」并**不进入下一步**
3. 成功后**打印明确反馈**（如 `✓ 已安装 <guid>`、`✓ config 已写入（<N> 字节），语法有效`），失败打印 `✗ <错误>`
4. `snapshot diff` 每次执行后**打印统计行**：`变更：+N 新增 / -M 删除 / ~K 修改 / 0 无变化`（可读性）
5. 涉及写操作（install/uninstall/config set/snapshot restore）失败时**提示恢复路径**（快照 restore / 重试命令）

### 5.5 端到端验证索引（P0-7 DoD 对应）

```
① install → status                    # 验证 FxProperties 槽位被 VxAPO CLSID 接管 + childApoPath 键创建
② install → config set → config show  # 验证 config 写入 + 语法可读回
③ install → config set → 播放音频     # P0-4 手动听感验证（热重载生效、无爆音）
④ install（use_original=true）→ 播放  # P0-6 child 委托链端到端联调（真实 audiodg + 已注册 APO）
⑤ uninstall → status                  # 验证槽位还原/清空 + childApoPath 键删除 + 再安装回全量路径
⑥ 槽位失守模拟：手动改槽位为非 VxAPO → status 标注失守 → install 重装 → childapo 覆盖为最新前任（P0-6 ④）
```

---

## 六、修改约束（硬性）

1. **不触碰 pipeline/RT**：CLI 仅依赖 `install/` + `config/parser`（读回验证）+ `sys/`（经 driver）；
   违反 intent.md 三层分离禁止。
2. **不自行写注册表**：install/uninstall 一律走 `install_endpoint`/`uninstall_endpoint`（Transaction 保护）；
   槽位失守检测只读（`enumerate_devices` + `read_all_slots`），写动作经 driver。
3. **不重复实现**：端点枚举用 `enumerate_devices`（唯一入口，install 5.4）；废弃 CLI 自实现 winreg 枚举。
4. **config 写归 CLI**：driver 只兜底 default config（object 7.1.8）；CLI 负责 config set（intent
   「应用层写文件」）。
5. **权限**：install/uninstall 需管理员（写入 HKLM）；CLI 检测非管理员运行时应提示并拒绝操作。

---

## 七、关联

- P0-7（roadmap）：CLI 端到端验证——本规范落地后 P0-7 可进入 Implementing
- P0-6 遗留：child 委托链完整测试 + P0-4 听感验证——均靠 CLI（install/config set）端到端联调覆盖
- intent.md 五节（三层分离）/ 七节（槽位失守检测 v8.5）/ 十一（config.toml 模型）——CLI 行为上位约束
