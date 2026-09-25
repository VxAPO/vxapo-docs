# 05 弹窗与框选工具栏

## 1. 弹窗基础

使用 Radix Dialog。基础类名：

- `.vx-dialog-overlay`：`position:fixed; inset:0; background:rgba(10,13,18,0.45); z-index:60`。
- `.vx-dialog-content`：居中，`width:min(420px, calc(100vw - 48px))`，背景 `var(--card)`，
  边框 `1px solid var(--border)`，圆角 20px，阴影 `0 12px 40px`（深色用 `--shadow-ink` 阴影）。
- 头部 `.vx-dialog-head`、标题 `.vx-dialog-title`、关闭 `.vx-dialog-close`、正文 `.vx-dialog-body`。

## 2. 设置弹窗

- 行 `.vx-setting-row`：名称 + 描述 + 值控件。
- 主题设置三态：浅色 / 深色 / 跟随系统（`.theme-seg`，240px 宽，thumb 平滑左移）。
- 语言设置：简体中文 / English。

## 3. 安装设备弹窗

- `.install-dialog` / `.install-body` / `.install-list`（`.os-scroll` + OverlayScrollbar）。
- 每项 `.install-item`：设备名、格式信息、安装按钮。
- 安装按钮 `.install-btn`：圆角胶囊，背景 `var(--brand-soft)`，
  文字 `var(--brand-deep)`（浅色）/ `var(--brand)`（深色），
  边框 `1px solid rgba(71,195,209,0.45)`（深色为 `var(--brand)`）；
  过渡 `background-color / color / border-color 0.18s ease`。
- hover：背景 `rgba(71,195,209,0.2)`。
- disabled：`opacity:0.6; cursor:wait`。

### 3.1 安装验证进度面板（`install --verify` 闭环）

- 点击安装后切换为进度视图 `.install-progress`（单设备一次只装一个）。
- 状态机：`installing → restarting ↔ verifying → done | failed`；重试对用户透明
  （restarting/verifying 随事件循环推进，retry 仅换文案）。
- 数据源：App 后端 `install_device` 异步执行 CLI `install --verify`，逐行解析 JSON
  事件并 `emit("install-progress")`；前端 `onInstallProgress` 订阅。
- 展示：
  - `.install-progress-head`：设备名 + 旋转 spinner（运行中）。
  - `.install-progress-track` / `.install-progress-bar`：轨道透明，仅显示进度条；
    installing/verifying 脉冲、restarting 定宽 45%、done/failed 100%。
  - `.install-progress-text`：阶段文案（写配置 / 停服务 / 启服务 / 验证 / 重试 / 成功 / 失败）。
- 终态：done →「完成」按钮关闭并刷新设备列表；failed →「重试 / 完成」，
  重试重新走 `handleInstall`，完成关闭；两种终态都触发设备列表刷新
  （失败时 best 配置已写入，设备按已安装态出现；另有 `rollback_install` 兜底回滚）。
- 返回契约：`InstallResult { success, mode, score, attempts, best_mode, best_score }`。

## 4. 卸载弹窗

- 危险操作确认 + 进度列表（`.op-progress`，`.os-scroll` + OverlayScrollbar）。
- 按钮：取消 + 确认卸载（危险色）。

## 5. 保存自定义预设弹窗

- 类名 `.vx-dialog-wide`：`width:min(520px, calc(100vw - 48px))`。
- 字段 `.vx-field`：标签 + 输入框。
  - 名称 `.vx-text-input`：默认取当前框选内容的推导名，可为空（保存时回退占位名）。
  - 配色 `.preset-swatches`：24 色固定色盘（`SWATCHES`）。
    - 色块 `.preset-swatch`：24×24px 圆，边框 2px transparent。
    - 选中 `.preset-swatch.active`：**环画在盘面之外**——`--swatch-ring: var(--swatch-hover,
      var(--text-primary))` + `box-shadow: 0 0 0 2px var(--swatch-ring)`。**不要**用点亮
      `border-color` 的方式做环：色块是 `<button>`，UA 默认 `box-sizing: border-box`，那 2px 边框
      画在 24px 盒**内**，不透明时会把底下的背景吃掉一圈——盘面直径 24 → 20，观感就是
      「选中之后圆圈变小了」（踩过）。改外圈后盘面尺寸不变（实测选中/未选中盘面同为 22px）。
      原来那圈 `0 0 0 2px var(--card)` 隔离环同时撤掉：它与弹窗底色同色，本来就不起分离作用。
    - **深色下这圈环要再压深一档**（覆盖写在 `dark.css`，只换 `--swatch-ring` 的颜色，几何仍在
      `cards.css`）：
      `--swatch-hover` 是 L 0.45 的加深变体，浅色下压亮底很清楚，但深色下它与填充（L 0.6）**差得不够**，
      描边读不出来。那里掺 32% 纯黑（≈L 0.35），改成靠「亮盘被切掉一圈」的**深浅差**读选中，
      填充保持原色不动、浅色规则也不动。反向方案（深色改用亮色描边）试过并弃用：与「浅色压深、
      深色提亮」的直觉相反，整盘看着发散。
    - 每个色块的 `--swatch-hover` 由 `accentHoverColor(c)` 在 TS 中算好注入。
  - 逐行描述 `.preset-band-list`（`.os-scroll` + OverlayScrollbar）：分区标题用通用的
    `t("filters")`（「滤波器」/「Filters」）；每行 `.preset-band-row` 显示频段标签（`fc · gain`）
    + 输入框，输入框占位文字是 `preset.bandDesc`（「语义描述」/「Semantic description」），
    **默认为空**（不再预填任何内容），保存时按行写入。
    - `padding: 10px 0`：横向**不留余量**——焦点只用描边交代、不往外画光晕，没有会被裁到的东西。
      留个提醒：它是滚动容器，`overflow-y:auto` 会把**横向也一并裁在 padding box 上**，而输入框是
      `flex:1`、右缘正好压着这个边界——**一旦再给焦点加外圈，右侧就会缺一节**（左缘前面有频段标签、
      本来就有余量，所以只有右边看得出来）。要加外圈就得同时留出等宽内缩。
    - 频段标签 `.preset-band-tag`：`width:122px`、`font-size:12px`（与 `.vx-field-label`、输入框同级；
      原先 11px/110px 既字小、列里又空）。列宽按最长标签 `16.00 kHz · +10.5 dB` 在该字号下留余量，
      `tabular-nums` 让数字列对齐。
