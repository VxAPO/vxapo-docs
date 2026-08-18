# App 引用规范

> **目的**：定义 VxAPO App（`vxapo-app`）的规范边界、架构草案、模块结构、数据流与修改路线。
> **定位**：App 是**面向终端用户**的界面层（overview「三层分离」），只做决策与写文件，
> 不做实时音频处理；与 DLL 的唯一通信通道是文件系统（config.toml / 预设 TOML）。
> **依据**：源码实读 `D:\APO_Project\VxAPO\vxapo-app`（Vite 7 + React 19 + TS + Tailwind 4 +
> framer-motion + lucide-react + @tauri-apps/api 2，2026-08-13）+ 架构草案（src/ 目录树）
> + `CLI 引用规范.md` / `overview/项目概览.md` / driver `配置与DSP设计.md`。

---

## 一、App 定位与三层分离

| 层面 | App 可做 | App 不做 |
|------|----------|----------|
| **决策/呈现** | 预设选择、维度调节、高级参数编辑、EQ 频响预览、设备切换、继承复制、导入导出 | 不解释 DSP 内部实现 |
| **文件系统** | 写 `C:\ProgramData\VxAPO\{GUID}\config.toml`（经后端命令）；读写 `_global/presets/*.toml`；写回时**主动限幅** | 不写注册表（安装/卸载经 CLI/driver） |
| **driver/RT** | 依赖 driver 作为 library（或复用 CLI 逻辑）做设备枚举/安装/快照 | 不触碰 pipeline/RT，不做实时音频处理 |
| **响度补偿** | 只提供开关（写 `Loudness: on\|off`，默认 on） | **不做可视化**（已定决策） |

**核心原则（与项目方案一致）**：DLL 只认 config.toml；App 负责一切决策；配置**永不回写**——
超范围参数由 App 写回时主动限幅，DLL 只在内存里 clamp。

---

## 二、现状（源码实读，2026-08-13）

### 2.1 工程骨架

```text
vxapo-app/
├── package.json            # Vite 7 + React 19 + TS 5.8 + Tailwind 4 + framer-motion 13 + lucide-react
│                           # + @dnd-kit + @radix-ui + @tauri-apps/api 2 + @tauri-apps/cli 2
├── index.html / vite.config.ts / tsconfig*.json
├── src-tauri/              # Tauri 2 Rust 后端（已初始化）
│   ├── Cargo.toml
│   ├── tauri.conf.json
│   └── src/
│       ├── main.rs
│       └── lib.rs          # Tauri commands：config 读写、设备列表、安装/卸载、导入导出等
└── src/
    ├── main.tsx            # 入口（挂载 App + I18nProvider）
    ├── App.tsx             # 主应用：设备/配置/效果器/预设/高级视图
    ├── App.css / new.css   # 样式
    ├── assets/             # 图标
    ├── components/         # 25+ UI 组件（TopBar/Sidebar/CurvePlot/AdvancedView/...）
    ├── data/library.ts     # 预设库数据
    ├── hooks/              # useConfig/useDevices/useDragSort/useTheme/useToast/...
    └── lib/                # api/model/toml/effects/blocks/channels/storage/i18n/...
```

**现状结论**：

- Tauri **Rust 后端已初始化**，提供 `write_config`、`read_config`、`list_devices`、
  `install_device`、`uninstall_device`、`read_progress`、`export_config` 等命令。
- `App.tsx` 已拆分：`src/components/*`、`src/hooks/*`、`src/lib/*`、`src/data/*` 均已落地。
- 已接入设备枚举（CLI `list --json`）、config.toml 读写、自动保存（300ms 去抖）、
  安装/卸载（提权 CLI）、拖拽导入导出、i18n 中英文界面。
- 当前数据模型以 `Block`/`Band`/`EffectItem` 为主（PEQ 块 + 非 PEQ 效果器），
  不再是草案中的 `DimensionMapping/Dimension/Filter/Preset` 静态示例。

### 2.2 实际源码结构（2026-08-13）

