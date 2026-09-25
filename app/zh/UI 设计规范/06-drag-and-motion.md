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

- `.drag-fly`：飞行副本，基础类 `position:relative; width:fit-content`。
- **两层的挂载点不同，层级与裁剪口径相同**：
  - **抓取悬浮层** `.drag-fly.overlay-fixed`：portal 进 `.device-body`（**不是**滚动容器），
    `position:fixed; top/left:0; z-index:75`，阴影 `0 10px 28px`（深色使用 `--shadow-ink`）。
    裁剪靠 `.device-body` 的 `clip-path: inset(0)`——clip-path 会连整棵子树一起裁，包括 fixed 后代。
    它仍是视口定位、也不在滚动容器里，所以**不随内容滚动位移**：指针不动时卡片在视口里也不动
    （光标锁定靠的就是这一点）。
    ⚠️ 反例（踩过）：改用「给容器加 `transform`、让 fixed 以它为包含块」——若那个容器是滚动容器，
    包含块会跟着内容跑，现象是「滚动多少、卡片就偏多少」，跟手直接废掉。
  - **飞行副本** `.drag-fly.fly-anim`：portal 进滚动容器 `.tuning-scroll`，`position:absolute;
    z-index:70`，用**内容坐标**定位（松手后没有跟手约束），滚动跟随由浏览器合成线程完成
    （零延迟，不再抖），被容器的 overflow 裁在内容区边界。
- 层级：两层都落在 `.device-body` 这个层叠上下文里（`position:relative; z-index:0`，整块被设备标签栏
  `.tab-bar` 的 40 压在下面，所以两层都会被标签栏挡住）；在这个上下文内，它们只需压过卡片（1）、
  框选盒（40）与染色层（25–36）。
- **悬浮层跟手位移走 `transform`**：`left`/`top` 固定为 0 作静态基准，JS 每帧写
  `transform: translate()`。逐帧写 `left`/`top` 属于布局属性、每帧都要重新布局；`transform`
  只改视觉位置。刻意不用 `translate3d`/`will-change`——升成合成层后拖动停住时文字栅格与常规层
  不同（同飞行副本「落地交接」的理由）。
- **悬浮层被夹在内容区矩形内**（`.content`，那个大圆角矩形），并留 `DRAG_BOUNDS_INSET_PX = 8`
  的内缩——**不贴死边缘**（阴影与圆角要留余地）：定位时把卡片左上角 clamp 到
  `[bounds.left + 8, bounds.right − cardW − 8] × [bounds.top + 8, bounds.bottom − cardH − 8]`
  （`DragSession.bounds` / `cardW` / `cardH`，抓取时测一次）。指针可以继续往外移——
  **系统光标没法被网页锁住**，能限制的是这张卡片：它到边界就停住，不再往外跑。
  **占位命中按卡片中心算**（`DragSession.curMidX/curMidY`），不按指针——卡片被夹住时指针可能已经
  在内容区外，按指针判会误判成「槽位外」、占位框被甩到末尾，与眼睛看到的卡片位置对不上（踩过）。
  「槽位外＝追加到末尾」仍然可用：把卡片拖到卡片区下方的空白（离所有槽位 48px 外）即可。
  飞行的起点照旧取悬浮层的真实矩形。
- **贴边带动页面滚（边缘自动滚动）**：悬浮层贴住内容区上/下界、且容器那个方向还能滚时，每帧推进
  `EDGE_SCROLL_SPEED_PX_S = 600`（px/s，按帧时长换算，60/120Hz 观感一致；长帧夹在 64ms 内，
  避免从后台切回来猛跳一段）。指针不动也持续滚：`flushFrame` 滚完继续排下一帧，方向不再贴边、
  或那个方向滚到头，就自然停下。滚动本身复用既有的「槽位矩形平移 + 命中重算」，
  悬浮层不随滚动位移（它锁的是指针）。
- 抓取时阴影从 hover 阴影平滑扩大到拖拽阴影：`drag-shadow-lift 0.18s ease-out`。
- 落地灰条 `drag-bar-land 0.4s ease-out`：时长与弧线飞行（`FLY_ANIM_MS`）对齐，保证副本卸载前跑完
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
- 落地动画总时长 `FLY_TOTAL_MS = 530 = FLY_ANIM_MS(400) + FLY_HOLD_MS(100) + FLY_TAIL_MS(30)`：
  0.4s 二次贝塞尔弧线飞行 + 0.1s 无阴影停顿 + 副本卸载前的时序缓冲。
