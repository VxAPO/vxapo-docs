# 04 频响曲线与设备属性卡

## 1. 底部固定行

`.bottom-row`：绝对定位于 `.content` 底部，`left:0; right:0; bottom:0; z-index:20;
display:flex; gap:12px; padding:10px 12px 12px; pointer-events:none`。

子项可交互：

- `.dev-props-card`：设备属性卡
- `.curve-wrap`：频响曲线卡

两者均使用毛玻璃：

- 浅色：`background: color-mix(in srgb, var(--card) 56%, transparent);
  backdrop-filter: blur(40px) saturate(1.4)`。
- 深色：`background: color-mix(in srgb, var(--card) 82%, transparent);
  backdrop-filter: blur(28px) saturate(1.15)`。

## 2. 设备属性卡 `.dev-props-card`

- `flex:none; min-width:240px; border-radius:16px; padding:10px 18px 12px;
  border:1px solid var(--border); box-shadow: var(--card-shadow)`。
- 内容：设备名、采样率/通道/位深、当前增益、段数统计、归一化按钮。
- 按钮 `.dev-prop-btn`：高 20px，padding `0 8px`，圆角 9999px，
  边框 `1px solid var(--border)`，文字 `var(--text-secondary)`；
  hover 边框/文字 `var(--brand-deep)`（浅色）/ `var(--brand)`（深色），背景 `var(--brand-soft)`。

## 3. 频响曲线卡 `.curve-wrap`

- `flex:1; min-width:0; border-radius:16px; padding:10px 18px 14px;
  border:1px solid var(--border); box-shadow: var(--card-shadow); overflow:hidden`。
- 标题行 `.curve-head`：高度 32px，标题“频响曲线”16px 700 + 声道选择器。
- SVG 左移 `margin-left:-10px`，宽度由 `CurvePanel` 根据容器宽度动态计算，最小 660px。
- 曲线内 `path/circle/line` 描边色过渡 `stroke 0.32s cubic-bezier(0.4,0,0.2,1)`（主题切换插值）。

## 4. 曲线绘制（CurvePlot + CurveGrid）

`lib/curve.ts` 提供坐标映射与评估点：

- 频率轴 `logX`：`x = 40 + (log10(f/20)/3) * (w - 80)`（20Hz→40，20kHz→w-40）。
- 增益轴 `dbY`：`y = 24 + ((top - db) / (top - bottom)) * 180`（视口 220 高）。
- 评估点 `buildEvalFreqs`：全局 20Hz–20kHz 对数扫描（481 点）+ 频段中心 +
  高 Q（>3）邻域细化（±2 倍半宽 ×24 点）+ 相邻中心几何中点；按频率升序返回。
- 曲线 path 颜色：`var(--curve-path)`（浅色 `#009aa2` / 深色 `#47c3d1`）。

网格由 `CurveGrid` 绘制：

- Y 网格：步长由跨度定（`>26dB` 用 4dB，否则 2dB，`yStepFor`）。**量程本身由
  `axisRange(peakGain, troughGain)` 对齐到步长整数倍**（先按 2dB 档取整并夹在软边界 6–30，
  再对齐步长，最多迭代两次收敛）——这样网格能从 `yTop` 一路铺到 `yBottom`：首末两条正好压在
  绘图区上下沿，0dB 也必然落在某条刻度线上。
  行位置再经 `snapPx()` 取整：间距常带半像素（如 `180×4/32 = 22.5px`），不取整时文字落在亚像素上
  会被渲染器取整，表现为「刻度数字有概率往下偏」。标签用 `dominantBaseline="middle"` 让垂直中心
  正对网格行，不依赖 `+3` 这类经验偏移。
  虚线 `stroke: var(--border)`，`strokeDasharray="4 4"`。
  （历史弯路：曾在绘制侧用 `floor`/`ceil` 去凑步长整数倍——量程不是步长倍数时首条网格线会缩进来
  半格，虚线便贴不住纵轴顶端、标签整体偏移。）
- X 网格：20/50/100/200/500/1k/2k/5k/10k/20k 对数位置，虚线同上。
- 坐标轴实线：`stroke: var(--curve-axis)`（浅色 `#b3bbc8` / 深色 `#7c7d7f`）。
- 刻度标签与十字光标样式见 `CurveGrid` / `CurvePlot` 实现。

## 5. 性能：节流重算

- `useThrottledCompute`（`THROTTLED_COMPUTE_MS = 42`，≈24fps）：拖动滑块/曲线重算走固定间隔
  重算 + 停止后补算一次；滑块 move 只重渲染被拖卡片，保证拖动帧数。
- RBJ 系数按频段缓存（`rbj.ts bandDbCached`），避免重复计算。

## 6. 光标悬浮窗

类名：`.curve-tip`。绝对定位，`left/top` 由 `useCurveHover` 计算，通过 `transform: translate(x,y)` 定位。

### 6.1 水平避让

- 悬浮窗以左上角为基准。
- 右侧最远停到 12kHz；左侧最远停到 23Hz。
- 基础水平位置：`cx + sign * gH`，`gH` 由屏幕斜率计算，范围 0–10px。
- 最终 `left = clamp(base, x(23Hz), x(12kHz))`。
- 左右贴边侧通过 `hSide` 记录，用于翻转动画。

### 6.2 垂直避让

- 垂直间隙 `gV = 10px`。
- 曲线点上方/下方切换有安全区 `SAFE_RADIUS = 10px`，在安全区内保持原侧，避免抖动。

### 6.3 跟随动画

- **临界阻尼弹簧**（`hooks/useCurveHover.ts`）：`FOLLOW_SETTLE_MS = 240` 视为"基本停稳"的时间，
  角频率取 `omega = 6.6 / (FOLLOW_SETTLE_MS / 1000)`，每帧按弹簧积分推进（非固定比例趋近）。
- **吸附即停**：位置 `FOLLOW_SNAP_PX = 0.5` 与速度 `FOLLOW_SNAP_V = 40`（px/s）同时满足才判定
  停稳，避免浮点残留造成的微抖。
- 基准侧切换时播放 280ms 满速 ease-out 平移动画。

### 6.4 内容

- `.curve-tip-body`：背景 `var(--accent-bg)`，文字 `var(--on-accent)`，圆角 10px，padding `6px 10px`。
- 浅色：无边框。
- 深色：`border: 1px solid var(--border-strong)`。
- 第一行频率：`fmtFreq`（≥1000Hz 显示 kHz，否则 Hz）。
- 第二行增益：`.tip-gain`，颜色 `var(--brand)`。

## 7. 声道选择器

- 参数视图下曲线卡标题行显示声道下拉；通道选择器开启后按声道切换曲线。
- 语义视图下显示“全部声道”且不可用。
