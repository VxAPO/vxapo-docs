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
- `.drag-fly.overlay-fixed`：`position:fixed; z-index:75; pointer-events:none`，
  阴影 `0 10px 28px`（深色使用 `--shadow-ink`）。
- 抓取时阴影从 hover 阴影平滑扩大到拖拽阴影：`drag-shadow-lift 0.18s ease-out`。

## 2. 槽位检测与避让

`useDragSort.tsx` 实现自定义指针级槽位拖拽引擎（常量在 `lib/dragSortTypes.ts`）：

- 进入槽位消抖 `ENTER_DEBOUNCE_MS = 500`（必须长于所有拖拽动画，避免动画未结束又触发新一轮布局）。
- 布局动画 `LAYOUT_ANIM_MS = 400`（组外 `LAYOUT_ANIM_OUTSIDE_MS = 320`）。
- 松手时若布局动画未结束，多等 `ANIM_SETTLE_BUFFER_MS = 80` 再落地，避免动画被硬切。
- 距所有槽位超过 `OUTSIDE_DIST = 48` 才算真正离开卡片区（未离开原位时“槽位外=末尾”不生效）。
- 落地动画总时长 `FLY_TOTAL_MS = 430`：0.3s 二次贝塞尔弧线飞行 + 0.1s 无阴影停顿 + 缓冲。
- 避让中的卡片先弹回原位再跳到新位，避免“闪一下/抽搐”。

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
| 拖拽落位布局动画 | `320–400ms cubic-bezier(0.22,1,0.36,1)` |
| 拖拽飞行 | `430ms` |
| 频响悬浮窗跟随 | 每帧 8% 指数趋近 |
| 频响悬浮窗翻侧 | `280ms ease-out` |
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
