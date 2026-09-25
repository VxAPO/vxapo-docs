# 01 设计基础与主题令牌

以 `src/new.css` 为样式入口（`@import styles/*` 按固定顺序，`dark.css` 最后），
实际令牌定义在 `src/styles/theme.css`。本文只描述当前实际使用的令牌与规则。

## 1. 主题结构

主题由 `document.documentElement.dataset.theme` 驱动，取值为 `light` | `dark`。对应选择器：

- `:root[data-theme="light"]`
- `:root[data-theme="dark"]`

浅色与深色共用同一套组件类名，只切换 CSS 变量。所有颜色变量用 `@property` 注册为可插值
自定义属性，主题切换时通过 `.theme-transition` 类给整页加 0.35s 颜色过渡（结束后移除，
避免日常全局过渡卡顿）。

## 2. 核心颜色令牌（styles/theme.css）

### 浅色

| 令牌 | 值 | 用途 |
|---|---|---|
| `--frame` | `#f0f3f6` | 窗口底、TopBar |
| `--content` | `#f7f9fc` | 主内容区 |
| `--card` | `#f7f9fc` | 侧边栏、卡片 |
| `--surface-inset` | `rgb(52 63 81 / 2%)` | 拖拽占位槽内陷底 |
| `--text-primary` | `#3d4552` | 主文本 |
| `--text-secondary` | `#3d4552` | 次级文本 |
| `--text-weak` | `#69707c` | 弱文本 |
| `--border` | `#d9dee7` | 常规描边 |
| `--border-strong` | `#b3bbc8` | 强描边 |
| `--curve-axis` | `#b3bbc8` | 曲线坐标轴实线 |
| `--curve-path` | `#009aa2` | 曲线主体 |
| `--hover` | `#e3e8ef` | hover 底 |
| `--track` | `#e3e7ee` | 轨道/槽 |
| `--theme-hover-bg` | `#c2cbd8` | 主题三段 hover 底 |
| `--brand` | `#47c3d1` | 品牌青 |
| `--brand-deep` | `#009aa2` | 品牌深青 |
| `--brand-soft` | `rgba(71,195,209,0.1)` | 品牌软底 |
| `--brand-bg` | `#e2f5f7` | 视图滑块 thumb 底 |
| `--danger` | `#dc3d43` | 危险/删除 |
| `--danger-soft` | `rgba(220,61,67,0.1)` | 危险软底 |
| `--on-accent` | `#ffffff` | 深色底上的文字 |
| `--accent-bg` | `#3b3c3f` | 激活胶囊底、Toast 底 |
| `--thumb` | `#ffffff` | 滑块圆头 |
| `--gs-thumb-border` | `#3d4552` | 滑块圆头描边 |
| `--gs-thumb-disabled` | `#ffffff` | 禁用滑块圆头（浅色同白） |
| `--ring-hover` | `rgba(15,23,42,0.14)` | 输入 focus 环 |
| `--scrollbar` | `#d9dce4` | 滚动条 thumb |
| `--scrollbar-hover` | `#c2c7d1` | 滚动条 hover |
| `--disabled-fill` | `#b8bec9` | 禁用填充 |
| `--tab-active-bg` | `#e9edf1` | 活跃标签底 |

### 深色

| 令牌 | 值 | 用途 |
|---|---|---|
| `--frame` | `#16181b` | 窗口底、TopBar |
| `--content` | `#1d1f22` | 主内容区 |
| `--card` | `#1d1f22` | 侧边栏、卡片 |
| `--surface-inset` | `rgba(147,197,253,0.02)` | 拖拽占位槽内陷底 |
| `--text-primary` | `#c1c6cd` | 主文本 |
| `--text-secondary` | `#969ca4` | 次级文本 |
| `--text-weak` | `#717882` | 弱文本 |
| `--border` | `#373a3e` | 常规描边 |
| `--border-strong` | `#51555b` | 强描边 |
| `--curve-axis` | `#7c7d7f` | 曲线坐标轴实线 |
| `--curve-path` | `#47c3d1` | 曲线主体 |
| `--hover` | `#25292e` | hover 底 |
| `--track` | `#22262a` | 轨道/槽 |
| `--theme-hover-bg` | `#363c45` | 主题三段 hover 底 |
| `--brand` | `#00919e` | 深色品牌基础色 |
| `--brand-deep` | `#006265` | 深色品牌 hover/更深色 |
| `--brand-soft` | `rgba(71,195,209,0.14)` | 品牌软底 |
| `--brand-bg` | `#1d3e44` | 视图滑块 thumb 底 |
| `--danger` | `#ea5556` | 危险/删除 |
| `--danger-soft` | `rgba(234,85,86,0.15)` | 危险软底 |
| `--on-accent` | `#c1c6cd` | 深色激活底上的文字 |
| `--accent-bg` | `#43474e` | 激活胶囊底、Toast 底 |
| `--shadow-ink` | `15, 18, 23` | 阴影冷灰近黑（深色用） |
| `--thumb` | `#c5c9d0` | 滑块圆头 |
| `--gs-thumb-border` | `#1d1f22` | 滑块圆头描边（同卡片底） |
| `--gs-thumb-disabled` | `#6d7279` | 禁用滑块圆头（比线条浅、比启用深） |
| `--ring-hover` | `rgba(255,255,255,0.14)` | 输入 focus 环 |
| `--scrollbar` | `#3f4348` | 滚动条 thumb |
| `--scrollbar-hover` | `#4f5358` | 滚动条 hover |
| `--disabled-fill` | `#32363b` | 禁用填充 |
| `--tab-inactive` | `#2f3135` | 标签 hover 底 |
| `--tab-active` | `#45474c` | 活跃标签底 |
| `--tab-active-bg` | `#45474c` | 活跃标签底（tab-btn 用） |

