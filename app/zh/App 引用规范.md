# App 引用规范

> **目的**：定义 VxAPO App（`vxapo-app`）的规范边界、架构、模块结构、数据流与修改路线。
> **定位**：App 是**面向终端用户**的界面层（overview「三层分离」），只做决策与写文件，
> 不做实时音频处理；与 DLL 的唯一通信通道是文件系统（config.toml / 预设 TOML）。
> **依据**：源码实读 `D:\APO_Project\VxAPO\vxapo-app`（Vite 7 + React 19 + TS 5.8 +
> framer-motion 13 + lucide-react + @radix-ui + @dnd-kit + @tauri-apps/api 2，2026-08-23）
> + `CLI 引用规范.md` / `overview/项目概览.md` / driver `配置与DSP设计.md`。
>
> **最近修订**：2026-09-12 —— 按 app/driver/cli 三仓库当前代码回填旧 GUID 残留横幅与
> 迁移/清理、轮询与缓存策略、组件/hook 清单与后端命令表。

---

## 一、App 定位与三层分离

| 层面 | App 可做 | App 不做 |
|------|----------|----------|
| **决策/呈现** | 预设选择、语义/参数视图、效果器编辑、EQ 频响预览、设备切换、导入导出、框选批量操作 | 不解释 DSP 内部实现 |
| **文件系统** | 写 `C:\ProgramData\VxAPO\{GUID}\config.toml`（经后端命令）；读写 `_global/presets/*.toml`；写回时**主动限幅** | 不写注册表（安装/卸载经 CLI/driver） |
| **driver/RT** | 依赖 CLI/driver 做设备枚举/安装/快照 | 不触碰 pipeline/RT，不做实时音频处理 |

**核心原则（与项目方案一致）**：DLL 只认 config.toml；App 负责一切决策；配置**永不回写**——
超范围参数由 App 写回时主动限幅，DLL 只在内存里 clamp。

---

## 二、现状（源码实读，2026-08-23）

### 2.1 工程骨架

```text
vxapo-app/
├── package.json            # Vite 7 + React 19 + TS 5.8 + framer-motion 13 + lucide-react 1.x
│                           # + @dnd-kit + @radix-ui（react-dialog/select/slider/switch）
│                           # + @tauri-apps/api 2 + plugin-dialog + plugin-opener
│                           # + sharp/@resvg/resvg-js（图标管线）
│                           # scripts: dev / build / build:win / icon:fix
├── index.html / vite.config.ts / tsconfig*.json
├── src-tauri/              # Tauri 2 Rust 后端
│   ├── Cargo.toml / tauri.conf.json / icons/（多尺寸 ico + png）
│   └── src/
│       ├── main.rs
│       └── lib.rs          # 18 个 Tauri commands + 提权 CLI 封装（见第八章）
└── src/
    ├── main.tsx            # 入口（挂载 App + I18nProvider）
    ├── App.tsx             # 主应用：设备/配置/效果器/预设/语义与参数视图编排
    ├── App.css / new.css   # new.css 只做 @import 汇总，样式按分区在 styles/
    ├── styles/             # theme/topbar/sidebar/cards/tabs/device/curve/dialogs/
    │                       # toast/drag/overlay-scroll/dark（dark 最后级联）
    ├── assets/             # VxAPO_icon_v4.svg 等图标
    ├── components/         # 28 个 UI 组件（见 3.3）
    ├── data/library.ts     # 预设库数据
    ├── hooks/              # 15 个 hooks（见 3.4）
    └── lib/                # api/model/toml/effects/blocks/channels/curve/rbj/...
```

**现状结论**：

- Tauri **Rust 后端**提供 18 个命令：config 读写（含指纹短路读）、设备列表、安装/卸载
  （含失败回滚）、旧 GUID 残留列表/迁移/清理/ACL 修复、进度读取、导入导出、资源管理器定位、
  语言读写、窗口显示；并启用 `opener` / `dialog` 插件。
- 前端按 `components / hooks / lib / data / styles` 拆分；业务状态集中在 `App.tsx` 与 hooks，
  组件受控。
