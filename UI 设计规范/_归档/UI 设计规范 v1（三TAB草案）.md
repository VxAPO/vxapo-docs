# VxAPO App — UI 设计规范 v1

> 本版仅覆盖 **TAB 1「预设控件」** 的完整 UI 规范。
> TAB 2「感知调节」、TAB 3「高级模式」后续单独制定，届时顶栏、标签栏、设备区、底栏复用本版定义。

---

## 一、设计令牌（Design Tokens）

### 1.1 色彩系统

避免纯白 `#FFFFFF` 和纯黑 `#000000`，灰色中性偏微暖。

#### 浅色模式

```css
:root[data-theme="light"] {
  /* 表面层级 — 自顶向下从深到浅 */
  --surface-frame:     #E8E8E8;   /* 顶栏、底栏 */
  --surface-tab:       #ECECEC;   /* 标签栏、收藏栏（与 frame 同级微调） */
  --surface-content:   #F4F4F4;   /* 主内容区 */
  --surface-card:      #FAFAFA;   /* 模块卡片 */
  --surface-raised:    #F7F7F7;   /* 活跃标签、hover 填充 */
  --surface-sunken:    #E0E0E0;   /* 下拉面板、浮层 */
  --surface-overlay:   rgba(0, 0, 0, 0.3);  /* 模态蒙层 */

  /* 描边 */
  --border-strong:     #C8C8C8;   /* 同色背景控件描边 */
  --border-default:    #D6D6D6;   /* 标准分界线 */
  --border-subtle:     #E4E4E4;   /* 弱分割线 */

  /* 文字 */
  --text-primary:      #1C1C1C;   /* 标题、重要信息 */
  --text-secondary:    #555555;   /* 正文 */
  --text-tertiary:     #8A8A8A;   /* 标签、提示 */
  --text-disabled:     #B0B0B0;   /* 不可交互项 */
  --text-on-accent:    #F8F8F8;   /* 彩色背景上的文字 */

  /* 功能色 — 极少使用 */
  --accent-green:      #16A34A;   /* 调音开关-启用 */
  --accent-green-soft: rgba(22, 163, 74, 0.12);
  --danger-red:        #DC2626;   /* 拖拽删除区域 */
  --danger-red-soft:   rgba(220, 38, 38, 0.08);

  /* 模糊与阴影 */
  --blur-sm:           8px;
  --blur-md:           20px;
  --shadow-sm:         0 1px 2px rgba(0,0,0,0.04);
  --shadow-md:         0 2px 8px rgba(0,0,0,0.06), 0 1px 3px rgba(0,0,0,0.03);
  --shadow-lg:         0 4px 16px rgba(0,0,0,0.08), 0 2px 6px rgba(0,0,0,0.04);
  --shadow-card:       0 1px 3px rgba(0,0,0,0.04), 0 0 0 1px var(--border-default);
}
```

#### 深色模式

```css
:root[data-theme="dark"] {
  --surface-frame:     #202020;
  --surface-tab:       #232323;
  --surface-content:   #2A2A2A;
  --surface-card:      #313131;
  --surface-raised:    #383838;
  --surface-sunken:    #1A1A1A;
  --surface-overlay:   rgba(0, 0, 0, 0.5);

  --border-strong:     #484848;
  --border-default:    #3A3A3A;
  --border-subtle:     #303030;

  --text-primary:      #E8E8E8;
  --text-secondary:    #A0A0A0;
  --text-tertiary:     #686868;
  --text-disabled:     #484848;
  --text-on-accent:    #F8F8F8;

  --accent-green:      #22C55E;
  --accent-green-soft: rgba(34, 197, 94, 0.15);
  --danger-red:        #EF4444;
  --danger-red-soft:   rgba(239, 68, 68, 0.12);

  --blur-sm:           8px;
  --blur-md:           20px;
  --shadow-sm:         0 1px 2px rgba(0,0,0,0.2);
  --shadow-md:         0 2px 8px rgba(0,0,0,0.3);
  --shadow-lg:         0 4px 16px rgba(0,0,0,0.4);
  --shadow-card:       0 1px 3px rgba(0,0,0,0.2), 0 0 0 1px var(--border-default);
}
```

