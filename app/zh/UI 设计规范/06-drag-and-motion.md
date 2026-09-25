# 06 拖拽、框选与动效

## 1. 拖拽卡片

每张可拖拽卡片由 `DragCard` 包裹，类名 `.drag-card`。

- 拖拽手柄 `.drag-bar`：卡片顶部区域，抓取后启动拖拽。
- 拖拽时通过 `setPointerCapture` 捕获指针，避免丢针。

### 1.1 拖拽状态

| 状态 | 类名 | 表现 |
|---|---|---|
| hover | `.drag-card:hover` | `box-shadow: var(--hover-shadow)`；深色额外 `0 0 0 1px rgba(255,255,255,0.05)` |
| 拖拽中 | `.drag-card.is-dragging` | 原位变占位槽：`background: var(--surface-inset); border:1px dashed var(--border-strong); box-shadow:none; > * { visibility:hidden }` |
| 选中 | `.drag-card.is-selected` | `outline: 2px solid var(--card-accent, var(--brand)); outline-offset:-2px` |
| 选中+拖拽中 | `.drag-card.is-dragging.is-selected` | 去掉 outline，虚线边框改组色 |
| 有组占位 | `.drag-card.is-dragging:not(.is-selected).sem-group` | `border-left-color: var(--card-accent)` |

### 1.2 拖拽副本

- `.drag-fly`：飞行副本，`position:relative; width:fit-content`。
- `.drag-fly.overlay-fixed`：`position:fixed; top/left:0; z-index:75; pointer-events:none`，
  阴影 `0 10px 28px`（深色使用 `--shadow-ink`）。
- **悬浮层跟手位移走 `transform`**：`left`/`top` 固定为 0 作静态基准，JS 每帧写
  `transform: translate()`。逐帧写 `left`/`top` 属于布局属性、每帧都要重新布局；`transform`
  只改视觉位置。刻意不用 `translate3d`/`will-change`——升成合成层后拖动停住时文字栅格与常规层
  不同（同飞行副本「落地交接」的理由）。
- 抓取时阴影从 hover 阴影平滑扩大到拖拽阴影：`drag-shadow-lift 0.18s ease-out`。
- 落地灰条 `drag-bar-land 0.3s ease-out`：时长与弧线飞行（`FLY_ANIM_MS`）对齐，保证副本卸载前跑完
  （原先 0.45s 超过落地总时长，尾部被硬切）。

## 2. 槽位检测与避让

`useDragSort.tsx` 实现自定义指针级槽位拖拽引擎（常量在 `lib/dragSortTypes.ts`）：

- 进入槽位消抖 `ENTER_DEBOUNCE_MS = 500`（必须长于所有拖拽动画：下界 = `LAYOUT_ANIM_MS +
  ANIM_SETTLE_BUFFER_MS` = 480，避免动画未结束又触发新一轮布局）。
- 布局动画 `LAYOUT_ANIM_MS = 400`（组外 `LAYOUT_ANIM_OUTSIDE_MS = 320`），曲线 `LAYOUT_EASE`。
- 避让中的卡片不会「先弹回原位再跳到新位」（清位移与重排同帧提交），避免“闪一下/抽搐”。
- 松手时若布局动画未结束，多等 `ANIM_SETTLE_BUFFER_MS = 80` 再落地，避免动画被硬切。记账按
  **本次实际使用的时长**（槽位外补位按 320ms 记，不按 400ms），否则动画早已停住、落地却还在等。
- 距所有槽位超过 `OUTSIDE_DIST = 48` 才算真正离开卡片区（未离开原位时“槽位外=末尾”不生效）。
- 落地动画总时长 `FLY_TOTAL_MS = 430 = FLY_ANIM_MS(300) + FLY_HOLD_MS(100) + FLY_TAIL_MS(30)`：
  0.3s 二次贝塞尔弧线飞行 + 0.1s 无阴影停顿 + 副本卸载前的时序缓冲。
- 位置段只占总时长的 `FLY_MOVE_RATIO = 0.72`（`FLY_MOVE_MS = 216`）：framer 的 `x`/`y` `times` 与
  阴影收尾的 `times` 同源换算，阴影从位置停下那一刻才开始收，不再各记一套比例。