- 已实现：设备枚举、config 读写与自动保存（300ms 去抖）、外部热更新轮询
  （2s + 内容指纹短路，窗口不可见时暂停）、安装/卸载（`--verify` 闭环 + 进度事件）、
  旧 GUID 残留横幅（迁移/清理 + 写入被拒时 ACL 自修复）、拖拽导入导出、框选批量操作、
  i18n 中英文、自绘 overlay 滚动条、语义/参数双视图与效果器语义强度映射。
- 数据模型以 `Block` / `Band` / `EffectItem` 为主（PEQ 块 + 非 PEQ 效果器）。

### 2.2 实际源码结构（2026-08-23）

```text
src/
├── App.tsx                    # 主布局 + 设备/视图/配置/框选状态编排
├── App.css / new.css          # 样式入口（new.css @import styles/*）
├── main.tsx                   # 入口
├── components/                # TopBar, Sidebar, PresetDeck, PresetView, AdvancedView,
│                              # CurvePanel, CurvePlot, CurveGrid, DeviceTabs, DevicePropsCard,
│                              # EffectCard, EffectSemanticCard, SemanticUnitCard, BandParamCard,
│                              # GainSlider, SelectionToolbar, DragCard, DragLayer,
│                              # InstallDialog, UninstallDialog, ImportDialog, SavePresetDialog,
│                              # SettingsDialog, ConfirmDialog, OverlayScrollbar,
│                              # StaleInstallBanner, Toast, VxSelect
├── data/library.ts            # 预设库（含中英文字段）
├── hooks/                     # useConfig, useDevices, useDragSort, useViewAnimation,
│                              # useMarqueeSelection, useChannelState, usePresetActions,
│                              # useCurveHover, useThrottledCompute, useTheme, useToast,
│                              # useInterval, useWindowControls, useEdgeTintLayer, useGlassRing
├── lib/
│   ├── api.ts                 # Tauri 命令封装
│   ├── model.ts               # 共享类型 + 类型守卫
│   ├── toml.ts                # buildToml / parseConfigWithTail
│   ├── effects.ts             # 效果器定义/参数/语义强度往返
│   ├── blocks.ts / accent.ts  # 块工具/组色推导与 OKLCH 配色
│   ├── channels.ts            # 声道标签
│   ├── curve.ts / rbj.ts      # 频响评估点 / RBJ 双线性变换
│   ├── normalize.ts           # 基准电平归一化规划
│   ├── dragSortTypes.ts       # 拖拽排序类型
│   ├── storage.ts             # 本地存储（自定义预设/主题/语言等）
│   ├── snap.ts                # 亚像素吸附
│   └── i18n.tsx + i18n/{core,zh,en}.ts   # 中英文国际化
└── styles/                    # 分区样式（dark.css 最后）
```

---

## 三、实际模块结构（逐文件职责）

### 3.1 根文件 / lib

| 文件 | 职责 | 约束 |
|------|------|------|
| `main.tsx` | React 挂载入口，引入 `I18nProvider` 与全局样式 | 不含业务逻辑 |
| `App.tsx` | 主布局 + 状态编排：设备选择、视图切换、配置加载/自动保存、安装/卸载流程、框选 | 只做组合与状态，组件实现下沉到 components/hooks |
| `App.css` / `new.css` | 样式入口；new.css 按固定顺序 `@import styles/*` | dark.css 最后级联 |
| `lib/model.ts` | 共享类型：`Device`、`Block`、`Band`、`EffectItem`、`PresetLibraryEntry`、`ViewMode`、`ThemeMode` 等 + 类型守卫 | 组件/面板不得各自定义业务类型 |
| `lib/api.ts` | Tauri 命令封装：`readConfig` / `writeConfig` / `listDevices` / `installDevice` / `uninstallDevice` / `rollbackInstall` / `exportConfig` 等 | 不直接操作文件系统 |
| `lib/toml.ts` | `buildToml` / `parseConfigWithTail`：生成与解析 config.toml（PEQ 块 + 非 PEQ 效果器 + tail 保留） | 与 driver `[[effects]]` 模型一致 |
| `lib/effects.ts` | 效果器定义、参数合法性、默认值、`semanticStrength` / `applySemanticStrength` | 与 driver 参数键一致 |
| `lib/blocks.ts` / `accent.ts` | Block 工具、语义单元、预设展开、组色/预设色推导（OKLCH 配色） | — |
| `lib/channels.ts` | 声道标签/名称 | — |
| `lib/curve.ts` / `rbj.ts` | 频响评估点生成 / RBJ 系数（供 CurvePlot 实时算曲线） | — |
| `lib/normalize.ts` | 归一化规划（基准电平 + 峰值补偿） | — |
| `lib/storage.ts` | 本地存储：自定义预设、主题、语言、元数据 | 带类型守卫 |
| `lib/snap.ts` | 数值吸附/亚像素取整 | — |
| `lib/i18n.tsx` + `lib/i18n/*` | 中英文国际化（`core.ts` 提供 `t()` / `setLang`） | 键统一维护在 zh/en 字典 |