### 品牌色规则

- 浅色：`--brand` 用于胶囊底色/描边/主题文本；`--brand-deep` 用于需要更深对比的文字与 hover 外圈。
- 深色：`--brand` 是基础品牌色；`--brand-deep` 用于 on 状态的 hover 外圈等需要“更深”的场景。
- 声道胶囊 `.ch-pill.active`、安装胶囊 `.install-btn`、保存预设胶囊 `.sel-action.save`、
  设备属性按钮 `.dev-prop-btn:hover` 的描边与字体在深色下统一用 `--brand`。

## 3. 阴影

### 浅色

```css
--card-shadow: 0 1px 2px rgba(15, 23, 42, 0.02), 0 1px 3px rgba(15, 23, 42, 0.04);
--hover-shadow: 0 2px 4px rgba(15, 23, 42, 0.06), 0 4px 10px rgba(15, 23, 42, 0.08);
```

### 深色

深色阴影不使用纯黑，使用冷灰近黑 `--shadow-ink: 15,18,23`，避免阴影像发光：

```css
--card-shadow: 0 1px 3px rgba(var(--shadow-ink), 0.7), 0 4px 14px rgba(var(--shadow-ink), 0.4);
--hover-shadow: 0 4px 10px rgba(var(--shadow-ink), 0.8), 0 6px 16px rgba(var(--shadow-ink), 0.55);
```

## 4. 圆角与间距

### 圆角

| 变量/类 | 值 |
|---|---|
| 卡片、曲线卡、设备属性卡 | `16px` |
| 侧边栏、主内容区 | `26px` |
| 弹窗 | `20px` |
| 胶囊按钮、标签、输入框 | `9999px` / `8px` / `10px` |
| 标签按钮 | `16px` |

### 间距

- App 外壳左右下内边距：`10px`
- 侧边栏与主内容区之间：`10px`（`--side-gap`）
- 卡片网格间距：`12px`
- 表单行间距：`8px`

## 5. 字体

- 字体族：`"Plus Jakarta Sans", "Microsoft YaHei UI", system-ui, sans-serif`
- 数字使用 `font-variant-numeric: tabular-nums`，保证参数输入/滑块数值不跳动。
- 字号：主文本 13px，卡片标题 13–16px，弱文本 11–12px；预设 pill 组副标题 12px。

## 6. 层级

| 层 | z-index | 内容 |
|---|---|---|
| 内容 | 1 | 卡片 |
| 标签关闭/选中 | 3–5 | 标签页内元素 |
| 侧边栏 resizer | 20 | 拖动分隔条 |
| 曲线悬浮窗 | 20 | 频响悬浮卡 |
| 底部行 | 20 | 设备属性 + 曲线卡 |
| 框选工具栏 | 30 | 浮动工具栏 |
| 框选矩形 | 40 | 选区 |
| 滚动条轨道（非对话框） | 50 | 内容页/侧栏 overlay 滚动条 |
| 弹窗 overlay | 60 | Dialog overlay |
| 弹窗内容 | 61 | Dialog content |
| 滚动条轨道（对话框内） | 65 | 安装/保存预设/导入预览/卸载进度 |
| Toast | 70 | 轻提示 |
| 拖拽飞行副本 | 70 | `drag-fly.fly-anim`；portal 进滚动内容层，落在 `.device-body` 的层叠上下文内（该上下文 `z-index:0`，整块被标签栏的 40 压在下面），70 只与卡片(1)/框选盒(40)/染色层(25–36) 比大小 |
| 拖拽悬浮层 | 75 | `drag-fly.overlay-fixed`；同样 portal 进滚动内容层，一起被标签栏与内容区边界挡掉；仍是 `position:fixed`，但包含块是带 `transform` 的 `.tuning-scroll`（跟手需要它不随滚动位移） |
| 窗口缩放手柄 | 9999 | resize handles |