```text
src/
├── App.tsx                    # 主布局 + 设备/视图/配置状态编排
├── App.css / new.css          # 样式
├── main.tsx                   # 入口
├── components/                # TopBar, Sidebar, PresetView, AdvancedView, CurvePanel,
│                              # DeviceTabs, InstallDialog, UninstallDialog, ImportDialog, ...
├── data/library.ts            # 预设库（含中英文字段）
├── hooks/                     # useConfig, useDevices, useDragSort, useTheme, useToast, ...
└── lib/
    ├── api.ts                 # Tauri 命令封装
    ├── model.ts               # Device/Block/Band/EffectItem/PresetLibraryEntry 类型
    ├── toml.ts                # buildToml / parseConfigWithTail
    ├── effects.ts             # 效果器参数默认值/合法性
    ├── blocks.ts              # 块工具/语义单元
    ├── channels.ts            # 声道标签
    ├── storage.ts             # 本地存储（自定义预设/主题等）
    ├── snap.ts                # 吸附/数值规整
    └── i18n.tsx               # 中英文国际化

src-tauri/src/lib.rs           # Tauri commands + 提权 CLI 封装
```

---

## 三、实际模块结构（逐文件职责）

### 3.1 根文件 / lib

| 文件 | 职责 | 约束 |
|------|------|------|
| `main.tsx` | React 挂载入口，引入 `I18nProvider` 与全局样式 | 不含业务逻辑 |
| `App.tsx` | 主布局 + 状态编排：设备选择、视图切换、配置加载/自动保存、安装/卸载流程 | 只做组合与状态，组件实现下沉到 components/hooks |
| `App.css` / `new.css` | 全局样式 | — |
| `lib/model.ts` | 共享类型：`Device`、`Block`、`Band`、`EffectItem`、`PresetLibraryEntry`、`ViewMode` 等 | 组件/面板不得各自定义业务类型 |
| `lib/api.ts` | Tauri 命令封装：`readConfig` / `writeConfig` / `listDevices` / `installDevice` / `uninstallDevice` / `exportConfig` 等 | 不直接操作文件系统 |
| `lib/toml.ts` | `buildToml` / `parseConfigWithTail`：生成与解析 config.toml（PEQ 块 + 非 PEQ 效果器 + tail 保留） | 与 driver `[[effects]]` 模型一致 |
| `lib/effects.ts` | 效果器类型/默认参数/合法性 | — |
| `lib/blocks.ts` | Block 工具：稳定 id、语义单元、预设展开 | — |
| `lib/channels.ts` | 声道标签/名称 | — |
| `lib/storage.ts` | 本地存储：自定义预设、主题、元数据 | — |
| `lib/snap.ts` | 数值吸附/规整 | — |
| `lib/i18n.tsx` | 中英文国际化 | — |

### 3.2 data/

| 文件 | 职责 | 约束 |
|------|------|------|
| `data/library.ts` | 预设库（含 `group_en/name_en/desc_en` 等中英文字段） | 类型化数据，不含 UI |

### 3.3 components/（实际组件）

| 组件 | 职责 |
|------|------|
| `TopBar.tsx` | 顶栏：窗口控制、主题、设置入口 |
| `Sidebar.tsx` | 侧边导航：预设 / 自定义 / 高级 |
| `PresetView.tsx` | 预设选择视图：预设卡片、分组、应用 |
| `AdvancedView.tsx` | 高级视图：PEQ 块/频段编辑、效果器、曲线 |
| `CurvePanel.tsx` / `CurvePlot.tsx` | 频响曲线计算与绘制 |
| `DeviceTabs.tsx` / `DevicePropsCard.tsx` | 设备列表/属性展示 |
| `EffectCard.tsx` / `EffectSemanticCard.tsx` / `SemanticUnitCard.tsx` | 效果器卡片与语义单元 |
| `BandParamCard.tsx` / `GainSlider.tsx` | 频段参数与增益滑条 |
| `SelectionToolbar.tsx` | 批量选择工具栏 |
| `DragCard.tsx` / `DragLayer.tsx` | 拖拽排序/拖拽层 |
| `InstallDialog.tsx` / `UninstallDialog.tsx` | 安装/卸载确认与进度 |
| `ImportDialog.tsx` / `SavePresetDialog.tsx` / `SettingsDialog.tsx` / `ConfirmDialog.tsx` | 导入/保存/设置/确认 |
| `Toast.tsx` | 轻提示 |
| `VxSelect.tsx` | 统一下拉选择 |