### 3.2 data/

| 文件 | 职责 | 约束 |
|------|------|------|
| `data/library.ts` | 预设库（含 `group_en/name_en/desc_en` 等中英文字段） | 类型化数据，不含 UI |

### 3.3 components/（实际组件）

| 组件 | 职责 |
|------|------|
| `TopBar.tsx` | 顶栏：Logo、设置/导入/导出、视图切换（语义/参数）、窗口控制 |
| `Sidebar.tsx` | 侧边导航：预设 / 自定义 / 高级 三段 + 可拖拽分栏 |
| `PresetDeck.tsx` | 预设列表（紧凑 pill：圆点 + 组名 + 组副标题 + 已添加态） |
| `PresetView.tsx` | 语义视图：组卡/滤波器卡/效果器卡 + 强度滑块 |
| `AdvancedView.tsx` | 参数视图：PEQ 块/频段编辑、效果器参数、声道胶囊 |
| `CurvePanel.tsx` / `CurvePlot.tsx` / `CurveGrid.tsx` | 频响曲线计算、绘制与网格 |
| `DeviceTabs.tsx` / `DevicePropsCard.tsx` | 设备标签页（含调音开关）/ 设备属性卡 |
| `EffectCard.tsx` / `EffectSemanticCard.tsx` / `SemanticUnitCard.tsx` | 效果器参数卡 / 语义强度卡 / 语义单元卡 |
| `BandParamCard.tsx` / `GainSlider.tsx` | 频段参数卡与增益滑条 |
| `SelectionToolbar.tsx` | 框选批量工具栏（保存/删除/复制到声道） |
| `DragCard.tsx` / `DragLayer.tsx` | 拖拽排序卡 / 飞行副本层 |
| `OverlayScrollbar.tsx` | 自绘 overlay 滚动条（不占布局宽度、淡入淡出、跨设备存活） |
| `StaleInstallBanner.tsx` | 旧 GUID 残留横幅（当前设备命中残留时提示迁移/清理 + 迁移确认弹窗） |
| `InstallDialog.tsx` / `UninstallDialog.tsx` | 安装（`--verify` 进度闭环）/ 卸载进度 |
| `ImportDialog.tsx` / `SavePresetDialog.tsx` / `SettingsDialog.tsx` / `ConfirmDialog.tsx` | 导入 / 保存自定义预设 / 设置 / 确认 |
| `Toast.tsx` | 轻提示 |
| `VxSelect.tsx` | 统一下拉选择（Radix Select 封装） |

### 3.4 hooks/

| Hook | 职责 |
|------|------|
| `useConfig.ts` | 读取/解析/自动保存 config.toml（300ms 去抖）；2s 轮询外部热更新（内容指纹短路 + 窗口不可见暂停）；写被拒时提权修 ACL 后重试；31 段上限；按设备初始化调音开关状态 |
| `useDevices.ts` | 设备列表 + 旧 GUID 残留列表（5s 轮询，安装中暂停）、选中设备、安装/卸载、残留迁移/清理状态 |
| `useEdgeTintLayer.ts` | 外部 Canvas 环带染色层（卡片/工具栏光源采样与脏区重绘） |
| `useGlassRing.ts` | 玻璃环带几何：按实测宽高注入四角角度与顶部亮线衰减角 |
| `useDragSort.tsx` | 自定义指针级槽位拖拽引擎（避让/布局动画/飞行落位） |
| `useViewAnimation.tsx` | 视图切换动画编排（0.32s 平移 + 800ms 高度收窄 + 滚动位置恢复） |
| `useMarqueeSelection.ts` | 框选矩形与卡片命中（含视图切换残留处理） |
| `useChannelState.ts` | 通道选择状态（逐设备记忆开关与活动声道） |
| `usePresetActions.ts` | 预设应用/自定义预设读写（基于 blocks） |
| `useCurveHover.ts` | 曲线悬浮窗跟随与避让 |
| `useThrottledCompute.ts` | 重计算节流（42ms ≈ 24fps，拖动滑块时固定间隔重算 + 停止补算） |
| `useTheme.ts` | 亮/暗/跟随系统 |
| `useToast.ts` | 通知 |
| `useInterval.ts` / `useWindowControls.ts` | 可暂停固定间隔轮询 / 窗口控制 |

