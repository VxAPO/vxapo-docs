# 04 频响曲线与设备属性卡

## 1. 底部固定行

`.bottom-row`：绝对定位于 `.content` 底部，`left:0; right:0; bottom:0; z-index:20; display:flex; gap:12px; padding:10px 12px 12px; pointer-events:none`。

子项可交互：

- `.dev-props-card`：设备属性卡
- `.curve-wrap`：频响曲线卡

两者均使用毛玻璃：

- 浅色：`background: color-mix(in srgb, var(--card) 56%, transparent); backdrop-filter: blur(40px) saturate(1.4)`。
- 深色：`background: color-mix(in srgb, var(--card) 82%, transparent); backdrop-filter: blur(28px) saturate(1.15)`。

## 2. 设备属性卡 `.dev-props-card`

- `flex:none; min-width:240px; border-radius:16px; padding:10px 18px 12px; border:1px solid var(--border); box-shadow: var(--card-shadow)`。
- 内容：设备名、采样率/通道/位深、当前增益、段数统计、归一化按钮。
- 按钮 `.dev-prop-btn`：高 20px，padding `0 8px`，圆角 9999px，边框 `1px solid var(--border)`，文字 `var(--text-secondary)`；hover 边框/文字 `var(--brand-deep)`，背景 `var(--brand-soft)`。

## 3. 频响曲线卡 `.curve-wrap`

- `flex:1; min-width:0; border-radius:16px; padding:10px 18px 14px; border:1px solid var(--border); box-shadow: var(--card-shadow); overflow:hidden`。
- 标题行 `.curve-head`：高度 32px，标题“频响曲线”16px 700 + 声道选择器。
- SVG 左移 `margin-left:-10px`，宽度由 `CurvePanel` 根据容器宽度动态计算，最小 660px。

## 4. 曲线绘制

`CurvePlot.tsx` 计算并绘制 20Hz–20kHz 对数扫频曲线。

- 频率轴映射：`x = 40 + (log10(f/20)/3) * (curveW - 80)`。
- 增益轴映射：`y = 24 + ((top - db) / (top + 16)) * 180`。
- 曲线 path：`stroke: var(--brand-deep)`（浅色）/ `#47c3d1`（深色）。
- 坐标轴实线：`stroke: var(--border-strong)`（浅色）/ `rgba(255,255,255,0.42)`（深色）。
- 网格虚线：`stroke: var(--border)`，`strokeDasharray="4 4"`。
- 光标圆点：`r=3`，填充 `var(--card)`，描边 `var(--brand-deep)`（浅色）/ `#47c3d1`（深色）。

## 5. 光标悬浮窗

类名：`.curve-tip`。绝对定位，`left/top` 由 JS 计算，通过 `transform: translate(x,y)` 定位。

### 5.1 水平避让

- 悬浮窗以左上角为基准。
- 右侧最远停到 12kHz；左侧最远停到 23Hz。
- 基础水平位置：`cx + sign * gH`，`gH` 由屏幕斜率计算，范围 0–10px。
- 最终 `left = clamp(base, x(23Hz), x(12kHz))`。
- 左右贴边侧通过 `hSide` 记录，用于翻转动画。

### 5.2 垂直避让

- 垂直间隙 `gV = 10px`。
- 曲线点上方/下方切换有安全区 `SAFE_RADIUS = 10px`，在安全区内保持原侧，避免抖动。

### 5.3 跟随动画

- 指数趋近：每帧补足剩余距离的 8%（`FOLLOW_FACTOR = 0.08`）。
- 基准侧切换时播放 280ms 满速 ease-out 平移动画。

### 5.4 内容

- `.curve-tip-body`：背景 `var(--accent-bg)`，文字 `var(--on-accent)`，圆角 10px，padding `6px 10px`。
- 浅色：无边框。
- 深色：`border: 1px solid var(--border-strong)`。
- 第一行频率：`fmtFreq`（≥1000Hz 显示 kHz，否则 Hz）。
- 第二行增益：`.tip-gain`，颜色 `var(--brand)`。

## 6. 声道选择器

- 高级视图下曲线卡标题行显示声道下拉；通道选择器开启后按声道切换曲线。
- 预设视图下显示“全部声道”且不可用。