### 3.4 hooks/

| Hook | 职责 |
|------|------|
| `useConfig.ts` | 读取/解析/自动保存 config.toml（300ms 去抖），轮询外部热更新 |
| `useDevices.ts` | 设备列表、选中设备、安装/卸载状态 |
| `useDragSort.tsx` | 拖拽排序 |
| `useTheme.ts` | 亮/暗/跟随系统 |
| `useToast.ts` | 通知 |
| `useInterval.ts` / `useWindowControls.ts` | 定时器 / 窗口控制 |

### 3.5 src-tauri（Rust 后端命令）

| 命令 | 职责 |
|------|------|
| `write_config` / `read_config` | 原子写 / 读 `C:\ProgramData\VxAPO\{guid}\config.toml` |
| `list_devices` | 调 `vxapo-cli list --json` 并反序列化为强类型 `Device[]` |
| `install_device` / `uninstall_device` | 提权运行 CLI 安装/卸载，返回进度 |
| `read_progress` | 读取提权 CLI 的进度文件 |
| `read_import_file` / `export_config` / `open_in_explorer` | 导入导出与资源管理器定位 |
| `show_main_window` | 页面渲染完成后显示主窗口 |

## 四、状态管理与数据流

### 4.1 状态来源

- **设备状态**：`useDevices` 维护 `devices`、`selectedGuid`、`installedDevices`、安装/卸载目标与进度。
- **配置状态**：`useConfig` 维护 `blocks`（PEQ 块）、`effects`（非 PEQ 效果器）、`tuningMap`（顶层 enabled）、
  `loaded`、`dirtyRef` 与 `tailRef`（未知第三方效果器原文保留）。
- **UI 状态**：`App.tsx` 持有视图模式、通道模式、拖拽排序、对话框等 UI 状态；组件受控。

### 4.2 数据流

```text
UI 操作（增删频段/改参数/切换 enabled/应用预设）
  → markDirty()
  → 300ms 去抖后 buildToml(blocks, enabled, effects, channelCtx) + tail
  → Tauri write_config(guid, content)  # Rust 原子写 C:\ProgramData\VxAPO\{guid}\config.toml
  → driver watcher 检测变更 → 热重载
  → useConfig 每 2s 轮询 read_config，外部变更自动刷新 UI（编辑中跳过）
```

### 4.3 派生规则（实际实现）

- `buildToml`：输出 `version = 1` / `enabled` / `[meta]` / `[[effects]]`（type="peq" 或非 PEQ 效果器）。
- `parseConfigWithTail`：解析 PEQ 块与非 PEQ 效果器；首个未知效果器起保留为 `tail`，保存时原样拼回。
- `applyPreset`：检查 31 频段上限后按当前语言/声道插入预设频段。
- 通道模式：`ChannelCtx { mode, first, active }` 决定写入 `channels` 与过滤块。

---

## 五、类型规范（lib/model.ts）

实际类型以 `src/lib/model.ts` 为准：

```ts
export type ViewMode = "preset" | "advanced";
export type SideSection = "preset" | "custom" | "advanced";
export type PeqBandKind = "peaking" | "low_shelf" | "high_shelf" | "low_pass" | "high_pass";

export interface Band {
  fc: number;
  gain_db: number;
  q: number;
  kind?: PeqBandKind;
}

export interface Block {
  id?: string;
  group?: string;
  name?: string;
  channel?: string;
  enabled: boolean;
  bands: Band[];
}

export interface Device {
  index: number;
  name: string;
  guid: string;
  installed_version: string;
  install_mode: string;
  slots: Record<string, string | null>;
  sample_rate?: number | null;
  channels?: number | null;
  bit_depth?: number | null;
  kind?: "playback" | "capture" | null;
  volume?: number | null;
  eapo?: string;
  lost_slot?: string;
}

export interface EffectItem {
  id?: string;
  type: string;
  enabled: boolean;
  params?: Record<string, number | string>;
  channels?: string[];
}

export interface PresetLibraryEntry {
  id: string;
  group: string;
  name: string;
  desc: string;
  group_en?: string;
  name_en?: string;
  desc_en?: string;
  color?: string;
  bands: (Band & { name?: string; name_en?: string })[];
}
```

