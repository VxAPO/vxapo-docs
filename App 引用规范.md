# App 引用规范

> **目的**：定义 VxAPO App（`vxapo-app`）的规范边界、架构草案、模块结构、数据流与修改路线。
> **定位**：App 是**面向终端用户**的界面层（intent.md「三层分离」），只做决策与写文件，
> 不做实时音频处理；与 DLL 的唯一通信通道是文件系统（config.toml / 预设 TOML）。
> **依据**：源码实读 `D:\APO_Project\VxAPO\vxapo-app`（Vite 7 + React 19 + TS + Tailwind 4 +
> framer-motion + lucide-react + @tauri-apps/api 2，2026-08-06）+ 用户提供的架构草案（src/ 目录树）
> + `CLI 引用规范.md` / `intent.md` / driver `config 模块规范.md`。

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

## 二、现状（源码实读，2026-08-06）

### 2.1 工程骨架

```text
vxapo-app/
├── package.json            # Vite 7 + React 19 + TS 5.8 + Tailwind 4 + framer-motion 13 + lucide-react
│                           # + @tauri-apps/api 2 + @tauri-apps/cli 2（tauri script 已配，Rust 侧未初始化）
├── index.html / vite.config.ts / tsconfig*.json
└── src/
    ├── main.tsx            # 入口（挂载 App）
    ├── App.tsx             # **1249 行单文件草案**：types + 示例预设数据 + 全部 UI（未拆分）
    ├── App.css             # 样式
    └── vite-env.d.ts
```

**现状结论**：

- Tauri **Rust 后端尚未初始化**（无 `src-tauri/`）；前端依赖已就绪，`npm run tauri` 待建。
- 现有 `App.tsx` 是单体草案：内联 `DimensionMapping/Dimension/Filter/Preset` 类型、
  示例预设（如「深夜影院」）与全部组件逻辑——**目标结构尚未落地**。
- 数据全部为前端静态示例，未接设备枚举、config 读写与 TOML 预设库。

### 2.2 目标架构草案（用户给定，规范以此为准）

```text
src/
├── App.tsx                    # 主布局 + 状态管理（拆分后只做组合与状态）
├── App.css                    # 样式
├── main.tsx                   # 入口
├── types.ts                   # 类型定义（唯一类型源）
├── data/
│   └── presets.ts             # 预设和设备数据（先静态示例，后接后端加载）
├── components/
│   ├── Header.tsx             # 顶栏
│   ├── Footer.tsx             # 底栏状态
│   ├── DeviceSelector.tsx     # 设备切换下拉
│   ├── PresetCard.tsx         # 预设卡片
│   ├── DimensionSlider.tsx    # 感知维度滑条
│   ├── FreqResponseCurve.tsx  # EQ 曲线预览
│   └── FilterRow.tsx          # 滤波器行
└── tabs/
    ├── PresetPanel.tsx        # TAB 1：预设选择
    ├── EffectPanel.tsx        # TAB 2：维度调节
    └── AdvancedPanel.tsx      # TAB 3：高级参数
```

---

## 三、目标模块结构（逐文件职责）

### 3.1 根文件

| 文件 | 职责 | 约束 |
|------|------|------|
| `main.tsx` | React 挂载入口，引入全局样式 | 不含业务逻辑 |
| `App.tsx` | **主布局 + 状态管理**：tab 切换、当前设备、激活预设集合、各维度值、intensity、config 生成/写回触发 | 只做组合与状态，**不内联组件实现**；拆分后目标 < 300 行 |
| `App.css` | 全局样式（Tailwind 之外的自定义样式） | — |
| `types.ts` | 全部共享类型（见 §五） | 组件/面板不得各自定义业务类型（移除现有 App.tsx 内联类型） |

### 3.2 data/

| 文件 | 职责 | 约束 |
|------|------|------|
| `presets.ts` | 预设与设备示例数据（启动期静态数据） | 后续改为后端加载：`load_presets()` / `list_devices()` 命令；文件内只留类型化数据，不含 UI |

### 3.3 components/（通用组件）

