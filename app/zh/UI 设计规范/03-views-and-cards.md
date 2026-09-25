# 03 视图与卡片

## 1. 视图切换（语义 / 参数）

- 语义视图（`PresetView`）与参数视图（`AdvancedView`）使用 `motion.div`，类名 `.view-stage`。
- 切换动画：`x: ±100%`，duration `0.32s`，ease `easeInOut`。
- **两套视图常驻 DOM**（`components/ViewStage.tsx`）：非当前视图 `display:none`；切换时退场视图加
  `.is-exiting`（绝对定位让出文档流）演完平移，再以 `display:none` 收起。不再使用
  `AnimatePresence` 的挂载/卸载——避免反复重建 31 张参数卡 DOM 造成的首帧尖峰。
- `.view-stage`：`display:flex; flex-direction:column; gap:22px`。
- 两个视图的**可见块判据同源**：`lib/filters.visibleBlockFor`（通道模式开＝当前声道，无声道标识的块归
  首声道；关＝回退首声道）。语义视图**不再是**「非通道模式全部显示」——那样关掉通道选择器时会把别的声道的
  卡也画出来（随后又随合并消失，见 04 第 7 节）。
- 切换编排由 `useViewAnimation` 负责：记录旧滚动位置 → 锁 `.view-stack` 高度 →
  平移结束（`VIEW_SLIDE_MS = 320ms`，按定时器对齐，不等动画回调）的那一刻立即开始
  高度收窄过渡（`cubic-bezier(0.22,1,0.36,1)` 先快后慢；时长按高度差缩放
  `260ms + 2ms/px`，上限 `VIEW_COLLAPSE_MS = 800ms`，差值 < 4px 直接对齐不播动画）；
  收窄终点是**新视图的自然高度**（`min-height` 只在大于内容高度时影响渲染高度，终点给 0 会让小高度差看起来像瞬移）；
  滚动条长度在过渡期间取 max(旧内容, 新内容)，避免先变短再变长。

## 2. 卡片网格

- `.cards.device-cards`：grid 布局。
- 列数：默认 2 列；≥900px 3 列；≥1200px 4 列；≥1700px 5 列。
- 间距 12px，`align-items:start`；组卡 `grid-column: 1 / -1` 通栏。

## 3. 滤波器卡（参数视图）

类名：`.band-card`（参数视图）与组卡内频段行共用 `.band-params` 结构。

- 背景 `var(--card)`，边框 `1px solid var(--border)`，圆角 16px，padding `12px 14px`，卡片阴影。
- 启用态 `.enabled`：边框 `rgba(71,195,209,0.55)`。
- 禁用态 `.disabled`：标题/标签/输入文字 `var(--text-weak)`；`input` 不可交互。

### 3.1 头部 `.b-head`

- `.enable-dot`：数字开关，26×20px，圆角 9999px，边框 2px solid `var(--text-weak)`，
  文字 `var(--text-secondary)`，tabular-nums。
- `.enable-dot.on`：背景/边框 `var(--brand)`，文字 `var(--on-accent)`。
- `.enable-dot:hover`：`border-color: var(--brand-deep)`（浅色）/ `var(--brand)`（深色）。
- `.b-type`：PEAK 类型文字，12px 600。

### 3.2 参数区 `.band-params`

每行 `.band-param-row`：`display:flex; align-items:center; gap:8px`。

- 标签 `.band-param-label`：`var(--text-weak)`，12px。
- 滑块 `.gs-root`（Radix Slider 封装，圆头 `--thumb` + 描边 `--gs-thumb-border`；
  禁用态 `--gs-thumb-disabled`）。
- 输入框 `.gain-input`：宽 56px，高 22px，圆角 8px，边框 `1px solid var(--border)`，
  背景 `var(--card)`，文字 `var(--text-primary)`，右对齐。
- focus-visible：`border-color: var(--brand); box-shadow: 0 1px 3px rgba(15,23,42,0.1)`。

## 4. 组卡（语义视图 / 预设视图）

类名：`.group-card`。组卡使用同一张卡，但按是否有 `group` 追加 `sem-group`。

