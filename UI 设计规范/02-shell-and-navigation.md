# 02 窗口骨架与导航

## 1. 总体结构

`App.tsx` 渲染 `.app-shell-new`，结构如下：

```
.app-shell-new
├── .topbar                      顶部栏
├── .main
│   ├── .sidebar                 侧边栏（预设 / 自定义 / 高级）
│   └── .content                 主内容
│       ├── .tab-bar             设备标签页
│       ├── .device-body
│       │   ├── .no-device       空态（无设备）
│       │   └── .tuning-scroll   滚动区
│       │       ├── .view-stage  预设视图 / 高级视图
│       │       ├── .sel-toolbar 框选工具栏
│       │       └── .marquee-box 框选矩形
│       └── .bottom-row          底部固定行
│           ├── .dev-props-card  设备属性卡
│           └── .curve-wrap      频响曲线卡
└── 弹窗们
```

- `.app-shell-new`：`display:flex; flex-direction:column; height:100%; padding: 0 10px 10px; gap:0; background: var(--frame)`。
- `.main`：`flex:1; min-height:0; display:flex; gap: 10px`。
- `.content`：`flex:1; display:flex; flex-direction:column; background: var(--content); border: 1px solid var(--border); border-radius: 26px; overflow:hidden`。

## 2. TopBar

类名：`.topbar`。高度 48px，横向 flex，`gap:6px`，左右 padding 12px。

内容顺序：

1. `.logo`：VxAPO 图标，18×18px。
2. 设置 / 导入 / 导出按钮：`.pill`，高 28px，padding `0 12px`，圆角 9999px。
3. `.spacer`：可拖拽窗口区域（`data-tauri-drag-region`）。
4. 视图切换：`.seg.view-seg`。
5. `.spacer`
6. 窗口控制：`.pill.winbtn`，最小化 / 最大化 / 关闭；关闭按钮 hover 使用 `--danger-soft` 底 + `--danger` 文字。

### 2.1 视图切换（语义 / 参数）

类名：`.seg.view-seg`。

- 绝对居中于 TopBar：`position:absolute; left:50%; top:50%; transform:translate(-50%,-50%)`。
- 容器为 `display:flex; align-items:center; gap:4px; height:26px; border:1px solid var(--border); border-radius:9999px; background:transparent`。
- 两个按钮 `.view-seg button`：`flex:1; height:22px; padding:0 14px; gap:6px`，字号 12px。
- 大胶囊 `.view-seg .seg-thumb`：绝对定位，`top:-2px; bottom:-2px; width:calc(50% - 2px)`，比小胶囊上下各多 2px；`background: var(--brand-bg); border:1px solid var(--brand); border-radius:9999px; box-shadow: var(--card-shadow)`。
- 右侧态 `.seg-thumb.right`：`left: calc(50% + 2px)`。
- 过渡：`left 0.25s cubic-bezier(0.4, 0, 0.2, 1)`。
- 选中按钮：`color: var(--brand)`；图标倾斜动画 `vx-icon-tilt-left/right 0.65s ease`。
- 禁用条件：语义视图按钮在 `channelOn || noDevices` 时禁用；参数视图按钮在 `noDevices` 时禁用。

## 3. Sidebar

类名：`.sidebar`。宽度 `clamp(210px, 15vw, 260px)`，`flex:none`，背景 `var(--card)`，边框 `1px solid var(--border)`，圆角 26px。

### 3.1 三段切换 `.side-seg`

- 三个按钮：预设 / 自定义 / 高级。
- 未选中：`color: var(--text-weak)`，高度 30px。
- 选中：`color: var(--text-primary)`。
- 底部指示条 `.track`：高 2px，`background: var(--border)`。
- 活动指示 `.ind`：高 2px，`background: var(--text-primary)`，`transition: left 0.22s cubic-bezier(0.4,0,0.2,1), width 0.22s`。

### 3.2 预设侧

- `.cards` 纵向排列，间距 12px。
- 预设卡 `.preset-card`：背景 `var(--card)`，边框 `1px solid var(--border)`，圆角 16px，padding `10px 12px 12px`；左侧 3px 骑边色条由 `box-shadow: inset 3px 0 0 var(--preset-accent, transparent)` 实现。
- 已使用态 `.preset-card.used`：`opacity:0.55; filter:grayscale(0.8); cursor:not-allowed`。
- 添加按钮 `.add`：高 24px，padding `0 12px`，圆角 9999px，背景 `var(--preset-accent, var(--brand))`，文字 `var(--on-accent)`；hover `filter:brightness(1.08)`。

### 3.3 自定义侧

- 同样使用 `.preset-card`。
- 空态 `.preset-card.preset-empty`：文字居中，显示“暂无自定义预设，框选卡片后可保存”。
- 删除按钮 `.preset-del`：危险色文字，hover 危险软底。

### 3.4 高级侧

- `.adv-list`：纵向排列，间距 6px。
- `.adv-cat`：分类标题，13px 700，颜色 `var(--text-secondary)`，margin `14px 2px 2px`。
- `.adv-pill`：行按钮，高 30px，padding `0 12px`，圆角 9999px，背景 `var(--card)`，边框 `1px solid var(--border)`，卡片阴影。
- `.adv-pill .adv-plus`：加号，颜色 `var(--brand-deep)`（浅色） / `var(--brand)`（深色），flex:none，active 时旋转 45°。
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
- 标签按钮 `.tab-btn`：padding `7px 30px`，font-size 13px，border-radius 16px，margin `4px 6px 4px 2px`，宽 100%，文字省略。
- 未选中 hover：`background: var(--frame)`（浅色）/ `var(--tab-inactive)`（深色），文字 `var(--text-secondary)`。
- 活跃：`background: color-mix(in srgb, var(--frame) 98%, #000000)`（浅色）/ `var(--tab-active)`（深色），文字 `var(--text-primary)`。

### 4.1 调音启用开关 `.tab-dot`

- 绝对定位在标签左侧，13×13px 圆，边框 2px solid `var(--text-weak)`，背景透明。
- 开启 `.tab-dot.on`：背景/边框 `var(--brand)`。
- hover：`border-color: var(--brand-deep)`（浅色）/ `var(--brand)`（深色 off）、`var(--brand-deep)`（深色 on）。
- 点击切换该设备调音开关，不切换标签。

### 4.2 关闭标签 `.tab-close`

- 18×18px 圆，绝对定位右侧，hover 背景 `--danger-soft`，文字 `--danger`。

### 4.3 添加设备 `.tab-add`

- 22×22px 圆，`color: var(--text-secondary)`。
- hover：`color: var(--brand-deep)`（浅色）/ `var(--brand)`（深色），背景 `var(--brand-soft)`。

## 5. 空态

- `.no-device`：设备为空时显示在 `.device-body` 中，居中，背景 `var(--frame)`。
- `.no-device-logo`：256×256，`opacity:0.16; filter:grayscale(1) invert(1)`。
- `.no-device-row`：安装入口按钮。
- `.no-device-plus`：26px 圆，背景 `var(--brand)`，文字 `var(--on-accent)`；hover 背景 `var(--brand-deep)`（浅色）。