### 1.2 圆角

```css
:root {
  --radius-xs:   4px;
  --radius-sm:   6px;
  --radius-md:   8px;
  --radius-lg:   12px;
  --radius-pill: 9999px;
}
```

### 1.3 间距

```css
:root {
  --space-2xs:  2px;
  --space-xs:   4px;
  --space-sm:   8px;
  --space-md:   12px;
  --space-lg:   16px;
  --space-xl:   24px;
  --space-2xl:  32px;
}
```

### 1.4 字体

```css
:root {
  --font-sans:  "DM Sans", "Microsoft YaHei UI", system-ui, sans-serif;
  --font-mono:  "Space Mono", "Cascadia Code", monospace;
}
```

| 用途 | 字号 | 字重 | 行高 |
|---|---|---|---|
| Logo | 18px | 700 | 1 |
| 标签页标题 | 13px | 500 | 1 |
| 卡片标题 | 14px | 600 | 1.3 |
| 卡片描述/场景 | 12px | 400 | 1.5 |
| 控件工具提示 | 12px | 400 | 1 |
| 状态栏 | 11px | 400 | 1 |
| 快捷键标注 | 11px | 400 mono | 1 |
| 模块序号 | 11px | 500 mono | 1 |

---

## 二、交互状态规范

### 2.1 光标（Cursor）

| 状态 | CSS | 触发场景 |
|---|---|---|
| 默认箭头 | `cursor: default` | 不可交互区域、空白处 |
| 食指手型 | `cursor: pointer` | 按钮、标签、列表项、可点击区域 |
| 禁止 | `cursor: not-allowed` | 未安装设备的槽位、禁用控件 |
| 抓取 | `cursor: grab` | 可拖拽元素（收藏栏预设、模块手柄） |
| 抓取中 | `cursor: grabbing` | 正在拖拽时 |
| 放置 | `cursor: copy` | 拖拽预设进入模块列表区域时 |

### 2.2 按钮按下反馈

所有可点击按钮统一：

```css
.btn-press:active {
  transform: scale(0.96);
  filter: brightness(0.92);
  transition: transform 0.1s ease, filter 0.1s ease;
}
```

### 2.3 描边规则

| 场景 | 处理 |
|---|---|
| 控件与父容器**同色背景** | `1px solid var(--border-strong)` + 微阴影 |
| 控件与父容器**异色背景** | 不加描边，靠颜色差异区分 |
| 描边颜色 | 接近控件自身背景色，对比度 ≤ 1.2:1 |

---

## 三、图标清单

全部使用 **Lucide React**（已安装 `lucide-react`）。

本页涉及的所有图标使用位置均以 **`[图标名]`** 标注，不再使用文字或 HTML 实体。

| 图标组件名 | 导入名 | 本页使用位置 |
|---|---|---|
| `<Settings />` | `Settings` | 顶栏工具按钮 |
| `<Save />` | `Save` | 顶栏工具按钮 |
| `<Upload />` | `Upload` | 顶栏工具按钮 |
| `<Share2 />` | `Share2` | 顶栏工具按钮 |
| `<Check />` | `Check` | 设置菜单当前选项指示 |
| `<ChevronRight />` | `ChevronRight` | 设置子菜单展开指示 |
| `<ChevronDown />` | `ChevronDown` | 设备下拉按钮展开指示 |
| `<Volume2 />` | `Volume2` | 播放设备指示、设备列表项前缀 |
| `<Mic />` | `Mic` | 捕获设备指示、设备列表项前缀 |
| `<Download />` | `Download` | 安装按钮（未安装状态） |
| `<CircleCheck />` | `CircleCheck` | 安装按钮（已安装状态） |
| `<GripVertical />` | `GripVertical` | 模块卡片拖拽手柄 |
| `<Trash2 />` | `Trash2` | 拖拽删除区域 |
| `<X />` | `X` | 模态框关闭 |

---

## 四、页面总体布局