### 3.5 src-tauri（Rust 后端命令）

| 命令 | 职责 |
|------|------|
| `write_config` / `read_config` | 原子写 / 读 `C:\ProgramData\VxAPO\{guid}\config.toml` |
| `read_config_checked` | 带内容指纹的读：与传入 `known_revision` 相同则只回指纹、`text = null`（轮询短路） |
| `read_lang` / `write_lang` | 读写界面语言 `lang.txt`（与安装器共用） |
| `list_devices` | 调 `vxapo-cli list --json` 并反序列化为强类型 `Device[]` |
| `install_device` | 后台线程流式执行 `vxapo-cli install --verify --progress-file`，逐行 emit `install-progress`，返回 `InstallResult` |
| `uninstall_device` / `rollback_install` | 提权运行 CLI 卸载 / 安装失败兜底回滚 |
| `list_stale_installs` | 只读调用 `vxapo-cli stale list --json`，返回 `StaleInstall[]` |
| `migrate_stale_install` / `cleanup_stale_install` / `repair_stale_acl` | 提权调用 CLI `stale migrate` / `cleanup` / `fix-acl`，迁移旧 GUID 残留、清理孤儿记录、修 config ACL |
| `read_progress` | 读取提权 CLI 的进度文件 |
| `read_import_file` / `export_config` / `open_in_explorer` | 导入导出与资源管理器定位 |
| `show_main_window` | 按系统明暗设置背景色后显示主窗口（消除白屏） |

---

## 四、状态管理与数据流

### 4.1 状态来源

- **设备状态**：`useDevices` 维护 `devices`、`staleInstalls`、`selectedGuid`、`installedDevices`、
  安装/卸载目标与进度、残留迁移/清理 busy 态；设备与残留列表同一次 `Promise.allSettled`
  拉取并做浅比较，内容未变时保留旧引用避免整树重渲染。
- **配置状态**：`useConfig` 维护 `blocks`（PEQ 块）、`effects`（非 PEQ 效果器）、`tuningMap`
  （逐设备顶层 enabled）、`loaded`、`dirtyRef` 与 `tailRef`（未知第三方效果器原文保留）。
- **通道状态**：`useChannelState` 逐设备记忆通道选择器开关与活动声道。
- **UI 状态**：`App.tsx` / `useViewAnimation` / `useMarqueeSelection` 持有视图模式、框选、
  拖拽排序、对话框等 UI 状态；组件受控。

### 4.2 数据流

```text
UI 操作（增删频段/改参数/切换 enabled/应用预设/语义强度）
  → markDirty()
  → 300ms 去抖后 buildToml(blocks, enabled, effects, channelCtx) + tail
  → Tauri write_config(guid, content)   # Rust 原子写 C:\ProgramData\VxAPO\{guid}\config.toml
  → driver watcher 检测变更 → 热重载
  → useConfig 每 2s 轮询 read_config（后端按内容指纹短路；窗口不可见时暂停，恢复可见补一次），
     外部变更自动刷新 UI（编辑中跳过）

曲线预览（独立链路）：
  → useThrottledCompute(42ms) 固定间隔重算 buildEvalFreqs + curveRange（RBJ 系数缓存）
  → CurvePlot 渲染 SVG；滑块 move 只重渲染被拖卡片，保证拖动帧数
```

### 4.3 派生规则（实际实现）

- `buildToml`：输出 `version = 1` / `enabled` / `[meta]` / `[[effects]]`（type="peq" 或非 PEQ
  效果器）；PEQ 块额外写 `crossover_hz = 200` 与通道模式下的 `channels`。
- `parseConfigWithTail`：解析 PEQ 块与非 PEQ 效果器；首个未知效果器起保留为 `tail`，
  保存时原样拼回。