- **拖拽中滚动页面**：槽位矩形按滚动增量整体平移（不重新测量——卡片上挂着布局动画的 `transform`，
  量到的是中间态），命中用最后指针位置重算。拖着的卡跟手不动，卡片区在动。
- **落点取提交后的真实位置**：松手先同步提交「清位移 + 重排」（`flushSync`），同一任务内再量被拖卡
  的真实矩形作飞行终点——拖动期间滚过、重排后堆积微移都不会让落点差几像素。
- **飞行只用 transform 合成**：`left`/`top` 作静态基准、路径走 `x`/`y`（合成器友好，不重排也不重绘
  卡片内容），尺寸静态设定（逐帧写 `width`/`height` 会让文字每帧重新折行）。基准与每帧偏移都吸附到
  设备像素栅格、末帧落在吸附后的落点，合成偏移才是整数设备像素，副本文字栅格原点才与静止卡片一致。
- **基准放落点**：副本的 `left`/`top` 直接设成（吸附后的）落点，路径偏移相对落点算且末帧为 `0`。
  反过来（基准在起点、末帧偏移 = 落点差）一旦 framer 在动画结束后用缓存重写一次 `transform`，就会在
  落点上再叠一次偏移：落点漂移、副本卸载时闪回原位。
- **落地交接**：位置动画跑完（`FLY_MOVE_MS = 216`）后 `FLY_HANDOVER_MS`（+40ms）撤掉 `transform` 与
  `will-change`，让副本从合成层回到常规绘制——合成层的文字抗锯齿与栅格分辨率与常规层不同，整段停在
  合成层上、飞行停下后就看得出"分辨率下降"。交接是幂等的：基准本来就是落点，无需再改坐标。
- **松手不产生布局变更**：落点结算沿用**已生效的槽位**（`d.entered`），不按松手瞬间的指针位置重算——
  否则占位框会在松手那一下跳到尚未确认的槽位，紧接着还要等这记布局动画跑完才起飞，串联起来就是
  松手卡一下。现在占位框停在哪、卡片就落在哪（所见即所得）。`finalizeDrop` 也推进定时器执行，不在
  `pointerup` 回调里同步做「提交重排 + 量落点 + 两次 `flushSync` 渲染」，避免拖住输入管线。
- **飞行副本跟随内容滚动**：副本挂在 `fixed` 层（在滚动容器之外），落点是创建那一刻的视口坐标；
  用户一滚，整条弧线就会被内容甩掉。飞行期间监听滚动，按活动视图舞台的位移反向平移副本基准
  （命令式写 `left`/`top`，不重渲染），弧线跟着内容走，终点始终压在目标槽位上。
- **跟手与滚动合帧**：`pointermove` / `scroll` 只记录最新状态，定位与命中在 `requestAnimationFrame`
  里每帧结算一次（先平移槽位、再定位与命中）。高频指针（高刷触控板、游戏鼠标）一秒能发几百个
  `move`，逐个处理等于同一帧里反复写样式、反复测矩形；飞行副本的跟随滚动补偿同样合帧（测舞台矩形
  是强制布局，一帧最多一次）。松手前先同步结算未决帧——从悬浮层量到的飞行起点必须是指针最后停下的位置。
- **曲线单源**：避让曲线 `LAYOUT_EASE` 与视图收窄 `COLLAPSE_EASE` 是同一个常量（`lib/motionEase.ts`
  的 `EASE_OUT_SOFT = cubic-bezier(0.22,1,0.36,1)`），改一处即三处生效（含滚动条长度变形）。
- 上述时长关系由 `lib/dragSortTypes.test.ts` 钉住（消抖下界、落地三段之和、位置段比例、曲线同源、
  CSS 灰条时长 ≤ 飞行时长）。

## 3. 框选（useMarqueeSelection）

- 在 `.device-body` 上 `pointerdown` 空白处启动框选。
- `.marquee-box`：`position:absolute; z-index:40; border:1px dashed var(--brand-deep);
  background:rgba(71,195,209,0.1); pointer-events:none`。
- 坐标按 `snapPx()` 取整。
- 框选结果驱动 `.drag-card.is-selected`。
- **视图/通道切换残留处理**：视图切换时 `AnimatePresence` 可能同时保留退场/进场两个
  `view-stage`，框选命中只在当前 `view-stage` 内测量；框选浮窗在视图切换动画结束后再淡入，
  避免框选状态被残留清除。