> 本节描述的"内容区"即 TAB 1「预设控件」的内容页面。
> TAB 2、TAB 3 将各自定义自己的内容区，与本版共享顶栏、标签栏、设备区、底栏。

```
Window (Tauri decorations: false, 自绘标题栏)
┌──────────────────────────────────────────────────────────────────┐
│ [VxAPO]  [Settings][Save][Upload][Share2]    拖拽区    [─][□][×]│ ← 顶栏
├──────────────────────────────────────────────────────────────────┤
│ [预设控件] [感知调节] [高级模式]  [播放/捕获][设备▾][安装][调音]  │ ← 标签栏
├──────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ 🎬 深夜影院   🎯 FPS脚步   🎙️ 人声清晰   🔊 低音增强      │  │ ← 收藏栏 ★
│  ├────────────────────────────────────────────────────────────┤  │
│  │                                                            │  │
│  │  ┌ 模块卡片 #1 ──────────────────────────────────────┐    │  │
│  │  ┌ 模块卡片 #2 ──────────────────────────────────────┐    │  │ ← 模块列表 ★
│  │  ┌ 模块卡片 #3 ──────────────────────────────────────┐    │  │
│  │  │  ...                                              │    │  │
│  │  └───────────────────────────────────────────────────┘    │  │
│  └────────────────────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────────────────┤
│ ● VxAPO已安装 │ 配置已保存 │ 当前激活:... │ 调音已生效 │ 2ch·48k │ ← 底栏
└──────────────────────────────────────────────────────────────────┘

★ = 仅限 TAB 1「预设控件」页面
```

---

## 五、全局框架（三 TAB 共用）

### 5.1 顶栏（Top Bar）

**背景：** `var(--surface-frame)`
**高度：** 40px
**左右内边距：** `var(--space-lg)`
**布局：** flex 水平

#### VxAPO Logo

- **内容：** 文字 `V` + `x` + `APO`（纯文本占位，无图标）
- `V` `APO`：`var(--text-primary)`
- `x`：`var(--accent-green)`
- 字体：`var(--font-sans)`，18px，700
- 位置：顶栏最左侧

#### 工具按钮组

Logo 右侧，间隔 `var(--space-xs)`，从左到右：

| 序号 | 图标 | 功能名 | 快捷键 |
|---|---|---|---|
| 1 | `[Settings]` | 设置 | `Ctrl+,` |
| 2 | `[Save]` | 保存 | `Ctrl+S` |
| 3 | `[Upload]` | 导入 | `Ctrl+I` |
| 4 | `[Share2]` | 导出 | `Ctrl+E` |

**按钮样式：**

| 属性 | 值 |
|---|---|
| 尺寸 | 28px × 28px |
| 图标尺寸 | 16px |
| 图标颜色 | `var(--text-secondary)` |
| 默认背景 | 透明 |
| Hover 背景 | `var(--surface-raised)`，淡入 200ms |
| Hover 圆角 | `var(--radius-sm)` |
| 按下 | `.btn-press` |
| 光标 | `cursor: pointer` |

**Hover 工具提示（Tooltip）：**

| 属性 | 值 |
|---|---|
| 出现延迟 | 500ms |
| 背景 | `var(--surface-sunken)` |
| 圆角 | `var(--radius-sm)` |
| 阴影 | `var(--shadow-md)` |
| 内边距 | `6px 10px` |
| 字体 | 12px，功能名 `var(--text-primary)` + 快捷键 `var(--text-tertiary)` mono |
| 动画 | opacity 0→1 + translateY(-4→0)，150ms |
| 位置 | 按钮正下方居中，间距 6px |

#### 窗口拖拽区域

工具按钮组右侧至窗口控件之间的空白区域。
属性：`data-tauri-drag-region`

#### 窗口控件

顶栏最右侧，标准 Win11 最小化/最大化/关闭按钮。取系统默认样式，**不取 Windows 主题色**。

---

### 5.2 设置菜单

#### 一级菜单

点击 `[Settings]` 图标弹出（**非 hover 触发**）。

