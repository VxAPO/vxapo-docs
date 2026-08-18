# 06 拖拽、框选与动效

## 1. 拖拽卡片

每张可拖拽卡片由 `DragCard` 包裹，类名 `.drag-card`。

- 拖拽手柄 `.drag-bar`：卡片顶部区域，抓取后启动拖拽。
- 拖拽时通过 `setPointerCapture` 捕获指针，避免丢针。

### 1.1 拖拽状态

| 状态 | 类名 | 表现 |
|---|---|---|
| hover | `.drag-card:hover` | `box-shadow: var(--hover-shadow)`；深色额外 `0 0 0 1px rgba(255,255,255,0.05)` |
| 拖拽中 | `.drag-card.is-dragging` | 原位变占位槽：`background: var(--surface-inset); border: 1px dashed var(--border-strong); box-shadow:none; > * { visibility:hidden }` |
| 选中 | `.drag-card.is-selected` | `outline: 2px solid var(--card-accent, var(--brand)); outline-offset:-2px` |
| 选中+拖拽中 | `.drag-card.is-dragging.is-selected` | 去掉 outline，虚线边框改组色 |
| 有组占位 | `.drag-card.is-dragging:not(.is-selected).sem-group` | `border-left-color: var(--card-accent)` |

### 1.2 拖拽副本

- `.drag-fly`：飞行副本，`position:relative; width:fit-content`。
- `.drag-fly.overlay-fixed`：`position:fixed; z-index:75; pointer-events:none`，阴影 `0 10px 28px`（深色使用 `--shadow-ink`）。
- 抓取时阴影从 hover 阴影平滑扩大到拖拽阴影：`drag-shadow-lift 0.18s ease-out`。

## 2. 槽位检测与避让

`useDragSort.tsx` 实现自定义槽位拖拽引擎。

- 进入槽位消抖 `ENTER_DEBOUNCE_MS = 500`。
- 布局动画 `LAYOUT_ANIM_MS = 400`（组外 320ms）。
- 松手落地动画使用二次贝塞尔弧线飞行，`FLY_TOTAL_MS = 430`。
- 避让中的卡片先弹回原位再跳到新位，避免“闪一下/抽搐”。

## 3. 框选

- 在 `.device-body` 上 `pointerdown` 空白处启动框选。
- `.marquee-box`：`position:absolute; z-index:40; border:1px dashed var(--brand-deep); background:rgba(71,195,209,0.1); pointer-events:none`。
- 坐标按 `snapPx()` 取整。
- 框选结果驱动 `.drag-card.is-selected`。

## 4. 动效时长

| 动画 | 时长/曲线 |
|---|---|
| 视图切换 | `0.32s easeInOut` |
| 侧边栏指示条 | `0.22s cubic-bezier(0.4,0,0.2,1)` |
| 视图滑块 thumb | `0.25s cubic-bezier(0.4,0,0.2,1)` |
| 标签 hover/active | `0.15s ease` |
| 卡片 hover 阴影 | `0.18s ease` |
| 弹窗进入 | `0.18s cubic-bezier(0.2,0.8,0.3,1)` |
| Toast 进入 | `0.2s ease` |
| 拖拽阴影抬升 | `0.18s ease-out` |
| 拖拽落位布局动画 | `320–400ms cubic-bezier(0.22,1,0.36,1)` |
| 拖拽飞行 | `430ms` |
| 频响悬浮窗跟随 | 每帧 8% 指数趋近 |
| 频响悬浮窗翻侧 | `280ms ease-out` |

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