| 组件 | 职责 | Props（草案） | 状态归属 |
|------|------|---------------|----------|
| `Header.tsx` | 顶栏：App 名、设备下拉、当前状态摘要 | `deviceId, onDeviceChange, dirty` | 受控，无内部业务状态 |
| `Footer.tsx` | 底栏：当前激活预设总览、全局预增益、写入状态（已保存/未保存/错误） | `summary, writeState` | 受控 |
| `DeviceSelector.tsx` | 设备切换下拉 | `devices, deviceId, onChange` | 受控 |
| `PresetCard.tsx` | 预设卡片：名称/描述/场景/图标 + 强度条 + 启用开关 | `preset, intensity, active, onIntensity, onToggle` | 受控 |
| `DimensionSlider.tsx` | 感知维度滑条：名称、两端标签、说明、值 | `dimension, value, onChange` | 受控（值由 App 持有） |
| `FreqResponseCurve.tsx` | EQ 频响曲线预览（Canvas/SVG 绘制滤波器叠加后频响） | `filters` | 纯展示（`useMemo` 派生，不接后端） |
| `FilterRow.tsx` | 滤波器行（高级面板）：类型/频率/增益/Q/启用/删除 | `filter, index, onChange, onDelete` | 受控 |

### 3.4 tabs/（业务面板）

| 面板 | 职责 | 与 Tab 双向同步 |
|------|------|------------------|
| `PresetPanel.tsx`（TAB 1） | 预设卡片网格 + 强度 + 多预设叠加 | 改强度 → Tab2 维度值按比例缩放 |
| `EffectPanel.tsx`（TAB 2） | 维度滑条组（当前激活预设的 dimensions） | 改维度 → Tab1 强度反算 |
| `AdvancedPanel.tsx`（TAB 3） | 滤波器链编辑（FilterRow 列表）+ 频响曲线 + 预增益 + 导入导出 | 手动改参数 → 反算维度（自定义滤波器标注「自定义」） |

---

## 四、状态管理与数据流

### 4.1 状态单一来源

业务状态全部挂在 `App.tsx` 顶层（`useState`/`useReducer`），组件一律受控：

```ts
interface AppState {
  tab: TabId;                      // "preset" | "effect" | "advanced"
  deviceId: string | null;         // 当前设备 GUID（null = 未选择）
  activePresets: string[];         // 激活预设 id 集合（多预设叠加）
  intensities: Record<string, number>;        // presetId -> intensity [0,1]
  dimensionValues: Record<string, number>;    // dimensionId -> value [0,1]
  manualFilters: Filter[];         // Tab3 手动滤波器（自定义）
  writeState: "idle" | "dirty" | "saving" | "saved" | "error";
  error?: string;
}
```

### 4.2 数据流

```text
UI 操作（滑条/开关/编辑）
  → 更新 AppState（单一来源）
  → useMemo 派生：
      - 有效维度值 = dimension.value × intensity（钳制 [0,1]）
      - config 文本 = buildConfig(激活预设展开 + manualFilters + Preamp)
  → 写回（Tauri 命令 / CLI 逻辑）：
      write_device_config(deviceId, configText)   # 写 C:\ProgramData\VxAPO\{GUID}\config.toml
  → DLL watcher 检测变更 → 热重载（10ms 过渡）
  → Footer 状态：saved / error（写失败保留旧链）
```

### 4.3 派生规则（纯函数，放 `utils/` 或后端）

- `effectiveDimensionValue(dimension, intensity) = clamp(value × intensity, 0, 1)`
- `expandPreset(preset, intensity) -> Filter[]`：按 DimensionMapping 插值（Linear/Log/Exp）
- `mergeFilters(presets[])`：多预设同参数**加法合并** + 钳制到参数合理范围 + 自动预增益（intent 5.2）
- `buildConfig(...) -> string`：输出 TOML `[[effects]]` 模型文本（v9.11 起与 driver 一致；
  旧 EAPO txt 由 CLI `config convert` 一次性迁移，不再作为运行时格式）

> 这些规则最终放在 Tauri Rust 后端（复用 CLI/driver 逻辑），前端先以纯函数实现，阶段 C 迁移。

---

## 五、类型规范（types.ts）

与 intent.md 概念模型一一对应（Preset / Dimension / DimensionMapping / Filter）：

```ts
export type TabId = "preset" | "effect" | "advanced";

export interface Device {
  id: string;            // {GUID}
  name: string;
  installed: boolean;
  mode?: string;         // 安装模式（SfxEfx / SfxMfx / LfxGfx）
  slotStatus?: string;   // 槽位占用摘要（经 CLI/driver 枚举）
}

export interface DimensionMapping {
  filter_index: number;
  param: "gain" | "frequency" | "q";
  min_value: number;
  max_value: number;
  interpolation?: "linear" | "logarithmic" | "exponential";
}

export interface Dimension {
  id: string;
  name: string;
  low_label: string;
  high_label: string;
  description: string;
  default_value: number;
  mappings: DimensionMapping[];
}

export interface Filter {
  enabled: boolean;
  type: string;          // PK / LP / HP / LS / HS / AP / NO（driver 语法）
  frequency: number;
  gain: number;          // dB
  q: number;
  label: string;
  source: string;        // "preset:<id>" | "manual"
}

export interface Preset {
  id: string;
  name: string;
  description: string;
  use_case: string;
  icon: string;
  dimensions: Dimension[];
  filters: Filter[];     // 基准滤波器（dimension.value = default 时的展开）
  default_intensity: number;
}

export interface AppState { /* 见 §4.1 */ }
```