## 7. 颜色映射表（TS）

组色/预设色的浅色 hover、深色基础、深色 hover 均在 `src/lib/blocks.ts` + `src/lib/accent.ts` 预存：

- `ACCENT_HOVER_HEX`：浅色 hover 色
- `ACCENT_DARK_HEX`：深色基础色与深色 hover 色

计算方式：
1. 基础色转 OKLCH。
2. 浅色 hover：`L * 0.75`，`C` 不变；若 RGB 感知绿色占比高，H 向 OKLCH 绿基色相 `142.5°` 微偏。
3. 深色基础：`L = 0.6`，`C/H` 不变。
4. 深色 hover：在深色基础上再执行一次浅色 hover 算法。
5. OKLCH→sRGB 使用色域映射：保持 L/H 不变，二分缩小 C 到 sRGB 色域内，避免橙色系偏红。

## 8. 滚动条令牌

- 自绘 overlay 滚动条：轨道透明，圆头 `--scrollbar` 实色（宽 5px，圆角 9999px），
  hover `--scrollbar-hover`；淡入淡出走 `opacity 0.25s ease`，停止滚动 1.2s 后自动淡出。
- 原生滚动条（备用，`fluentOverlay` 风格）：thumb 同样用 `--scrollbar` /
  `--scrollbar-hover`，hover 过渡 `0.32s cubic-bezier(0.4,0,0.2,1)`。

## 9. 环带（玻璃边缘）令牌（2026-09 起）

环带 = 卡片/面板玻璃边缘的一圈受光描边，由三层叠加：外圈 `conic-gradient` 方向性高光、
底部 Canvas 暗部层（`.vx-edge-shade`）、以及卡片内光在暗部处的"留空"。

| 令牌 | 含义 | 浅色 | 深色 |
|---|---|---|---|
| `--ring-g1` / `--ring-g2` | 对角高光强度（右上 / 右下） | `0.42` / `0.24` | `0.16` / `0.09` |
| `--ring-s1` / `--ring-s2` | 侧边高光强度 | `0.3` / `0.28`（`@property` 初值） | `0.12` / `0.28`（`@property` 初值） |
| `--ring-l` / `--ring-r` | 左 / 右边缘亮度 | `0.08` / `0.04` | `0.035` / `0.018` |
| `--ring-t` / `--ring-b` | 上 / 下边缘亮度 | `0.32` / `0.14` | `0.115` / `0.05` |
| `--ring-m` | 底部暗部强度 | `0.22`（`@property` 初值） | `0.22`（`@property` 初值） |
| `--ring-base` | 基础描边颜色 | `rgba(217,222,231,0.25)` | `rgba(255,255,255,0.035)` |
| `--edge-shade` | Canvas 暗部层颜色 | `rgba(126,136,152,0.9)` | `rgba(8,10,13,0.9)` |

### 9.1 几何注入（`useGlassRing.ts`）

- `applyRingAngles()` 用 `getBoundingClientRect()` 实测宽高，按 `atan2` 计算四个圆角在
  `conic-gradient` 里的角度，注入 `--ang-tl` / `--ang-tr` / `--ang-br` / `--ang-bl`，
  让环绕受光真正落在四个角上。
- 顶部亮线两端衰减按像素换算成角度写入 `--ring-fade`：默认 `13px`，曲线卡 `30px`；
  圆角取 `16px`（`fx-toolbar` 为 `14px`），保证宽卡/窄卡视觉长度一致。
- 底部独立按左下圆角切线收敛，避免复用右上角造成不同宽卡手感不一。

### 9.2 主题插值与暗部层

- `.theme-transition` 对全部 `--ring-*` 数值令牌与 `--edge-shade` 走
  `0.35s cubic-bezier(0.4,0,0.2,1)` 插值，主题切换时环带颜色平滑过渡。
- Canvas 暗部层 `.vx-edge-shade`：主题切换时先用 `0.15s` 关掉，结束后 `0.6s` 淡回。
- 暗部不再单独画黑线：`shade` 控制内光在该处"留空"（`lit=0`），让底层默认高光样式的
  暗部自己透出来——内光弱的地方暗部自然更弱。

### 9.3 染色采样层（`useEdgeTintLayer.ts`）

- 设备卡/曲线卡的环带染色画在低层（`z-index: 25`），悬浮工具栏自己的环带画在高层（`z-index: 35`）。
- 颜色只来自卡片真正带主题色的部分（描边 / chip / 圆点），黑白区域不掺色相；
  曲线 SVG 路径作为离散点光源参与更高层工具栏的染色。
- 光源按 64px 网格分桶，环带事件节流 + 脏区清绘；静置期无常驻重绘与固定全量轮询
  （事件驱动刷新）。
