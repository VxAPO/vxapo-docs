# 01 设计基础与主题令牌

以 `src/new.css` 为主样式源，`src/App.css` 仅保留旧变量与基础重置。本文只描述当前实际使用的令牌与规则。

## 1. 主题结构

主题由 `document.documentElement.dataset.theme` 驱动，取值为 `light` | `dark`。对应选择器：

- `:root[data-theme="light"]`
- `:root[data-theme="dark"]`

浅色与深色共用同一套组件类名，只切换 CSS 变量。

## 2. 核心颜色令牌（new.css）

### 浅色

| 令牌 | 值 | 用途 |
|---|---|---|
| `--frame` | `#f0f3f6` | 窗口底、TopBar |
| `--content` | `#f7f9fc` | 主内容区 |
| `--card` | `#f7f9fc` | 侧边栏、卡片 |
| `--text-primary` | `#3d4552` | 主文本 |
| `--text-secondary` | `#3d4552` | 次级文本 |
| `--text-weak` | `#69707c` | 弱文本 |
| `--border` | `#d9dee7` | 常规描边 |
| `--border-strong` | `#b3bbc8` | 强描边 |
| `--hover` | `#e3e8ef` | hover 底 |
| `--track` | `#e3e7ee` | 轨道/槽 |
| `--brand` | `#47c3d1` | 品牌青 |
| `--brand-deep` | `#009aa2` | 品牌深青 |
| `--danger` | `#dc3d43` | 危险/删除 |
| `--on-accent` | `#ffffff` | 深色底上的文字 |
| `--accent-bg` | `#3b3c3f` | 激活胶囊底、Toast 底 |

### 深色

| 令牌 | 值 | 用途 |
|---|---|---|
| `--frame` | `#16181b` | 窗口底、TopBar |
| `--content` | `#1d1f22` | 主内容区 |
| `--card` | `#1d1f22` | 侧边栏、卡片 |
| `--text-primary` | `#c1c6cd` | 主文本 |
| `--text-secondary` | `#969ca4` | 次级文本 |
| `--text-weak` | `#717882` | 弱文本 |
| `--border` | `rgba(255,255,255,0.12)` | 常规描边 |
| `--border-strong` | `rgba(255,255,255,0.22)` | 强描边 |
| `--hover` | `#25292e` | hover 底 |
| `--track` | `#22262a` | 轨道/槽 |
| `--brand` | `#00919e` | 深色品牌基础色 |
| `--brand-deep` | `#006265` | 深色品牌 hover/更深色 |
| `--danger` | `#ea5556` | 危险/删除 |
| `--on-accent` | `#c1c6cd` | 深色激活底上的文字 |
| `--accent-bg` | `#43474e` | 激活胶囊底、Toast 底 |
| `--tab-inactive` | `#2f3135` | 标签 hover 底 |
| `--tab-active` | `#45474c` | 活跃标签底 |

### 品牌色规则

- 浅色：`--brand` 用于胶囊底色/描边/主题文本；`--brand-deep` 用于需要更深对比的文字与 hover 外圈。
- 深色：`--brand` 是基础品牌色；`--brand-deep` 只用于 on 状态的 hover 外圈等需要“更深”的场景。
- 声道胶囊 `.ch-pill.active`、安装胶囊 `.install-btn`、保存预设胶囊 `.sel-action.save` 的描边与字体在两种主题下统一用 `--brand`。

## 3. 阴影

### 浅色

- `--card-shadow`：轻量卡片阴影，用于卡片、曲线卡、设备属性卡。
- `--hover-shadow`：卡片 hover 抬升阴影。

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
- 字号：主文本 13px，卡片标题 13–16px，弱文本 11–12px。

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
| 弹窗 overlay | 60 | Dialog overlay |
| 弹窗内容 | 61 | Dialog content |
| Toast | 70 | 轻提示 |
| 拖拽飞行副本 | 75 | drag-fly |
| 窗口缩放手柄 | 9999 | resize handles |

## 7. 颜色映射表（TS）

组色/预设色的浅色 hover、深色基础、深色 hover 均在 `src/lib/blocks.ts` 预存：

- `ACCENT_HOVER_HEX`：浅色 hover 色
- `ACCENT_DARK_HEX`：深色基础色与深色 hover 色

计算方式：
1. 基础色转 OKLCH。
2. 浅色 hover：`L * 0.75`，`C` 不变；若 RGB 感知绿色占比高，H 向 OKLCH 绿基色相 `142.5°` 微偏。
3. 深色基础：`L = 0.6`，`C/H` 不变。
4. 深色 hover：在深色基础上再执行一次浅色 hover 算法。
5. OKLCH→sRGB 使用色域映射：保持 L/H 不变，二分缩小 C 到 sRGB 色域内，避免橙色系偏红。
