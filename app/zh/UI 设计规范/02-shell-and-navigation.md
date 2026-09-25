# 02 窗口骨架与导航

## 1. 总体结构

`App.tsx` 渲染 `.app-shell-new`，结构如下：

```
.app-shell-new
├── .topbar                      顶部栏
├── .main
│   ├── .sidebar                 侧边栏（预设 / 自定义 / 高级）
│   │   └── .sidebar-scroll > .sidebar-scroll-inner.os-scroll   + OverlayScrollbar
│   └── .content                 主内容
│       ├── .tab-bar             设备标签页
│       ├── .device-body
│       │   ├── .device-page（按设备重挂载：key=selectedGuid + AnimatePresence mode="wait"）
│       │   │   ├── .no-device   空态（无设备）
│       │   │   └── .tuning-scroll.os-scroll
│       │   │       ├── .view-stack > .view-stage（语义/参数视图）
│       │   │       ├── .sel-toolbar  框选工具栏
│       │   │       └── .marquee-box  框选矩形
│       │   ├── OverlayScrollbar（在 AnimatePresence 之外，跨设备存活）
│       │   └── .bottom-row      底部固定行
│       │       ├── .dev-props-card  设备属性卡
│       │       └── .curve-wrap      频响曲线卡
└── 弹窗们
```

- `.app-shell-new`：`display:flex; flex-direction:column; height:100%; padding:0 10px 10px;
  gap:0; background: var(--frame)`。
- `.main`：`flex:1; min-height:0; display:flex; gap:10px`。
- `.content`：`flex:1; display:flex; flex-direction:column; background: var(--content);
  border:1px solid var(--border); border-radius:26px; overflow:hidden`。
- `.content.is-empty`（无设备）：退回 `var(--frame)` 底色，隐藏 `.tab-bar`。
- 滚动区（`.tuning-scroll`、`.sidebar-scroll-inner`、对话框列表）统一加 `.os-scroll`
  隐藏原生滚动条，由 `OverlayScrollbar` 自绘（见 05 §9）。

## 2. TopBar

类名：`.topbar`。高度 48px，横向 flex，`gap:6px`，左右 padding 12px。

内容顺序：

1. `.logo`：VxAPO 图标（`VxAPO_icon_v4.svg`），18×18px。
2. 设置 / 导入 / 导出按钮：`.pill`，高 28px，padding `0 12px`，圆角 9999px。
3. `.spacer`：可拖拽窗口区域（`data-tauri-drag-region`）。
4. 视图切换：`.seg.view-seg`（语义视图 / 参数视图），外面裹着 `.view-seg-zone` 死区，见 2.1。
5. `.spacer`
6. 窗口控制：`.pill.winbtn`，最小化 / 最大化 / 关闭；关闭按钮 hover 使用 `--danger-soft` 底 + `--danger` 文字。

### 2.1 视图切换（语义 / 参数）

类名：`.seg.view-seg`（样式在 `styles/curve.css`），外层是 `.view-seg-zone`。

- **居中与死区都由 `.view-seg-zone` 负责**：`position:absolute; left:50%; top:50%;
  transform:translate(-50%,-50%); display:inline-flex; padding:8px`。它**不挂** `data-tauri-drag-region`，
  于是点歪到控件四周这 8px 圈上时，事件落在这个容器上而不是顶栏/`.spacer` 的拖拽区——
  不会误触发标题栏的双击最大化/还原（死区内也不再能拖着窗口移动，这是有意的取舍）。
- `.seg.view-seg` 自身只是 `position:relative`（给 `.seg-thumb` 当包含块）＋盒模型与配色。
- 容器：`display:flex; align-items:center; gap:4px; height:26px;
  border:1px solid var(--border); border-radius:9999px; background:transparent`。
- 两个按钮 `.view-seg button`：`flex:1; height:22px; padding:0 14px; gap:6px`，字号 12px，带图标。
- 大胶囊 `.view-seg .seg-thumb`：绝对定位，`top:-2px; bottom:-2px; width:calc(50% - 2px)`，
  `background: var(--brand-bg); border:1px solid var(--brand); border-radius:9999px;
  box-shadow: var(--card-shadow)`。
- 右侧态 `.seg-thumb.right`：`left: calc(50% + 2px)`。
- 过渡：`left 0.25s cubic-bezier(0.4, 0, 0.2, 1)`，另带 0.32–0.35s 的
  background-color / border-color / box-shadow 过渡（主题切换插值）。
- 选中按钮：`color: var(--brand)`；图标倾斜动画 `vx-icon-tilt-left/right 0.65s ease`
  （方向由 `data-dir` 决定）。
- 禁用条件：两者都只在 `noDevices` 时禁用。**打开通道选择器不再禁用语义视图、也不再强制切到参数视图**
  ——语义视图同样按声道过滤内容（`PresetView` 的 `visible`），声道切换入口是曲线卡上的选择器，两个视图都能用。

## 3. Sidebar

类名：`.sidebar`。宽度 `clamp(210px, 15vw, 260px)`，`flex:none`，背景 `var(--card)`，
边框 `1px solid var(--border)`，圆角 26px。内部滚动结构：

```text
.sidebar > .sidebar-scroll（overflow:hidden, border-radius:inherit）
  └── .sidebar-scroll-inner.os-scroll（padding: 12px 14px 14px）
        └── OverlayScrollbar（rounded 模式，上下避让 24/26px）
```