- 位置段只占总时长的 `FLY_MOVE_RATIO = 0.72`（`FLY_MOVE_MS = 288`）：framer 的 `x`/`y` `times` 与
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
- **落地交接**：位置动画跑完（`FLY_MOVE_MS = 288`）后 `FLY_HANDOVER_MS`（+40ms）撤掉 `transform` 与
  `will-change`，让副本从合成层回到常规绘制——合成层的文字抗锯齿与栅格分辨率与常规层不同，整段停在
  合成层上、飞行停下后就看得出"分辨率下降"。交接是幂等的：基准本来就是落点，无需再改坐标。
- **松手不产生布局变更**：落点结算沿用**已生效的槽位**（`d.entered`），不按松手瞬间的指针位置重算——
  否则占位框会在松手那一下跳到尚未确认的槽位，紧接着还要等这记布局动画跑完才起飞，串联起来就是
  松手卡一下。现在占位框停在哪、卡片就落在哪（所见即所得）。`finalizeDrop` 也推进定时器执行，不在
  `pointerup` 回调里同步做「提交重排 + 量落点 + 两次 `flushSync` 渲染」，避免拖住输入管线。
- **飞行副本挂进滚动内容层（滚动跟随零延迟）**：副本 portal 到 `.tuning-scroll`、按**内容坐标**定位
  （`内容坐标 = 视口坐标 - 容器矩形 + scrollTop`，起飞行那一刻换算一次），因此随内容一起滚——
  滚动跟随由浏览器在合成线程完成，没有 JS 参与，也就没有滞后。副作用（有意）：副本越出内容区的
  部分会被裁剪、被标签栏遮住。
  （历史两条弯路：① 先做成 fixed 顶层 + 在 `scroll` 事件里 JS 逐帧补偿，补偿天然晚一帧、滚动快时
  肉眼可见地抖；② 那次补偿方向还写反过——落点是视口坐标，必须**同向**平移，取反会差出两倍滚动量。）
- **跟手与滚动合帧**：`pointermove` / `scroll` 只记录最新状态，定位与命中在 `requestAnimationFrame`
  里每帧结算一次（先平移槽位、再定位与命中）。高频指针（高刷触控板、游戏鼠标）一秒能发几百个
  `move`，逐个处理等于同一帧里反复写样式、反复测矩形。松手前先同步结算未决帧——从悬浮层量到的
  飞行起点必须是指针最后停下的位置。
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
| 视图切换平移 | `0.32s easeInOut`（`VIEW_SLIDE_MS`）；进场只做 x 平移，**不补间整体 opacity**——淡入交给卡片错峰 |
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
| 拖拽飞行 | `530ms = 400ms 弧线（位置段 288ms，比例 0.72）+ 100ms 停顿 + 30ms 缓冲` |
| 拖拽落地灰条 | `0.4s ease-out`（与弧线飞行同长，副本卸载前跑完） |
| 切换时卡片错峰淡入 | 单张时长随距离递减：近端 `260ms`（`STAGGER_FADE_NEAR_MS`）→ 远端 `150ms`（`STAGGER_FADE_FAR_MS`）；卡片从上方 `6px`（`STAGGER_DROP_PX`）**往下落位**，曲线 `EASE_OUT_BACK = cubic-bezier(0.2, 1.5, 0.3, 1)`——过冲约 5% 后利落收回（「往下展一下再回弹」，尾形与 `EASE_OUT_SOFT` 同族，所以收尾不拖；**过冲量由 `P1.y` 定**：1.5 ≈ 5%、1.8 ≈ 12%）；延迟在**开播前一次算好**并按**对角线权重**（行号 + 列号）推：同一反对角线一起起跑，整体沿对角线从左上扫到右下；延迟 = 窗口 × √(权重 / 最大权重)，所以**间隔前疏后密**；窗口 = `min(200ms, 最大权重 × 14ms)`（`STAGGER_WINDOW_MS` / `STAGGER_STEP_MS`） |
| 频响曲线形状过渡 | `320ms cubic-bezier(0.4,0,0.2,1)`（与 stroke 过渡同长）；任意两次路径之间补间，采样点数由 `lib/pathMorph.ts` 对齐 |
| 频响悬浮窗跟随 | 临界阻尼弹簧：`FOLLOW_SETTLE_MS = 240` 视为基本停稳时间，`omega = 6.6 / (settle/1000)`；吸附即停位置 `0.5px`、速度 `40px/s` |
| 频响悬浮窗翻侧 | `280ms ease-out` |
| 框选工具栏玻璃淡入淡出 | `180ms ease-out`（`--glass-t` 0→1；退出同长，等它跑完再卸载） |
| 设备页切换 | 退场 `180ms easeInOut` 淡出（`DEVICE_FADE_MS`），**进场不补间 opacity**（淡入交给卡片错峰）：`AnimatePresence mode="wait"` + `key={selectedGuid}`，两段串行。退场的是**上一轮的旧元素实例**，它带着旧设备的 props 淡出——因此设备页数据（`blocks` / `effects` / `channelOn` / `activeChannel` / `channelNames`）必须由 App 经 `ViewStage` 透传进视图，**不能**让视图直连 store：store 是全局实时的，旧元素一旦订阅它，淡出途中就会渲染成新设备的内容（连页面高度都一起变），整段过渡观感就不对了 |
| 染色 canvas 跟随淡入淡出 | 逐帧按「目标可见度」缩放透明度：工具栏读 `--glass-t`、页面内目标读所在 `.device-page` 的实时 opacity；淡入淡出期间逐帧重绘（`lib/edgetint/renderLoop.ts`） |
| 曲线重算节流 | `42ms`（≈24fps，`useThrottledCompute`） |
| 滚动条淡入淡出 | `opacity 0.25s ease`；停止滚动 `1.2s` 后自动淡出 |
| 主题切换颜色过渡 | `0.35s cubic-bezier(0.4,0,0.2,1)`（`.theme-transition`，结束后移除） |