## 4. 动效时长

| 动画 | 时长/曲线 |
|---|---|
| 视图切换平移 | `0.32s easeInOut`（`VIEW_SLIDE_MS`） |
| 视图高度收窄（锁高回弹） | 平移结束后立即开始，`cubic-bezier(0.22,1,0.36,1)`（先快后慢）；时长按高度差缩放 **260–800ms**（`2ms/px`，上限 `VIEW_COLLAPSE_MS`），差值 < 4px 直接对齐不播动画 |
| 侧边栏指示条 | `0.22s cubic-bezier(0.4,0,0.2,1)` |
| 视图滑块 thumb | `0.25s cubic-bezier(0.4,0,0.2,1)` |
| 主题三段 thumb | `0.2s cubic-bezier(0.4,0,0.2,1)` |
| 标签 hover/active | `0.15s ease` |
| 卡片 hover 阴影 | `0.18s ease` |
| 加号（无设备页/标签页）/ 安装按钮 hover | `0.18s ease` |
| 弹窗进入 | `0.18s cubic-bezier(0.2,0.8,0.3,1)` |
| Toast 进入 | `0.2s ease` |
| 拖拽阴影抬升 | `0.18s ease-out` |
| 拖拽落位布局动画 | `320–400ms`，曲线与视图收窄同源（`cubic-bezier(0.22,1,0.36,1)`，`lib/motionEase.ts`） |
| 拖拽飞行 | `430ms = 300ms 弧线（位置段 216ms，比例 0.72）+ 100ms 停顿 + 30ms 缓冲` |
| 拖拽落地灰条 | `0.3s ease-out`（与弧线飞行同长，副本卸载前跑完） |
| 频响悬浮窗跟随 | 临界阻尼弹簧：`FOLLOW_SETTLE_MS = 240` 视为基本停稳时间，`omega = 6.6 / (settle/1000)`；吸附即停位置 `0.5px`、速度 `40px/s` |
| 频响悬浮窗翻侧 | `280ms ease-out` |
| 框选工具栏玻璃淡入淡出 | `180ms ease-out`（`--glass-t` 0→1；退出同长，等它跑完再卸载） |
| 设备页切换 | 淡出 / 淡入各 `180ms easeInOut`（`DEVICE_FADE_MS`）：`AnimatePresence mode="wait"` + `key={selectedGuid}`，两段串行。退场的是**上一轮的旧元素实例**，它带着旧设备的 props 淡出——因此设备页数据（`blocks` / `effects` / `channelOn` / `activeChannel` / `channelNames`）必须由 App 经 `ViewStage` 透传进视图，**不能**让视图直连 store：store 是全局实时的，旧元素一旦订阅它，淡出途中就会渲染成新设备的内容（连页面高度都一起变），整段过渡观感就不对了 |
| 染色 canvas 跟随淡入淡出 | 逐帧按「目标可见度」缩放透明度：工具栏读 `--glass-t`、页面内目标读所在 `.device-page` 的实时 opacity；淡入淡出期间逐帧重绘（`lib/edgetint/renderLoop.ts`） |
| 曲线重算节流 | `42ms`（≈24fps，`useThrottledCompute`） |
| 滚动条淡入淡出 | `opacity 0.25s ease`；停止滚动 `1.2s` 后自动淡出 |
| 主题切换颜色过渡 | `0.35s cubic-bezier(0.4,0,0.2,1)`（`.theme-transition`，结束后移除） |

## 5. 亚像素对齐

`src/lib/snap.ts`：

```ts
export function snapPx(v: number): number {
  return Math.round(v * DPR) / DPR;
}
```

所有定位、动画起止坐标、框选矩形、悬浮窗 transform 均经 `snapPx()`，避免小数像素导致文字发虚或 1px 偏移。

## 6. 交互打断原则

- 动画只影响视觉，不阻塞交互。
- 拖拽过程中窗口 blur / Escape 取消拖拽。
- 频响悬浮窗离开 SVG 即消失；翻转动画中途可被新目标打断。
- 视图切换动画中途可被打断（快速连点只触发一次进场/退场，重复点击当前视图不触发动画）。