> 类型定义统一收敛在 `lib/model.ts`，组件不自行声明业务实体。

## 六、已定决策（硬性）

1. **App 不触碰实时音频**：只做配置生成与设备管理，DSP 由 driver pipeline 处理。
2. **config 路径固定**：`C:\ProgramData\VxAPO\{GUID}\config.toml`（与 CLI/driver 对齐）。
3. **config 写回由 App/Rust 原子写**：临时文件 + rename；未知第三方效果器保留 `tail` 不破坏。
4. **自动保存**：300ms 去抖；外部热更新通过 2s 轮询同步 UI。
5. **安装/卸载走 CLI 提权**：Rust 侧隐藏提权调用 `vxapo-cli install/uninstall`，不直接写注册表。
6. **顶层总开关已实现**：`enabled` 字段控制整链 passthrough（driver v9.17+）。
7. **EQ 频响预览已实现**：`CurvePlot` 实时计算 PEQ 曲线。
8. **多语言已实现**：`i18n` 中英文界面；预设库含中英文字段。

---

## 七、后端命令接口（Tauri，已实现）

> Rust 后端（`src-tauri/src/lib.rs`）已实现以下命令，前端经 `@tauri-apps/api` invoke 调用。

| 命令 | 输入 | 输出 | 底层 |
|------|------|------|------|
| `list_devices` | — | `Device[]` | CLI `list --json` + serde 反序列化 |
| `read_config` | guid | `String` | 读 `C:\ProgramData\VxAPO\{guid}\config.toml` |
| `write_config` | guid, content | `()` | 原子写 config.toml |
| `install_device` / `uninstall_device` | guid | `String`（进度/结果） | 提权运行 CLI install/uninstall |
| `read_progress` | tag | `String` | 读取提权 CLI 进度文件 |
| `read_import_file` | path | `String` | 拖拽导入文件内容 |
| `export_config` / `open_in_explorer` | guid/path | `()` | 导出并在资源管理器选中 |
| `show_main_window` | — | `()` | 前端渲染完成后显示主窗口 |

---

## 八、当前状态与后续路线

### 已落地

- Tauri 2 后端、设备列表、config 读写、自动保存、安装/卸载提权、导入导出、i18n。
- 预设库、PEQ 块编辑、频响曲线、通道模式、拖拽排序、自定义预设存储。

### 后续可扩展

- 更多非 PEQ 效果器可视化编辑（目前保留未知效果器 tail）。
- 预设强度/多预设叠加（如需要可基于 `Block` 重新引入）。
- 快照 diff/restore 的 GUI 化（当前 CLI 已支持，App 可复用）。
- 安装包/签名/自动更新等发布工程。

## 九、硬性约束

1. **不触碰实时音频**：App 及其后端不做任何 pipeline/RT 处理；违反 intent 三层分离禁止。
2. **不直接写注册表**：安装/卸载/快照一律经 CLI/driver 逻辑（管理员权限由后端命令处理）。
3. **config 语法与 driver 一致**（TOML `[[effects]]` 模型，v9.11）；写文件固定
   `C:\ProgramData\VxAPO\{GUID}\config.toml`。
4. **不实现响度补偿可视化**（已定决策）；只提供开关。
5. **状态单一来源**：组件受控，业务状态只存 `App.tsx`；组件/面板不各自持有业务状态。
6. **主动限幅规则与 driver 一致**：`[-120, +48]`（滤波深切地板 -60）、NaN/inf 拒绝；只写回时限幅，不做 DLL 回写。

---

## 十、关联

- `overview/项目概览.md`：三层分离、产品定位与系统组成
- `CLI 引用规范.md`：后端复用的命令/设备解析/快照逻辑与硬性约束
- `driver/配置与DSP设计.md`：config.toml 模型、spec 指纹、热重载语义
- `driver/架构与模块规范.md`：DSP 能力与数值边界（限幅规则依据）
- `项目定位/VxAPO 项目完整方案.txt`：App 阶段路线（阶段 C/D）