### 4.1 切换时的卡片错峰淡入

`lib/staggerIn.ts` 的 `playStaggerIn(root)`：取 `root` 内 `.device-cards > *`（网格容器的直接子级，
即卡片本体），按「左上 → 右下」排序（先 `top`，行容差 `8px` 内按 `left`）后依次播
`opacity 0 → 1` + 从上方 `6px` **往下落位**（`STAGGER_DROP_PX`；起始位移必须是**负值**——写成正值
就成了「从下方往上收」，方向反了，曾如此），曲线 `EASE_OUT_BACK = cubic-bezier(0.2, 1.5, 0.3, 1)`：
过冲约 5% 后利落收回。尾形刻意取 `EASE_OUT_SOFT` 那一条（只把 `P2.x` 从 0.36 收到 0.3），
与拖拽/收窄同族——标准 ease-out-back 的尾太软，回弹收得拖沓。两个旋钮是独立的：
**位移**看 `STAGGER_DROP_PX`，**过冲大小**看曲线里的 `P1.y`（1.5 ≈ 5%、1.8 ≈ 12%）。

延迟 = **窗口 × √(对角线权重 / 最大权重)**，权重 = `行号 + 列号`：同一反对角线上的卡片同时刻起跑，
整体推进沿对角线从左上扫到右下。取**开方**而不是线性，是为了让**间隔前疏后密**——头几张拉得开、
尾段快速收束；线性会让整队匀速铺开，观感偏「排队」。窗口 = `min(200ms, 最大权重 × 14ms)`，
张数少时窗口跟着变小。

同一条 √ 曲线（记作 `t`）还同时作用于**单张时长**：`时长 = 近端 + (远端 − 近端) × t`，
即越靠左上落位越慢（`STAGGER_FADE_NEAR_MS = 260ms`）、越靠右下越快（`STAGGER_FADE_FAR_MS = 150ms`）。
尾部因此是「快而密」地收束，而不是和前几张一样拖着走。

所有延迟在开播前**一次算好**（交给 `el.animate` 的 `delay`），动画互相重叠——不是「等上一张跑完
再跑下一张」的串行，后者卡片会一个一个往外蹦，也不符合「错峰」的本意。