| 属性 | 值 |
|---|---|
| 触发 | 点击 |
| 背景 | `var(--surface-sunken)` + `backdrop-filter: blur(var(--blur-md))` |
| 圆角 | `var(--radius-md)` |
| 阴影 | `var(--shadow-lg)` |
| 边框 | `1px solid var(--border-default)` |
| 最小宽度 | 180px |
| 内边距 | `var(--space-xs)` 上下 |
| 出现动画 | opacity + translateY(-4→0)，150ms ease |
| 消失动画 | opacity + translateY→-4px，100ms ease |
| 关闭 | 点击菜单外部区域 |

**菜单项：**

| 属性 | 值 |
|---|---|
| 高度 | 32px |
| 内边距 | `0 12px` |
| 字体 | 13px，`var(--text-primary)` |
| Hover | 背景 `var(--surface-raised)` |
| 内部圆角 | `var(--radius-xs)` |
| 可展开指示 | 右侧 `[ChevronRight]` 图标，14px，`var(--text-tertiary)` |

| 菜单项 | 类型 |
|---|---|
| 默认主题 | 可展开（hover 展开子菜单） |
| 响度补偿 | 可展开（hover 展开子菜单） |

#### 二级子菜单

**触发：** Hover 到可展开项（**非点击**）
**展开延迟：** 300ms

| 属性 | 值 |
|---|---|
| 位置 | 一级菜单项右侧，间距 2px |
| 样式 | 同一级菜单 |
| 最小宽度 | 140px |

**默认主题 → 子菜单项：**

| 选项 | 当前选中指示 |
|---|---|
| 浅色模式 | 左侧 `[Check]` 图标（14px，`var(--text-secondary)`） |
| 深色模式 | 同上 |
| 跟随系统 | 同上 |

**响度补偿 → 子菜单项：**

| 选项 | 当前选中指示 |
|---|---|
| 开 | 左侧 `[Check]` 图标 |
| 关 | 同上 |

**子菜单项通用样式：**

| 属性 | 值 |
|---|---|
| 高度 | 32px |
| 字体 | 13px |
| 当前选中 | 文字 `var(--text-primary)` + 左侧 `[Check]` |
| 非当前 | 文字 `var(--text-secondary)`，无图标 |
| Hover | 背景 `var(--surface-raised)` |

---

### 5.3 标签栏（Tab Bar）

**背景：** `var(--surface-tab)`
**高度：** 38px
**布局：** flex 水平，左侧标签组 + 右侧设备区

#### 标签页 — 非活跃

| 属性 | 值 |
|---|---|
| 背景 | 同 `var(--surface-tab)`（与栏位融合） |
| 文字 | `var(--text-tertiary)`，13px，500 |
| 内边距 | `8px 16px` |
| 圆角 | 上方 `var(--radius-md)`，下方 0 |
| 光标 | `cursor: pointer` |
| 按下 | `.btn-press` |

#### 标签间分割线（仅非活跃标签之间）

| 属性 | 值 |
|---|---|
| 样式 | 竖线 |
| 高度 | 16px 居中 |
| 宽度 | 1px |
| 颜色 | `var(--border-subtle)` |

#### 标签页 — 活跃（"凸字"衔接）

```
标签栏 ─────────────────────────────────────────────
        [非活跃]  ┌─活跃标签─┐  [非活跃]
                  │ 预设控件  │
──────────────────┘          └──────────────────────
                  ╭────────────╮
                  │  内容区域   │
```

| 属性 | 值 |
|---|---|
| 背景 | `var(--surface-content)`（与内容区同色，视觉一体） |
| 文字 | `var(--text-primary)`，13px，600 |
| 内容区顶部 | `border-radius: var(--radius-lg) var(--radius-lg) 0 0` |
| 衔接效果 | 标签底部背景色与内容区顶部背景色融合，无断裂 |
| 过渡 | 背景色 + 文字色 200ms ease |

---

### 5.4 设备选择区域

固定在标签栏右侧，布局从左到右：

#### 播放/捕获切换滑块