- 输入框聚焦 `.vx-text-input:focus-visible`：**只改描边色** `border-color: var(--brand)`，**不画外圈光晕**
  ——与 `.vx-select:focus-visible` 同套。外圈试过两种（`0 0 0 3px var(--brand-soft)` 等宽硬环、以及
  1px 细环 + 6px 模糊晕），都不如描边本身干净；而且任何外圈都会被上面那个滚动容器裁掉。
- 保存动作：`onSave(name, desc, color, descriptions)`；简介字段已移除。

## 6. 通用确认弹窗

- `.vx-confirm-text` 显示确认文案。
- 用于删除自定义预设、关闭通道选择器等不可逆操作。

## 7. 框选工具栏 `.sel-toolbar`

- 当有框选卡片时出现。
- 结构：外层 `.fx-toolbar`（`position:absolute; left/top:0; translate:-50% 0; z-index:30`）› 玻璃面板
  `.sel-toolbar` › 内容（`.sel-toolbar-label` + `.sel-toolbar-actions`）。浮窗位移由
  `useMarqueeSelection` 的跟随循环写 `transform` 独占，`translate` 与入场抬升交给 CSS。
- **工具栏尺寸要参与"内缩 8px"的夹取**（与卡片同一个 `TOOLBAR_EDGE = 8`）：横向
  `clamp(cx, 8 + w/2, bodyW − 8 − w/2)`、纵向同理。所以尺寸必须是**真实值**：`ResizeObserver`
  之外还要在挂上时用 `getBoundingClientRect()` **同步量一次**——RO 回调要等下一次布局，
  只靠它的话首次落位会按兜底尺寸（`280×64`）算，而通道选择器（复制到声道）把工具栏拉宽到
  远超兜底值，右边界算得太靠右、浮窗直接顶出内缩线，下一帧才飞回来。
  RO 也必须**盯元素本身**、不拿 `selectedIds.length > 0` 之类做 deps：工具栏要等 `selGeom`
  算出来才挂载（`selGeom` 是被动 effect 的产物，比那一轮 layout effect 晚），按布尔量做 deps
  会在元素还不存在时跑一次、`el` 为 null 直接 return，之后再也不会重跑，尺寸就永久停在兜底值
  ——表现为"刷新后**首次**框选出界、取消一次再框选就正常"（退场动画期间旧元素还挂着，
  effect 那时才补上 RO）。踩过。
- 玻璃（浅色）：底色交给 `.sel-toolbar::before`（`--glass-bg` =
  `color-mix(in srgb, color-mix(in srgb, var(--card) 80%, var(--border)) 50%, transparent)`），
  `backdrop-filter: blur(calc(var(--glass-blur) * var(--glass-t))) saturate(1.1) contrast(0.55) brightness(1.38)`
  （`--glass-blur` 浅色 `12px`、深色 `8px`，深色另覆盖 `--glass-bg` / `--glass-shadow`）；
  投影 `--glass-shadow` 随底衬一起淡；边框透明——描边由外侧环带承担。
- **淡入淡出走 `--glass-t`（0→1），不用 opacity**：面板的 `backdrop-filter` 在祖先 `opacity < 1` 时会被
  浏览器整组降级，观感是「高斯模糊等淡入结束才出现」。`--glass-t` 注册为 `<number>` 才能被 transition
  插值：挂载后下一帧加 `.is-in`，`180ms ease-out` 渐变；模糊半径、`::before` 底衬、内容透明度全部由它驱动。
  内容须 `position:relative; z-index:1` 压在底衬之上（伪元素是定位元素，否则会盖住文字）。