（历史两条弯路：① `delay = min(i × 14ms, 200ms)` 这种**封顶**写法会把超出窗口的卡片挤到同一时刻、
断掉顺序关系；② 用**行主序**权重时推进方向会歪——第二行第一列被排到了第一行最后一列之后。）

- **用 WAAPI 命令式播，不重挂载卡片子树**：两套视图常驻 DOM、31 张参数卡刻意不重建
  （见 `ViewStage` 注释），重建一次子树的首帧布局尖峰比这段动画本身贵得多。
- **`fill: "backwards"`**：排在后面的卡片在延迟期间保持第一帧（透明）。少了它，卡片会先整张出现、
  再被动画拉回透明淡入——就是一下可见的闪烁；调用方因此必须用 `useLayoutEffect`（绘制前开播）。
- **整体不补间 opacity**：视图 stage 与设备页 page 进场时 `opacity` 直接到 1（`opacity: { duration: 0 }`）。
  整体淡入与卡片淡入相乘会让卡片永远亮不满、观感发灰。退场不变（设备页仍是 180ms 淡出，
  `AnimatePresence mode="wait"` 等它跑完再挂新页）。
- **触发点**：视图切换看 `viewAnimating` 变 `true`；设备页切换用 rAF 等「新的 `.device-page` 出现」
  （`mode="wait"` 先退旧页），等不到（无设备/加载失败）两秒后放弃。

### 4.2 频响曲线的形状过渡

曲线「自己会变形」：只要路径变了就平滑补间过去（`CurvePlot` 的 `useLayoutEffect`），**不区分场景**
——切声道、开关/增删滤波器、拖频段参数、切设备全走同一条路，调用方不必再特判「该不该动画」。

- **起点取「上一条动画的当前位置」（`commitStyles()`），回退到「上一个目标」**：`commitStyles()` 把动画
  当前值定格进内联样式再取消，所以变形途中目标又变时会从当前位置接着跑，不会跳回旧值——连续变更
  （拖滑块）表现为**指数式平滑跟随**，松手后收敛到准确值。**不能**读 `getComputedStyle(el).d`：
  layout 阶段 React 已写入新 `d`，拿到的是新值本身，补间会退化成「新值→新值」＝完全看不到动画（踩过）。
  同一时刻只允许一条 `d` 动画在跑，新动画启动前取消上一条——**只取消自己起的**，别动 `curve.css`
  里 stroke 的 CSS 过渡；结束/取消时清掉内联 `d`，把控制权交回 React 的 attribute。
- **纵轴量程跨档会先换算坐标系（`remapPathY`）**：量程（`yTop`/`yBottom`）按峰值自适应、每 2dB 跳一档。
  量程一变，同一形状的 y 像素含义就完全不同——补间起点若直接沿用旧像素，等于拿另一套坐标系的形状去插值
  当前坐标系，曲线会先冲出刻度范围再滑回来。先按 dB 换算到新量程，并**夹在绘图区内**。
- **采样点对齐到「关键点并集」（`lib/pathMorph.ts` 的 `alignPaths`）**：`buildEvalFreqs` 在 481 点对数
  网格之外，还会追加每个启用频段的中心频率、高 Q 邻域细化点、相邻中心中点——换声道、开关一个滤波器，
  点数就变了；而 `d` 的插值要求两侧命令序列逐段对应，点数不同时浏览器无法插值，要么直接跳到终值。
  **不能**按 x 均匀重采样到较大点数：细化点是几十个点挤在峰周围，均匀化会把峰削平——观感是「低采样点的
  曲线变成了高采样点的曲线」。正确做法是取两者 x 的**并集**作公共网格（基础网格本就相同，只多出各自的
  细化点）：峰全保留，且两侧 x 一一对应，插值时只有 y 在动，于是表现为「每个关键点各自升降」，
  而不是「整条线被揉」。两侧 x 跨度不一致（改宽度）时返回 `null`，宁可不动画。
- **用 `useLayoutEffect`**：必须在绘制之前接管，否则新 `d` 会先画一帧再被动画拉回去（闪一下）。
- 时长与 `stroke` 过渡同为 `320ms` / `cubic-bezier(0.4,0,0.2,1)`；动画不设 `fill`，结束后回到 React
  写入的 `d`，所以对齐与换算都不会污染最终渲染的曲线形状。

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