| 属性 | 值 |
|---|---|
| 容器 | 约 120px × 28px，`var(--surface-frame)`，`var(--radius-pill)` |
| 描边 | `1px solid var(--border-strong)` |
| 左半 | `[Volume2]` 图标（14px）+ "播放"，12px |
| 右半 | `[Mic]` 图标（14px）+ "捕获"，12px |
| 滑块白块 | `var(--surface-raised)`，约 56px 宽，`var(--radius-pill)`，`var(--shadow-sm)` |
| 切换到的一侧 | 白块覆盖，黑字 `var(--text-primary)` |
| 未选中侧 | 透明底，`var(--text-tertiary)` |
| 切换动画 | translateX 200ms cubic-bezier(0.4, 0, 0.2, 1) |
| 光标 | `cursor: pointer` |
| 按下 | `.btn-press` |

#### 设备下拉按钮

**触发：** 点击展开（非 hover）

| 属性 | 值 |
|---|---|
| 按钮背景 | `var(--surface-frame)` |
| 描边 | `1px solid var(--border-strong)` |
| 圆角 | `var(--radius-sm)` |
| 高度 | 28px |
| 内容 | `[Volume2]` 或 `[Mic]` 图标 + 设备名 + `[ChevronDown]` |
| 文字 | `var(--text-primary)`，13px |
| 展开动画 | opacity + scaleY(0.95→1)，transform-origin: top，180ms |

**下拉列表面板：**

| 属性 | 值 |
|---|---|
| 背景 | `var(--surface-sunken)` + `backdrop-filter: blur(var(--blur-md))` |
| 圆角 | `var(--radius-md)` |
| 阴影 | `var(--shadow-lg)` |
| 边框 | `1px solid var(--border-default)` |
| 内边距 | `var(--space-xs)` |
| 最大高度 | 200px（超出滚动） |

**列表项：**

| 属性 | 值 |
|---|---|
| 高度 | 34px |
| 内边距 | `0 10px` |
| 圆角 | `var(--radius-xs)` |
| 字体 | 13px |
| 左侧图标 | `[Volume2]` 或 `[Mic]`，14px |
| **已安装设备** | 文字 `var(--text-primary)` |
| **未安装设备** | 文字 `var(--text-disabled)`，`cursor: not-allowed` |
| **当前选中** | 背景 `var(--surface-raised)` |
| **Hover（仅已安装）** | 背景 `var(--surface-raised)`，淡入 150ms |
| **收起动画** | 抽屉式逐项向上淡出，每项间隔 30ms，总 200ms |

---

### 5.5 安装按钮

| 属性 | 值 |
|---|---|
| 位置 | 设备下拉右侧 |
| 尺寸 | 28px × 28px |
| 背景 | `var(--surface-frame)` |
| 圆角 | `var(--radius-sm)` |
| 描边 | `1px solid var(--border-strong)` |
| 光标 | `cursor: pointer` |
| 按下 | `.btn-press` |
| Hover | 背景 `var(--surface-raised)` + Tooltip（"安装"/"已安装"） |

**状态：**

| 状态 | 内容 |
|---|---|
| 未安装 | 空（仅边框） |
| 已安装 | `[CircleCheck]` 图标，16px，`var(--text-secondary)` |

#### 安装确认模态框

| 属性 | 值 |
|---|---|
| 蒙层 | `var(--surface-overlay)`，淡入 200ms |
| 弹窗背景 | `var(--surface-card)` |
| 圆角 | `var(--radius-lg)` |
| 阴影 | `var(--shadow-lg)` |
| 宽度 | ~400px |
| 内边距 | `var(--space-xl)` |
| 出现动画 | scale(0.95→1) + opacity(0→1)，250ms cubic-bezier(0.16, 1, 0.3, 1) |
| 关闭 | 右上角 `[X]` 图标，16px |

**内容结构：**

```
标题（16px 600）：  安装确认
描述（13px 400）：  确认安装后将执行以下操作：

  ○ 注册 SFX/MFX/EFX 槽位            [Check] 完成 / [X] 失败
  ○ 写入 APO CLSID
  ○ 备份原槽位值
  ○ 创建配置目录
  ○ 写入默认 config.txt

                    [ 取消 ]    [ 确认安装 ]
```

