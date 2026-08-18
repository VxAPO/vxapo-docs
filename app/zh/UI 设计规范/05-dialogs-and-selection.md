# 05 弹窗与框选工具栏

## 1. 弹窗基础

使用 Radix Dialog。基础类名：

- `.vx-dialog-overlay`：`position:fixed; inset:0; background:rgba(10,13,18,0.45); z-index:60`。
- `.vx-dialog-content`：居中，`width:min(420px, calc(100vw - 48px))`，背景 `var(--card)`，边框 `1px solid var(--border)`，圆角 20px，阴影 `0 12px 40px`（深色用 `--shadow-ink` 阴影）。
- 头部 `.vx-dialog-head`、标题 `.vx-dialog-title`、关闭 `.vx-dialog-close`、正文 `.vx-dialog-body`。

## 2. 设置弹窗

- 行 `.vx-setting-row`：名称 + 描述 + 值控件。
- 主题设置三态：浅色 / 深色 / 跟随系统。
- 其他设置项按现有实现为准。

## 3. 安装设备弹窗

- `.install-dialog` / `.install-body` / `.install-list`。
- 每项 `.install-item`：设备名、格式信息、安装按钮。
- 安装按钮 `.install-btn`：圆角胶囊，背景 `var(--brand-soft)`，文字 `var(--brand-deep)`（浅色）/ `var(--brand)`（深色），边框 `1px solid rgba(71,195,209,0.45)`（深色改为 `var(--brand)`）。
- hover：背景 `rgba(71,195,209,0.2)`。
- disabled：`opacity:0.6; cursor:wait`。

## 4. 卸载弹窗

- 危险操作确认。
- 按钮：取消 + 确认卸载（危险色）。

## 5. 保存自定义预设弹窗

- 类名 `.vx-dialog-wide`：`width:min(520px, calc(100vw - 48px))`。
- 字段 `.vx-field`：标签 + 输入框。
- 配色 `.preset-swatches`：24 色固定色盘。
  - 色块 `.preset-swatch`：24×24px 圆，边框 2px transparent。
  - 选中 `.preset-swatch.active`：`border-color: var(--swatch-hover, var(--text-primary)); box-shadow: 0 0 0 2px var(--card)`。
  - 每个色块的 `--swatch-hover` 由 `accentHoverColor(c)` 在 TS 中算好注入。
- 语义描述 `.preset-band-list`：每行显示频段标签 + 描述输入框。

## 6. 通用确认弹窗

- `.vx-confirm-text` 显示确认文案。
- 用于删除自定义预设、关闭通道选择器等不可逆操作。

## 7. 框选工具栏 `.sel-toolbar`

- 当有框选卡片时出现。
- `position:absolute; transform:translateX(-50%); z-index:30`。
- 浅色：`background: color-mix(in srgb, var(--card) 45%, transparent); backdrop-filter: blur(32px) saturate(1.3); border:1px solid color-mix(in srgb, var(--border) 55%, #ffffff)`，顶部高光 `inset 0 1px 0 rgba(255,255,255,0.4)`。
- 深色：`background: color-mix(in srgb, var(--card) 78%, transparent); backdrop-filter: blur(28px) saturate(1.2); border:1px solid rgba(255,255,255,0.16)`，顶部高光降为 `rgba(255,255,255,0.08)`。
- 动画：`sel-toolbar-in 0.22s ease-out`（深色 `sel-toolbar-in-dark`）。

### 7.1 操作按钮

- `.sel-action.save`：保存为自定义预设。背景 `var(--brand-soft)`，文字 `var(--brand-deep)`（浅色）/ `var(--brand)`（深色），边框 `1px solid rgba(71,195,209,0.45)`（深色为 `var(--brand)`）。
- `.sel-action.delete`：删除。背景 `var(--danger-soft)`，文字 `var(--danger)`，边框 `color-mix(in srgb, var(--danger) 45%, transparent)`。
- `.sel-copy-btn`：复制到声道。浅色边框/文字 `#3b3c3f`；深色边框 `var(--border-strong)`、文字 `var(--text-primary)`。
- `.sel-copy-menu`：复制到声道菜单，背景 `var(--card)`，边框 `1px solid var(--border)`。

## 8. Toast `.vx-toast`

- 固定在底部居中：`position:fixed; left:50%; bottom:32px; transform:translateX(-50%); z-index:70`。
- 背景 `var(--accent-bg)`，文字 `var(--on-accent)`，圆角 9999px，padding `8px 16px`，字号 12px。
- 动画 `vx-toast-in 0.2s ease`。