### 3.1 三段切换 `.side-seg`

- 三个按钮：预设 / 自定义 / 高级。
- 未选中：`color: var(--text-weak)`，高度 30px。
- 选中：`color: var(--text-primary)`。
- 底部指示条 `.track`：高 2px，`background: var(--border)`。
- 活动指示 `.ind`：高 2px，`background: var(--text-primary)`，
  `transition: left 0.22s cubic-bezier(0.4,0,0.2,1), width 0.22s`。

### 3.2 预设侧（PresetDeck）

- `.preset-list`：`display:flex; flex-direction:column; gap:12px`（flex + gap，无 margin 挤出问题）。
- 条目 `.adv-pill.preset-pill`：高 30px，padding `0 12px`，圆角 9999px，
  背景 `var(--card)`，边框 `1px solid color-mix(in srgb, var(--preset-accent, var(--brand)) 55%, var(--border))`。
- 结构：左侧 8px 圆点 `.preset-dot`（背景组色）+ `.preset-name`（组名 `.p-group` 加粗组色 +
  gap 8px + 组副标题 `.p-sub` 12px `--text-weak`）+ 右侧“已添加” `.preset-added`（12px 弱文本）。
- hover：`box-shadow: var(--hover-shadow); transform: translateY(-1px)`；圆点 `scale(1.25)`。
- 已使用 `.disabled`：`border-color: var(--border)`，圆点/组名转 `--text-weak`，禁用 hover 浮起。

### 3.3 自定义侧

- 与预设侧同款 `.preset-pill.custom`，右侧删除按钮 `.preset-pill-del`
  （20×20px 圆，hover `--danger-soft` 底 + `--danger` 文字）。
- 空态 `.preset-empty`：单行纯文字居中（12px `--text-weak`），无背景图案。

### 3.4 高级侧

- `.adv-list`：纵向排列，间距 6px。
- `.adv-cat`：分类标题，13px 700，颜色 `var(--text-secondary)`，margin `14px 2px 2px`。
- `.adv-pill`：行按钮，高 30px，padding `0 12px`，圆角 9999px，背景 `var(--card)`，
  边框 `1px solid var(--border)`，卡片阴影。
- `.adv-pill .adv-plus`：加号，颜色 `var(--brand-deep)`（浅色）/ `var(--brand)`（深色），
  flex:none，active 时旋转 45°。
- hover：`box-shadow: var(--hover-shadow); transform: translateY(-1px)`。
- disabled：`color: var(--text-weak); border-style:dashed; box-shadow:none`。

### 3.5 侧边栏拖拽分栏

- `.sidebar-resizer`：绝对定位，位于侧边栏右外 `calc(-1 * (--side-gap + 1px))`，宽 10px。
- 悬停/拖动时圆点指示器从 `--text-weak` 切换为 `--brand`。
- 侧边栏宽度变更走 JS 逻辑，CSS 只负责视觉。

## 4. DeviceTabs

类名：`.tab-bar`，位于 `.content` 顶部，高度 38px，padding `0 10px`，margin-top 8px。

每个设备标签 `.tab-item`：

- 活跃 `.tab-item.active`：z-index 3。
- 标签按钮 `.tab-btn`：`padding: 7px 30px`，font-size 13px，font-weight 500，
  border-radius 16px，margin `4px 6px 4px 2px`，宽 100%，文字省略。
- 未选中 hover：`background: var(--frame)`（浅色）/ `var(--tab-inactive)`（深色），
  文字 `var(--text-secondary)`。
- 活跃：`background: var(--tab-active-bg)`，文字 `var(--text-primary)`。

### 4.1 调音启用开关 `.tab-dot`

- 绝对定位在标签左侧（left 10px），13×13px 圆，边框 2px solid `var(--text-weak)`，背景透明。
- 开启 `.tab-dot.on`：背景/边框 `var(--brand)`。
- hover：`border-color: var(--brand-deep)`（浅色）/ `var(--brand)`（深色 off）、
  `var(--brand-deep)`（深色 on）。
- 点击切换该设备调音开关（逐设备记忆，不切换标签）；初次打开即按磁盘状态显示。

### 4.2 关闭标签 `.tab-close`

- 18×18px 圆，绝对定位右侧，hover 背景 `--danger-soft`，文字 `--danger`。

### 4.3 添加设备 `.tab-add`

- 22×22px 圆，`color: var(--text-secondary)`。
- hover：`color: var(--brand-deep)`（浅色）/ `var(--brand)`（深色），背景 `var(--brand-soft)`。
- 过渡：`background-color / color 0.18s ease`。

## 5. 空态

- `.no-device`：设备为空时显示在 `.device-body` 中，居中，背景 `var(--frame)`。
- `.no-device-logo`：256×256，`opacity:0.16; filter:grayscale(1) invert(1)`。
- `.no-device-row`：安装入口按钮。
- `.no-device-plus`：26px 圆，背景 `var(--brand)`，文字 `var(--on-accent)`；
  hover 背景 `rgba(88,205,219,0.75)`；过渡 `background-color / color 0.18s ease`。
- `.no-device-tip`：提示文字颜色由 `--text-primary` 派生
  （`color-mix(in srgb, var(--text-primary) 50%, transparent)`，深色 `--text-weak`）；
  主题切换时加入 `.theme-transition` 豁免列表，跟随根变量插值不慢拍。