| 元素 | 样式 |
|---|---|
| 操作项 | 13px，`var(--text-secondary)`，间距 `var(--space-sm)` |
| 执行前 | 圆形空心指示器，`var(--border-strong)` |
| 成功 | `[Check]` 图标，`var(--accent-green)` |
| 失败 | `[X]` 图标，`var(--danger-red)` |
| 逐项检测间隔 | 300ms 顺序动画 |
| 全部成功 | 1s 后自动关闭弹窗，安装按钮出现 `[CircleCheck]` |
| 存在失败 | 保持弹窗 |
| 取消按钮 | 描边按钮 |
| 确认安装按钮 | 填充按钮 |

---

### 5.6 调音开关（Toggle）

| 属性 | 值 |
|---|---|
| 位置 | 安装按钮右侧 |
| 轨道 | 40px × 22px，`var(--radius-pill)` |
| **关闭** | 轨道 `var(--surface-frame)` + `1px solid var(--border-strong)` |
| **开启** | 轨道 `var(--accent-green)`，无描边 |
| 滑块 | 16px × 16px，`var(--radius-pill)` |
| 滑块色 | 关闭 `var(--surface-card)` / 开启 `var(--text-on-accent)` |
| 滑块阴影 | `var(--shadow-sm)` |
| 切换动画 | translateX 200ms cubic-bezier(0.4, 0, 0.2, 1) |
| Hover | 轨道 filter: brightness(1.05) |
| 按下 | 滑块 scale(0.9) |
| 光标 | `cursor: pointer` |

---

### 5.7 信息底栏（Status Bar）

**背景：** `var(--surface-frame)`（与顶栏同色）
**高度：** 28px
**内边距：** `0 var(--space-lg)`
**顶部边框：** `1px solid var(--border-default)`
**字体：** 11px，`var(--text-tertiary)`
**布局：** flex 水平，各项间用分割线

**分割线（复用标签页分割线样式）：**

| 属性 | 值 |
|---|---|
| 竖线 | 高 12px 居中，宽 1px，`var(--border-subtle)` |

**信息项（从左到右）：**

| 项目 | 内容 | 状态指示 |
|---|---|---|
| 安装状态 | `VxAPO 已安装` / `VxAPO 未安装` | 已安装=绿色小圆点 8px `var(--accent-green)`；未安装=灰色小圆点 `var(--text-tertiary)` |
| 保存状态 | `配置已保存` / `配置未保存` | 未保存时文字加粗 `var(--text-secondary)` |
| 当前激活 | `当前激活：深夜影院、人声清晰化` / `当前激活：未使用预设` | 预设名 `var(--text-secondary)` |
| 调音状态 | `调音已生效` / `调音未启用` | 小圆点逻辑同安装状态 |
| 响度补偿 | `响度补偿已开启：-2.3 dB` / `响度补偿已关闭` | dB 值 mono 字体 |
| 音频信息 | `2ch · 48kHz · 32bit` | mono 字体 |

---

## 六、TAB 1「预设控件」内容页面

> 以下内容仅属于「预设控件」标签页。
> TAB 2「感知调节」和 TAB 3「高级模式」的各自内容页面后续单独定义。

### 6.1 预设收藏栏（Preset Favorites Bar）

**位于内容区顶部**，紧接内容区圆角下方。

| 属性 | 值 |
|---|---|
| 背景 | `var(--surface-tab)`（比内容区深一级，与标签栏同色） |
| 高度 | 44px |
| 内边距 | `0 var(--space-lg)` |
| 布局 | flex 水平，溢出水平滚动 |
| 底部边框 | `1px solid var(--border-subtle)` |
| 滚动条 | 隐藏 |

**预设分类：** 收藏栏包含两类预设，不设视觉分区，混合排列。

| 类型 | 可删除 | 排列 |
|---|---|---|
| 开发者预设（内置） | 不可删除 | 固定位置 |
| 用户自定义预设 | 可删除（右键菜单或长按） | 追加在开发者预设之后 |