> 草案中 App.tsx 内联的 `DimensionMapping/Dimension/Filter/Preset` 必须迁入 `types.ts`，
> 并补充 `Device / TabId / AppState`。

---

## 六、已定决策（硬性）

1. **响度补偿不做可视化**：只留开关（写 `Loudness: on|off`，默认 on）；不显示补偿曲线/补偿值。
2. **主动限幅在 App 写回时做**：`buildConfig` 输出前按 driver 规则 clamp——
   `[-120, +48] dB`（滤波深切地板 `-60 dB`）、NaN/inf 拒绝；**DLL 不回写文件**。
3. **调音组件开关、总开关**：后续再议（本规范预留 `enabled` 字段，不实现 UI）。
4. **EQ 频响预览保留**（`FreqResponseCurve`）：属于高级参数可视化，与「响度补偿可视化不做」不冲突。
5. **config 路径固定**：`C:\ProgramData\VxAPO\{GUID}\config.toml`（与 CLI/driver 对齐，不用 Documents）。
6. **多预设叠加**：加法合并 + 钳制 + 自动预增益（intent 5.2）。

---

## 七、后端命令接口（Tauri，规划）

> Rust 后端（`src-tauri/`）复用 CLI/driver 逻辑；前端只经 `@tauri-apps/api` invoke。

| 命令 | 输入 | 输出 | 底层 |
|------|------|------|------|
| `list_devices` | — | `Device[]` | driver `enumerate_devices` / CLI 逻辑 |
| `get_device_config` | deviceId | `{ text, valid, filters }` | 读 config.toml（模型校验由 driver 承担） |
| `write_device_config` | deviceId, text | `{ ok, error? }` | 写 `C:\ProgramData\VxAPO\{GUID}\config.toml`（写前主动限幅） |
| `load_presets` | — | `Preset[]` | 读 `_global/presets/*.toml` |
| `save_preset` | preset | `{ ok, error? }` | 写 TOML |
| `install_device` / `uninstall_device` | deviceId | `{ ok, error? }` | CLI/driver install 层（管理员） |
| `snapshot_diff` / `snapshot_restore` | deviceId | diff 文本 / ok | CLI 快照逻辑 |
| `restart_audio` | — | ok | 重启 AudioSrv（管理员） |

---

## 八、修改路线（分阶段）

### Phase A：架构拆分（当前）

- 把 1249 行 `App.tsx` 按目标目录拆分为 `types.ts / data/presets.ts / components/* / tabs/*`。
- **行为不变**：拆分阶段只搬代码 + 建立组件契约，不改业务逻辑。
- 验收：`npm run build`（tsc + vite）通过；功能与拆分前一致。

### Phase B：后端接入

- 初始化 `src-tauri/`（Tauri 2）；实现 §七 命令（先 `list_devices` + config 读写）。
- 前端 `data/presets.ts` 与设备列表改为 invoke 后端。

### Phase C：TAB 1 预设选择

- 预设卡片 + 强度条 + 多预设叠加（`mergeFilters`）；写回 config 生效。

### Phase D：TAB 2 维度调节

- 维度滑条 ↔ 强度双向同步；映射引擎（Linear/Log/Exp）前后端打通。

### Phase E：TAB 3 高级参数

- 滤波器链编辑（FilterRow）+ `FreqResponseCurve` 频响预览 + EAPO config 导入导出。

### Phase F：设备管理 + 发布准备

- 设备切换/继承复制/安装引导（复用 CLI）；`config normalize`（主动限幅回写）集成；
- 主题、托盘、安装包（MSI/NSIS）。

---

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

- `intent.md`：三层分离、Preset→Dimension→Filter 概念模型、多预设叠加、逆向意图识别（导入）
- `CLI 引用规范.md`：后端复用的命令/设备解析/快照逻辑与硬性约束
- `config 模块规范.md`：config.toml 模型、spec 指纹、热重载语义
- `pipeline 模块规范.md`：DSP 能力与数值边界（限幅规则依据）
- `项目定位/VxAPO 项目完整方案.txt`：App 阶段路线（阶段 C/D）