- `applyPreset`：检查 31 频段上限后按当前语言/声道插入预设频段。
- 通道模式：`ChannelCtx { mode, first, active }` 决定写入 `channels` 与过滤块；
  非通道模式下只写第一声道的块。
- 效果器默认参数在写 TOML 时补齐（`defaultEffectParams` + 已有 params 合并）。

### 4.4 旧 GUID 残留流（2026-09-10 新增）

```text
useDevices（挂载 + 每 5s 轮询；安装中暂停）
  → listDevices() + listStaleInstalls()（Tauri → CLI list --json / stale list --json）
  → devices / staleInstalls 分别浅比较后入库

StaleInstallBanner（仅当残留项 target_guid == 当前选中设备时渲染）
  → 命中多条时优先 matched_partial，其次按 config_mtime_ms 最新
  → [迁移]（matched_partial 显示"迁移修复"，否则显示"迁移配置"）
       → 确认弹窗（展示目标设备 / config / snapshot / 推断模式）
       → migrateStale(from, to) → Tauri migrate_stale_install → CLI stale migrate
       → refresh() 重新拉设备与残留列表
  → [清理] → 对命中项逐个 cleanupStale(guid) → CLI stale cleanup → refresh()

config 写入被拒（迁移后 ACL 继承管理员）
  → useConfig.writeConfigSafe 捕获 access denied → repairStaleAcl(guid)
       → CLI stale fix-acl（给交互用户授予 Modify）→ 重试原写入（每设备只修一次）
```

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
  kind?: PeqBandKind;          // 缺省 peaking；TOML 写为 [[effects.bands]].type
}

export interface Block {
  id?: string;                 // 客户端稳定 id，不写 TOML
  group?: string;
  name?: string;
  channel?: string;            // 通道模式下 channels 的第一个声道短名
  enabled: boolean;
  bands: Band[];
}