**预设项：**

| 属性 | 值 |
|---|---|
| 格式 | emoji + 空格 + 标题文字 |
| 内边距 | `6px 12px` |
| 圆角 | `var(--radius-sm)` |
| 字体 | 13px，`var(--text-primary)` |
| 默认背景 | 透明 |
| Hover | 背景 `var(--surface-raised)`，淡入 150ms |
| 按下 | `.btn-press` |
| 光标 | `cursor: grab`（可拖拽）/ `cursor: pointer`（可点击） |
| 间距 | 各项之间 `var(--space-xs)` |

**交互行为：**

| 操作 | 结果 |
|---|---|
| **点击** | 对应预设的模块卡片淡入到模块列表末尾 |
| **拖拽** | 见 §6.4 拖拽系统 |

---

### 6.2 模块列表（Module List）

**位于收藏栏下方**，占据内容区剩余空间。

| 属性 | 值 |
|---|---|
| 背景 | `var(--surface-content)` |
| 内边距 | `var(--space-lg)` |
| 布局 | flex column，间距 `var(--space-md)` |
| 溢出 | `overflow-y: auto` |
| 空状态 | 居中文字"从收藏栏拖入或点击添加预设"，`var(--text-tertiary)`，13px |

**新增方向：** 新模块卡片从列表末尾（下方）出现。

#### 自定义滚动条

| 属性 | 值 |
|---|---|
| 宽度 | 6px |
| 轨道 | 透明 |
| 滑块 | `var(--border-strong)`，`var(--radius-pill)` |
| 端点 | 圆头 |
| 淡入 | 鼠标进入列表区域，opacity 0→1，300ms |
| 淡出 | 鼠标离开列表区域，opacity 1→0，600ms |

---

### 6.3 模块卡片（Module Card）

```
┌──────────────────────────────────────────────────────────────────┐
│ [GripVertical]  #1  深夜影院    低频下沉，人声拉近   ████████ 70% │
└──────────────────────────────────────────────────────────────────┘
```

| 属性 | 值 |
|---|---|
| 背景 | `var(--surface-card)` |
| 圆角 | `var(--radius-md)` |
| 阴影 | `var(--shadow-card)` |
| 内边距 | `var(--space-md) var(--space-lg)` |
| 高度 | ~52px |
| 布局 | flex 水平，items-center |
| 按下 | `.btn-press` |

**内部元素（从左到右）：**

| 元素 | 内容类型 | 样式 |
|---|---|---|
| 拖拽手柄 | `[GripVertical]` 图标 | 16px，`var(--text-tertiary)`，hover `var(--text-secondary)`，`cursor: grab` |
| 序号 | 文字 `#1` | mono 11px，`var(--text-tertiary)`，前缀 `var(--space-sm)` |
| 标题 | 文字（如"深夜影院"） | 14px 600，`var(--text-primary)`，前缀 `var(--space-md)` |
| 简介 | 文字（如"低频下沉，人声拉近，高频柔化"） | 12px，`var(--text-tertiary)`，`text-overflow: ellipsis`，flex-1 |
| 强度条 | 交互控件 | 宽 ~120px，前缀 `var(--space-lg)` |
| 强度百分比 | 文字（如"70%"） | mono 12px，`var(--text-secondary)`，前缀 `var(--space-sm)` |

**强度条：**

| 属性 | 值 |
|---|---|
| 轨道高度 | 6px |
| 轨道背景 | `var(--surface-frame)` |
| 轨道圆角 | `var(--radius-pill)` |
| 填充色 | `var(--text-secondary)` |
| 滑块 | 隐藏（直接拖拽轨道） |
| 光标 | `cursor: pointer` |

---

### 6.4 拖拽系统（仅限 TAB 1）

#### A. 从收藏栏拖拽预设到模块列表

