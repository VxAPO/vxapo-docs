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
- **量程与曲线路径必须取自同一份快照**：`App` 里 `curveSnap = deferredCurve ?? liveCurve`，
  评估频点与峰值/谷值一次算完；`yTop/yBottom`（`axisRange`）与交给 `CurvePlot` 的
  `blocks` / `evalFreqs` 全部来自它——绘制侧不再自行按当前 blocks 算评估点。
  两者档位一旦不一致就会露馅：曾经是「量程走节流（按设计**滞后一档**）、路径按当前 blocks 现算」，
  于是关掉通道选择器那一帧里路径已是整条链、量程还是单声道的旧值，曲线先按错量程补间一档、
  下一档才回正——现象是「过渡第一次取到错误值，随后恢复」（形状一直是对的，错的是坐标轴，踩过）。
  曲线**宽度**仍每次现算（保证与网格/viewBox 一致、拖拽改宽度不越界），不受这份快照的滞后影响。
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

- 曲线卡标题行的声道下拉：通道选择器**开启后**列出各声道，切换即换曲线（`setActiveChannel`，与全局活动声道联动）；
  **未开启时锁定在首通道（左）**，选择器置灰、不让改（选项列表始终是各声道，不再有「全部声道」这一档）。
- 曲线目标声道**始终是单个声道**：`App` 里 `curveChannel = channelOn ? effActiveChannel : firstChannel`，
  曲线、量程（`axisRange`）与峰值增益都按它取块。**不要**在关闭选择器时把各声道的块叠起来算：叠加出来的
  那条响应在真实链路里并不存在（每段块各自带 `channel`，驱动侧是分开作用的），而且关掉选择器那一瞬间
  会先画出一条「叠加曲线」、等节流那一档才回正——观感就是「过渡第一次取到错误值，然后恢复」（踩过）。
  基准电平照旧：通道模式关闭时配置本身就是「合并后的全局 preamp」（`setChannelPreampMode` 在切换时
  拆分/合并），所以仍按 `target === "all"` 取那一条。
- 关闭选择器时 **preamp 与块一起合并到首声道**：`setChannelPreampMode` 只留无声道标识或首声道的块并抹掉
  声道标识，与 `buildToml` 在 `mode=false` 时的落盘口径**完全一致**（只写首声道的块、不写 `channels`），
  于是内存与文件同一步到位，不存在「界面已关、文件还是分声道」的中间态；非首声道的块连同其频段在此被丢弃
  ——这是「关闭＝真合并」的既定语义。若只在内存里合并 preamp、把块留给 2s 轮询去收，别的声道的卡会先被
  画出来、随后再消失（语义视图尤其明显，踩过）。
- 显示判据**只有一条**：`lib/filters.visibleBlockFor`（通道模式开＝当前声道，无声道标识的块归首声道；
  关＝无声道标识或首声道的块）。两个视图的 `visible`、`App` 的曲线取块都用它；`CurvePanel` 不再自行过滤
  ——它拿到的是 App 那份**曲线快照**里的块（见第 5 节），再按当前声道过一遍会在切声道那一档把曲线滤空。
  判据一旦各写一份，就会出现「参数视图关了、语义视图还全部显示」这类分歧（踩过）。
- 曲线卡在两个视图里都存在，所以**语义视图下同样可以切换声道**（旧文档写的「语义视图下显示全部声道且不可用」
  已废弃：那时通道模式会强制落在参数视图，现在不再限制视图）。语义视图的滤波器分区头部也有同一套
  声道胶囊（见 03 第 6 节），两处入口同源。