- 高光带同样跟随：`.fx-toolbar::after`（顶/底/侧向 conic 高光）`opacity = var(--ring-op) * var(--glass-t)`
  （`--ring-op` 浅色 `0.4` / 深色 `0.8`）；`.fx-toolbar::before`（内渗模糊层）`opacity: var(--glass-t)`。
- 退出：`usePresence()` 等 `180ms` 跑完再 `safeToRemove()` 卸载——染色 canvas 是独立图层，
  DOM 一消失它会硬消失；绘制侧按 `--glass-t` 缩放透明度，并由 `driveFor()` 显式推动逐帧重绘
  （DOM 只改 class 不产生 MutationObserver 回调，没人推就会"比 DOM 慢一截"）。
- **滤镜要整套归零**：`blur` 之外，`saturate/contrast/brightness` 也必须乘 `--glass-t` 回到中性——
  只缩模糊的话，浅色的 `contrast(0.55) + brightness(1.38)` 仍在生效，淡出到一半会漏出一层发白的底。

### 7.1 操作按钮

- `.sel-action.save`：保存为自定义预设。背景 `var(--brand-soft)`，
  文字 `var(--brand-deep)`（浅色）/ `var(--brand)`（深色），
  边框 `1px solid rgba(71,195,209,0.45)`（深色为 `var(--brand)`）。
- `.sel-action.delete`：删除。背景 `var(--danger-soft)`，文字 `var(--danger)`，
  边框 `color-mix(in srgb, var(--danger) 45%, transparent)`。
- `.sel-copy-btn`：复制到声道。**低对比度背景**：浅色边框/文字 `#3b3c3f`，
  背景 `var(--surface-inset)`；深色边框 `var(--border-strong)`、文字 `var(--text-primary)`；
  hover 背景 `var(--hover)`。高 32px，padding `0 14px`，圆角 9999px。
- `.sel-copy-menu`：复制到声道菜单。**必须 portal 到 `document.body`**——`position: fixed`，
  `left/top/min-width` 由 JS 按触发按钮的视口矩形写，`z-index: 70`（与 `.vx-select-content` 同档）。
  原因：它原先留在 `.fx-toolbar` 内做绝对定位，而 `.fx-toolbar` 的 `z-index: 30` **低于**染色 canvas
  （`Z_TOOL 35` / `Z_TOOL_SHADE 36）与 `.fx-toolbar::after` 高光环带；`.sel-toolbar` 又带
  `backdrop-filter`、自成层叠上下文，菜单在面板内部**无论给多大 z-index 都翻不上去**，于是被那层
  `mix-blend-mode: screen` 的染色整个盖住，看上去就像"底色是透明的"（踩过）。另外两点：位置要用
  rAF 跟按钮（工具栏会被跟随循环写 transform 移动）；外点关闭的判断必须把 `.sel-copy-menu` 也算作
  "内部"，否则点在菜单项上会先卸载菜单、`click` 没有元素可派（复制直接失效）。
  背景 `var(--card)`，边框 `1px solid var(--border)`，圆角 12px。

## 8. Toast `.vx-toast`

- 固定在底部居中：`position:fixed; left:50%; bottom:32px; transform:translateX(-50%); z-index:70`。
- 背景 `var(--accent-bg)`，文字 `var(--on-accent)`，圆角 9999px，padding `8px 16px`，字号 12px。
- 动画 `vx-toast-in 0.2s ease`。

## 9. 自绘 overlay 滚动条（OverlayScrollbar）

滚动容器统一加 `.os-scroll` 隐藏原生滚动条；`OverlayScrollbar` 用 Portal 渲染
`position:fixed` 轨道，不占布局宽度（详见 `App 引用规范` 与 `overlay-scroll.css`）：

- 激活：只响应**真实用户滚动**（滚轮 / 触摸 / 拖圆头）；程序性滚动（视图切换滚动恢复、
  高度收窄）只更新几何，不激活。
- 淡出：停止滚动 1.2s 自动淡出（`opacity 0.25s ease`）；内容不可滚时保留圆头几何、
  只去激活态平滑淡出，淡出中切视图不会“重入”。
- 设备切换：`deviceKey` 变化先淡出，组件本体跨设备存活，新设备挂好后按滚动重新呼出。
- 层级：非对话框 `z-index:50`（低于遮罩 60，被正常盖住变暗）；对话框内 `z-index:65`
  （高于内容 61，保持可见）。
- 各接入点：
  - 内容页 `.tuning-scroll`：`right:-2`、底部避让 26（只避底部圆角）。
  - 侧边栏 `.sidebar-scroll-inner`：`rounded`（上下各避让 24/26）。
  - 安装列表 `.install-list`、保存预设 `.preset-band-list`：圆头 `right:-6`、`z-index:65`。
  - 导入预览 `.import-preview-text`：底部避让 14、`z-index:65`。
  - 卸载进度 `.op-progress`：`z-index:65`。