- `.sem-group`：左侧 3px 组色边条 `border-left: 3px solid var(--card-accent, var(--brand))`。
- `.group-card.enabled`：边框 `rgba(71,195,209,0.55)`。
- `.group-card.enabled.sem-group`：边框 `color-mix(in srgb, var(--card-accent, var(--brand)) 55%, transparent)`。
- `.sem-group .enable-dot.on`：背景/边框 `var(--card-accent, var(--brand))`，文字 `var(--on-accent)`。
- `.sem-group .enable-dot:hover`：边框 `var(--card-accent, var(--brand))`。
- `.sem-group .enable-dot.on:hover`：边框 `var(--card-accent-hover, ...)`（浅色）/
  `var(--card-accent-hover-dark, ...)`（深色）。
- `.sem-chip`：组标签，20px 高，padding `0 8px`，圆角 9999px，
  背景 `color-mix(in srgb, var(--card-accent, var(--brand)) 14%, transparent)`，
  文字 `var(--card-accent, var(--brand-deep))`；hover 背景 24% 混合。
- 组头 `.group-head`：序号圆 `.ord` + 组名（14px 600；可下拉改名 `.sem-select`）+ 开关。

### 4.1 数字开关规则

| 场景 | 浅色 | 深色 |
|---|---|---|
| 无 group off hover | `var(--brand-deep)` | `var(--brand)` |
| 无 group on hover | `var(--brand-deep)` | `var(--brand-deep)` |
| 有 group off hover | 组色 | 深色组色 |
| 有 group on hover | 浅色组 hover 色 | 深色组 hover 色 |

## 5. 效果器卡

类名：`.effect-card`。

- 背景 `var(--card)`，边框 `1px solid var(--border)`，圆角 16px，padding `12px 14px`。
- 启用态 `.effect-card.enabled`：边框 `rgba(71,195,209,0.55)`。
- 头部 `.effect-head`：启用圆点 `.effect-dot`（规则同 `.enable-dot`）+ `.effect-name` +
  删除 `.close-x`。
- `.effect-desc`：描述文字，`var(--text-weak)`。
- 参数行 `.effect-param-row`：标签 + 控件（滑块/下拉/输入框）。
- 语义视图强度行 `.effect-strength-row`：强度百分比 `.g-val`，文字 `var(--text-weak)`。

### 5.1 内置效果器（语义视图强度映射见 `App 引用规范` 第六章）

`preamp`（基准电平）、`wide`（声场处理）、`aural`（谐波激励器）、`reverb`（板式混响）、
`compressor`（压缩器）、`loudness`（等响补偿）。语义强度滑块只映射核心参数：

- `wide`：强度 = 中置空气（距离感），高频补偿留在参数视图手动调。
- `aural` / `reverb`：强度 = 湿声（`wet = 0.9s`、`dry = 1 - wet`，和 ≤ 1）；reverb 额外联动
  decay / damping / pre_delay / room_size。
- `compressor`：强度 = 压缩比（1:1 → 20:1）。
- `loudness`：强度 = 目标响度相对参考响度的下探量。
- `preamp`：强度 = 增益（-24dB → +24dB）。

## 6. 声道胶囊 `.ch-pill`

- **语义视图与参数视图的滤波器分区头部都显示**声道胶囊组（通道模式开启时）：外层 `.col-head`
  （`inline-flex` + `var(--card)` 底 + `padding: 0 12px`），胶囊 `height:24px; padding:0 16px;
  border-radius:9999px`。两个视图共用同一套标记，切换即 `setActiveChannel`（与曲线卡的声道选择器同源）。
- 未选中：`border:1px solid var(--border)`，文字 `var(--text-secondary)`；
  hover 边框/文字 `var(--brand-deep)`（浅色）/ `var(--brand)`（深色）。
- 选中 `.ch-pill.active`：边框/文字 `var(--brand-deep)`（浅色）/ `var(--brand)`（深色），
  背景 `var(--brand-soft)`。

## 7. 卡片删除按钮 `.close-x`

- 18px 圆，hover 背景 `var(--danger-soft)`，文字 `var(--danger)`。

## 8. 空态提示 `.hint-row`

- 当滤波器/效果器为空时显示“从侧栏添加调音”/“从侧栏添加效果器”。
- 可通过 `hintShift` 水平偏移，配合空态居中。
- 错误态 `.hint-row.err`：`color: var(--danger)`。