| 阶段 | 视觉 |
|---|---|
| **抓取** | `cursor: grabbing` |
| **拖拽幽灵** | 光标位置圆角矩形，`var(--surface-card)` + `backdrop-filter: blur(var(--blur-sm))` + `var(--shadow-md)`，内容 `emoji + 标题`，~160px × 36px，`var(--radius-md)` |
| **进入列表区域** | `cursor: copy`；列表中浮现占位矩形，`var(--surface-frame)` 背景，`2px dashed var(--border-strong)` 边框，高度等于卡片默认高度 |
| **已有模块位移** | `transform: translateY()`，200ms cubic-bezier(0.16, 1, 0.3, 1)，类似安卓桌面图标位移 |
| **靠近列表边界** | 自动滚动：距顶/底 60px 内触发，2px/帧→12px/帧（距边界 10px 时） |
| **释放** | 幽灵消失，新卡片在占位处淡入（opacity + translateY(8→0)），250ms ease-out |

#### B. 模块卡片拖拽（重排序 / 删除）

| 阶段 | 视觉 |
|---|---|
| **抓取** | 按住 `[GripVertical]`，`cursor: grabbing` |
| **卡片变形** | scale(0.92)，opacity 0.8，150ms ease |
| **顶部删除区域浮现** | 列表顶部滑入红色条带（h: 56px），`var(--danger-red)` 背景，居中 `[Trash2]` 图标（20px，`var(--text-on-accent)`）+ 文字"拖拽到此区域删除"（13px，`var(--text-on-accent)`），200ms |
| **拖入删除区域** | 红色条带 filter: brightness(0.85) |
| **在删除区域释放** | 卡片缩小→消失（scale + opacity→0，200ms），红色条带滑上消失，其余模块上移填补 |
| **右键取消** | 卡片恢复原位（200ms），红色条带消失 |
| **拖拽幽灵** | 缩小版完整卡片样式（非 emoji+标题） |
| **边界自动滚动** | 同 A |

---

### 6.5 动画汇总表（TAB 1 相关）

| 动画 | 触发 | 变化 | 时长 | 缓动 |
|---|---|---|---|---|
| 工具按钮 hover | mouseenter | 背景 opacity | 200ms | ease |
| 工具提示出现 | hover 500ms | opacity + translateY | 150ms | ease |
| 设置菜单展开 | click | opacity + scale | 150ms | ease |
| 子菜单展开 | hover 300ms | opacity + translateX | 150ms | ease |
| 标签切换 | click | 背景色 + 文字色 | 200ms | ease |
| 设备类型滑块 | click | translateX | 200ms | cubic-bezier(0.4,0,0.2,1) |
| 设备列表展开 | click | opacity + scaleY | 180ms | ease |
| 设备列表收起 | click outside | 逐项淡出（间隔 30ms） | 200ms | ease |
| 安装弹窗出现 | click install | opacity + scale | 250ms | cubic-bezier(0.16,1,0.3,1) |
| 安装弹窗关闭 | confirm/cancel | opacity + scale | 200ms | ease |
| 安装步骤检测 | 逐项 | 状态图标淡入（间隔 300ms） | 200ms | ease |
| Toggle 切换 | click | 滑块 translateX + 轨道色 | 200ms | cubic-bezier(0.4,0,0.2,1) |
| 模块卡片入场 | 添加预设 | opacity + translateY(8→0) | 250ms | ease-out |
| 模块位移 | 拖拽插入 | translateY | 200ms | cubic-bezier(0.16,1,0.3,1) |
| 拖拽幽灵出现 | 开始拖拽 | opacity + scale(0.92) | 150ms | ease |
| 删除区域出现 | 拖拽卡片 | translateY(-56→0) | 200ms | ease |
| 删除释放 | 释放到删除区 | scale + opacity→0 | 200ms | ease |
| 拖拽取消 | 右键 | 恢复 scale + opacity | 200ms | ease |
| 滚动条淡入 | mouseenter 列表 | opacity 0→1 | 300ms | ease |
| 滚动条淡出 | mouseleave 列表 | opacity 1→0 | 600ms | ease |
| 按钮按下 | mousedown | scale(0.96) + brightness(0.92) | 100ms | ease |
| 按钮释放 | mouseup | 恢复 | 100ms | ease |