export interface Device {
  index: number;
  name: string;
  guid: string;
  device_id?: string | null;   // 设备实例 ID（CLI list --json）
  connection?: string | null;  // 连接名（当前后端输出空串，预留）
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

export interface PresetMetaEntry { presetId: string; accent: string; }
export type PresetMeta = Record<string, PresetMetaEntry>;

/** 非 peq 效果器（写入 config.toml 的 [[effects]]，driver 原生支持） */
export interface EffectItem {
  id?: string;                 // 客户端稳定 id，同类型多声道（如 preamp）用 id 区分
  type: string;
  enabled: boolean;
  params?: Record<string, number | string>;
  channels?: string[];         // 缺省 = 所有声道
}

export type ThemeMode = "light" | "dark" | "system";

/** 旧 GUID 残留（CLI `stale list --json` 反序列化） */
export interface StaleInstall {
  guid: string;
  device_instance_id: string;
  display_name: string;
  config_path?: string | null;
  config_mtime_ms?: number | null;
  snapshot_path?: string | null;
  snapshot_mtime_ms?: number | null;
  premix_slot?: string | null;
  postmix_slot?: string | null;
  inferred_mode: string;
  has_child_backup: boolean;
  has_sysfx_backup: boolean;
  target_guid?: string | null;
  target_name?: string | null;
  /** matched_healthy | matched_partial | unmatched */
  target_state: string;
}

/** 迁移报告（CLI `stale migrate --json`） */
export interface MigrationReport {
  success: boolean;
  target_guid: string;
  config_from?: string | null;
  snapshot_from?: string | null;
  config_migrated: boolean;
  snapshot_migrated: boolean;
  install_repaired: boolean;
  removed_guids: string[];
  warnings: string[];
}
```

> 类型定义统一收敛在 `lib/model.ts`，并带 `isPresetLibraryEntry` / `isPresetMeta` 类型守卫；
> 组件不自行声明业务实体。

## 六、效果器模型（lib/effects.ts，与 driver 参数键一致）

当前内置效果器：`preamp`、`wide`、`aural`、`reverb`、`compressor`、`loudness`。

| 效果器 | 中文名 | 参数 | 默认值 |
|--------|--------|------|--------|
| `preamp` | 基准电平 | `gain_db` | `0` |
| `wide` | 声场处理 | `gain`(高频补偿) / `air`(中置空气) / `air_side`(侧向空气) / `mix`(干湿) / `crossover_hz`(分频点 200–1000) | `0.05 / 0.3543 / 0 / 0.6 / 200` |
| `aural` | 谐波激励器 | `tune_hz` / `drive` / `odd` / `even` / `wet` / `dry` | `1760 / 1.7699 / 1.5 / 0.25 / 0.5 / 0.5` |
| `reverb` | 板式混响 | `room_size` / `decay` / `damping` / `pre_delay_ms` / `low_cut_hz` / `wet` / `dry` | `1 / 0.41 / 0.4083 / 0 / 100 / 0.27 / 0.73` |
| `compressor` | 压缩器 | `threshold_db` / `ratio` / `knee_db` / `attack_ms` / `release_ms` / `makeup_gain_db` / `wet` / `dry` | `-18 / 4 / 3 / 10 / 100 / 6 / 1 / 0` |
| `loudness` | 等响补偿 | `phon`(目标响度) / `reference_phon`(参考响度) | `80 / 80` |

语义强度往返（`semanticStrength` / `applySemanticStrength`）：

| 效果器 | 语义强度 = | 写回 |
|--------|-----------|------|
| `wide` | `air`（中置距离 = 空气吸收深度） | `air = s`（高频补偿由参数视图手动调） |
| `aural` | `wet / 0.9` | 干湿交叉淡化：`wet = 0.9s`、`dry = 1 - wet`（和 ≤ 1） |
| `reverb` | `wet / 0.9` | `wet = 0.9s`、`dry = 1 - wet`、`decay = 0.2 + 0.7s`、`damping = 0.15 + 0.63·decay`、`pre_delay_ms = clamp(s-0.3,0,0.7)·50`、`room_size = 0.85 + 0.5s` |
| `compressor` | `(ratio - 1) / 19` | `ratio = 1 + 19s`（1:1 → 20:1） |
| `loudness` | `(ref - phon) / 40` | `phon = ref - 40s` |
| `preamp` | `(gain_db + 24) / 48` | `gain_db = 48s - 24` |

> 干湿交叉淡化规则：有干湿的效果器（aural/reverb）语义写回时 `wet` 上限 0.9、
> `dry = 1 - wet`，保证干湿和 ≤ 1，避免削波。

---

## 七、已定决策（硬性）

1. **App 不触碰实时音频**：只做配置生成与设备管理，DSP 由 driver pipeline 处理。
2. **config 路径固定**：`C:\ProgramData\VxAPO\{GUID}\config.toml`（与 CLI/driver 对齐）。
3. **config 写回由 App/Rust 原子写**：临时文件 + rename；未知第三方效果器保留 `tail` 不破坏。
4. **自动保存与刷新**：300ms 去抖；外部热更新通过 2s 轮询同步 UI（带内容指纹短路，
   窗口不可见时暂停）；设备/残留列表 5s 轮询，安装进行中暂停。
5. **安装/卸载走 CLI 提权**：Rust 侧隐藏提权调用 `vxapo-cli install/uninstall`，
   安装走 `--verify` 闭环，失败有 `rollback_install` 兜底；不直接写注册表。
6. **顶层总开关已实现**：`enabled` 字段控制整链 passthrough；标签页调音开关逐设备记忆，
   初次打开即按磁盘状态显示。
7. **EQ 频响预览已实现**：`CurvePlot` + `CurveGrid`，拖动时 42ms 节流重算保帧数。
8. **多语言已实现**：`i18n` 中英文界面；预设库含中英文字段。
9. **滚动条自绘**：原生滚动条隐藏（`os-scroll`），overlay 滚动条不占布局宽度；
   只在真实用户滚动（滚轮/触摸/拖圆头）时出现，停止 1.2s 自动淡出（0.25s 过渡）；
   设备切换先淡出不硬消失。
10. **旧 GUID 残留**：由 `StaleInstallBanner` 在选中设备页提示；迁移/清理/ACL 修复一律
    经提权 CLI `stale` 子命令，App 不自行搬文件或改权限。

---

## 八、后端命令接口（Tauri，已实现）

> Rust 后端（`src-tauri/src/lib.rs`）已实现以下命令，前端经 `@tauri-apps/api` invoke 调用。

| 命令 | 输入 | 输出 | 底层 |
|------|------|------|------|
| `list_devices` | — | `Device[]` | CLI `list --json` + serde 反序列化 |
| `read_config` | guid | `String` | 读 `C:\ProgramData\VxAPO\{guid}\config.toml` |
| `read_config_checked` | guid, known_revision | `{revision, text?}` | 带内容指纹的读（未变时 text=null，供 2s 轮询短路） |
| `write_config` | guid, content | `()` | 原子写 config.toml（tmp + rename） |
| `install_device` | guid | `InstallResult` | 后台线程 `install --verify --progress-file`，emit `install-progress` |
| `uninstall_device` | guid | `String` | 提权 `uninstall -d <guid> --json` |
| `rollback_install` | guid | `String` | 安装失败后提权回滚卸载 |
| `list_stale_installs` | — | `StaleInstall[]` | CLI `stale list --json`（只读，不提权） |
| `migrate_stale_install` | from, to, config_from?, snapshot_from? | `String`（MigrationReport JSON） | 提权 `stale migrate --from --to --json` |
| `cleanup_stale_install` | guid | `String` | 提权 `stale cleanup -d <guid> --json` |
| `repair_stale_acl` | guid | `String` | 提权 `stale fix-acl -d <guid> --json` |
| `read_progress` | tag | `String` | 读取提权 CLI 进度文件 |
| `read_import_file` | path | `String` | 拖拽导入文件内容 |
| `read_lang` / `write_lang` | — / lang | `String` / `()` | 读写界面语言 `lang.txt`（与安装器共用） |
| `export_config` / `open_in_explorer` | guid/path | `()` | 导出并在资源管理器选中 |
| `show_main_window` | — | `()` | 按系统明暗设置背景色后显示并最大化主窗口 |

另启用 Tauri 插件：`tauri-plugin-opener`、`tauri-plugin-dialog`。

---

## 九、当前状态与后续路线

### 已落地

- Tauri 2 后端、设备列表、config 读写、自动保存、安装/卸载提权与验证闭环、失败回滚、
  导入导出、i18n。
- 预设库、PEQ 块编辑、频响曲线（节流重算）、通道模式、拖拽排序、框选批量操作、
  自定义预设存储、语义/参数双视图、效果器语义强度映射、自绘 overlay 滚动条。
- 旧 GUID 残留：横幅提示 + 一键迁移/清理 + 迁移后 ACL 自修复（与 driver/cli `stale` 打通）。

### 后续可扩展

- 更多非 PEQ 效果器可视化编辑（目前未知效果器保留 tail）。
- 预设强度/多预设叠加（如需要可基于 `Block` 重新引入）。
- 快照 diff/restore 的 GUI 化（当前 CLI 已支持，App 可复用）。
- 安装包/签名/自动更新等发布工程。

## 十、硬性约束

1. **不触碰实时音频**：App 及其后端不做任何 pipeline/RT 处理；违反 intent 三层分离禁止。
2. **不直接写注册表**：安装/卸载/快照一律经 CLI/driver 逻辑（管理员权限由后端命令处理）。
3. **config 语法与 driver 一致**（TOML `[[effects]]` 模型）；写文件固定
   `C:\ProgramData\VxAPO\{GUID}\config.toml`。
4. **状态单一来源**：组件受控，业务状态只存 `App.tsx` 与 hooks；组件/面板不各自持有业务状态。
5. **主动限幅规则与 driver 一致**：`[-120, +48]`（滤波深切地板 -60）、NaN/inf 拒绝；
   只写回时限幅，不做 DLL 回写。
6. **31 段上限**：`applyPreset` / `addBand` 均检查总段数，超限拒绝。
7. **残留迁移不经 App 直接改注册表/文件权限**：只调用提权 CLI `stale migrate/cleanup/fix-acl`。

---

## 十一、关联

- `overview/项目概览.md`：三层分离、产品定位与系统组成
- `CLI 引用规范.md`：后端复用的命令/设备解析/快照逻辑与硬性约束
- `driver/配置与DSP设计.md`：config.toml 模型、spec 指纹、热重载语义
- `driver/架构与模块规范.md`：DSP 能力与数值边界（限幅规则依据）
- `项目定位/VxAPO 项目完整方案.txt`：App 阶段路线（阶段 C/D）
