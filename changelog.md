# Changelog

## UI 设计规范 05 — 2026-08-13

变更类型：`文档新增（设计定稿）+ CLI 功能`——WinUI 3 与 driver 集成定调；driver 无变更。

- **《05 Tauri 与 driver 集成定调.md》**：文件系统解耦总纲（UI 只写
  `config.toml`、driver 只读配置）；仅安装 / 卸载 / 状态查询走 CLI；CLI `--json`
  契约（list/status/install/uninstall）；Tauri 2 + React 技术约定；非目标
  （不做 bridge / 管道 / 服务）。
- **CLI v0.3.0 `--json`**：list/status 输出设备 JSON 数组（index/name/guid/
  installed_version/install_mode/slots/eapo/lost_slot）；install/uninstall 输出
  ok/error JSON；错误约定 `{"ok":false,"error":"…"}` 退出码 1；交互菜单内部调用
  保持人类可读输出。
- **02/03/04 引用同步**：05 = WinUI 集成定调，交互细则顺延至 06。
- **v2 修订（2026-08-13）**：回迁 Tauri——05 改为「Tauri 与 driver 集成定调」
  （原子写 config.toml + CLI `--json` + `runas` 提权安装）；CLI `--json` 保留复用。
- **模块引用规范（无详细模块版）.md**：版本号保持 v9.16（设计定稿不升 driver 版本）。

> 对应 commit：driver `无变更` / cli `26a2ad9` / docs `7017e3b`

## UI 设计规范 04 — 2026-08-13

变更类型：`文档新增（设计定稿）`——规范样式描述（参数 × 文案）；driver / cli 无变更。

- **《04 规范样式描述.md》**：串联 01/02/03，逐 UI 部分给出**参数 × 文案**
  规范性描述——顶栏、侧边栏（预设/自定义/高级）、设备标签页与安装页、卡片列与
  通道选择栏头、EQ 曲线单元、下边栏、通用弹窗/确认/toast；非纯代码、不写交互。
- **02/03 后续引用同步**：04 定位为规范样式描述，交互细则顺延至 `05`；
  03 内「细则留 04」引用改为 05。
- **v2 修订（2026-08-13）**：侧边栏栏头改为**同行分段切换器**（预设｜自定义｜高级），
  卡片按列排布；品牌青**极度克制**（点缀为主，主视觉强对比黑白，弃用弱对比灰混主题色）；
  调音滑块**竖排**；每个 name = 单 band（不做一 name 多段）；高级 peak 卡竖排
  Fc → Q → Gain 滑块 → Gain 数值；预设语义卡 name 强 / 副标弱 / 竖滑块；
  所有卡片头带顺序标号（可拖拽重排，交互归 05）。
- **v3 修订（2026-08-13）**：侧边栏顶部加「效果」大标题 + **下划线分段切换器**
  （灰字选中变黑、灰条对应段变黑、点击左右滑动）；组卡宽度 **fit-content**
  （随子卡总宽收缩）；所有卡片右上角 X 删除；「从预设栏添加调音」改为**空态**
  （仅卡片列为空时居中显示，无虚线框）；标签页活跃 = **页面底色**外圈
  （`#F6F7F9` 系），非活跃 = 框架底色胶囊；**边框分层**——顶栏 + 侧边栏 +
  软件状态区同一框架底色，软件状态额外上边线，内容区与框架的直角分界线改
  **圆角分界线**；原全宽底栏取消，设备信息并入频响曲线卡片标题行。
- **v4 修订（2026-08-13）**：删除侧边栏「效果」标题；软件状态上边线**与侧边栏
  两边对齐**；标签页改为**反圆角标签页**（对齐 vxapo-app App.css `shape()`
  实现）——活跃标签背景 = 页面底色，与调音卡片、频响曲线同一颜色页面，
  体现设备与调音一体感；内容区全幅页面底色，**不用圆角矩形包裹**。
- **v5 修订（2026-08-13）**：频响曲线设备信息仅 `{通道数}ch · {采样率}Hz ·
  {位深}bit`（不含设备名）；曲线声道下拉去掉「声道」文字标签；曲线改**标准网格**
  （Y 轴 `+6/+3/0/-3/-6/-10/-16` 虚线、X 轴 `20/50/100/200/500/1k/2k/5k/10k/20k`
  对数虚线）；活跃标签最左侧反圆角转直线垂直左边框、圆角缩小至 6px、整边加 1px
  描线；修复卸载 / 通道管理弹窗样式；「管理」移出调音卡片，只出现在通道选择
  栏位最右侧（仅 selector on 显示），栏位 UI 同侧边栏下划线分段切换器。
- **v6 修订（2026-08-13）**：修正活跃标签衔接——不是去掉左反圆角，而是**标签页
  整体右移**（18px）后底部反圆角衔接内容矩形顶边；矩形顶边在标签左侧直线延伸至
  侧边栏（垂直直角，无圆弧）；「凸字形」上边框描线，标签底边与矩形顶边重合段
  不描线；活跃标签背景 = 页面底色（修复此前 outline 伪元素覆盖成描边色的问题）；
  反圆角放大至清晰可见。
- **v7 修订（2026-08-13）**：反圆角方向修正——底角为**向内凹（notch）**，
  对齐 vxapo-app App.css `shape()` 的 `curve to … with …` 语义（此前误画成
  外凸圆角矩形）；描线沿凹弧走，底边直线段不描；右移保持 18px。
- **v8 修订（2026-08-13）**：反圆角改用**现成伪元素圆形切口方案**（经典
  跨浏览器做法）——活跃标签三边描线（`border` + `border-bottom: none`），
  底角两个与标签栏同色圆形伪元素做内凹切口；替代手绘 SVG；右移 18px、
  底边重合段无线。
- **v9 修订（2026-08-13）**：标签页实现**直接复用 vxapo-app App.css 原套**——
  `.tab-bar` / `.tab-group` / `.tab-item` / `.tab-btn` / `.tab-sep` 及
  `.tab-item.active .tab-btn` 的 `shape()` 原样，不再自绘；预览同步该实现。
- **v10 修订（2026-08-13）**：描线改为 **1px 向上阴影**（`box-shadow: 0 -1px 0 0`
  `--border-strong`）——活跃标签与内容矩形顶边各一条；**标签页图层在矩形之上**，
  矩形阴影不盖到标签页；整体阴影呈现「凸字」描线效果。
- **v11 修订（2026-08-13）**：阴影改为**跟随反圆角**——同形无文字层 +
  `drop-shadow` 上 / 左 / 右三向各 1px（`box-shadow` 不跟剪裁形状，弃用）；
  活跃标签左右两侧也有阴影；**非活跃标签图层低于内容矩形**。
- **v12 修订（2026-08-13）**：标签阴影**加深加硬**——新增 `--tab-edge`
  （浅色 `#5C6470` / 深色 `#C9CED8`），上 / 左 / 右三向 drop-shadow 各**双叠一层**；
  **矩形顶边不再画线**（移除 box-shadow）。
- **v13 修订（2026-08-13）**：修复阴影不生效——`filter` 与 `clip-path` 同层时
  阴影会被剪掉；改为**嵌套层**：外层无剪裁挂 drop-shadow（上 / 左 / 右三向双叠
  `--tab-edge`），内层 `clip-path` 同形，阴影跟随反圆角。
- **模块引用规范（无详细模块版）.md**：版本号保持 v9.16（设计定稿不升 driver 版本）。

> 对应 commit：driver `无变更` / cli `无变更` / docs `26ce1ab`

## UI 设计规范 03 — 2026-08-13

变更类型：`文档新增（设计定稿）`——整体布局定调；driver / cli 无变更。

- **《03 整体布局定调.md》**：五区页面骨架（顶栏 / 侧边栏 / 内容块 / 下边栏 +
  设备标签页体系）ASCII 线框；顶栏 9 项、侧边栏三栏头（预设 / 自定义 / 高级）、
  设备标签页与 `+` 安装页、预设 / 高级视图内容差异表、通道选择器 off/on 状态机、
  每声道独立 ≤31 预算、下边栏状态项。
- **契约修订登记**：01 的 `3.1`（channels 语义：selector off 缺省应用全部 / on
  显式声明）与 `3.2`（按声道分组 ≤31）待随 04 同步修订；driver v9.16 全局合计
  校验改为按声道分组，另立任务实现。
- **v2 修订（2026-08-13）**：明确通道选择器入口——开启 = 侧边栏高级栏
  「通道选择器」项（添加即 on），开启后卡片列顶部出现通道选择栏头；关闭 =
  栏头最右「通道管理」→「关闭通道选择器」。02 图形理念同步明确：**决策 /
  交互元素 = 胶囊，信息布局 = 圆角矩形**。
- **模块引用规范（无详细模块版）.md**：版本号保持 v9.16（设计定稿不升 driver 版本）。

> 对应 commit：driver `无变更` / cli `无变更` / docs `8ac2d91`

## UI 设计规范 02 — 2026-08-12

变更类型：`文档新增（设计定稿）`——UI 图形与使用方法定调；driver / cli 无变更。

- **《02 UI 图形与使用方法定调.md》**：定调图形铁律（可点控件 = 胶囊、
  承载容器 = 圆角、嵌套半径 24/16）、Bento Grid 布局（4→2→1 列、gap 16px、
  1×1/2×1/2×2 变跨）、组件词汇表（胶囊组 / 圆角组）、使用方法
  （预设 / 高级 / 底部固定区 / 导入导出）、shadcn 落地约定与 UX 反馈基调；
  依据 ui-ux-pro-max Bento 模式 + 项目 v1 令牌。
- **02 定位调整**：原规划“02 预设视图交互细则”改为图形与使用方法定调，
  交互细则整体移入 03。
- **v2 修订（2026-08-12）**：色彩体系改为品牌青重建——基准 `#5ED4C9`
  （HSL 170–177° / S≈60），深浅变体取自 logo 资产（`#26827B` / `#47D1CA` /
  `#A6E5DE`），不再沿用 v1 `accent-green`。
- **v3 修订（2026-08-12）**：采纳技能推荐的深靛蓝暗色表面方向
  （背景 `#0F0F23` / muted `#27273B` / border `#4338CA`，橙色弃用）与
  Plus Jakarta Sans（本地打包 + 雅黑中文回退）；品牌青仍为唯一强调色。
- **v4 修订（2026-08-12）**：方向修正——**深靛蓝方向弃用**；主题回归
  白 / 黑基底微偏调（浅色 `#F5F5F7` 系 / 深色 `#202020` 系，均非纯色）；
  品牌青明确定位为**指引色**（强调 / 选中 / 高亮，不主导主题）。
- **v5 修订（2026-08-12）**：新增「主题令牌（品牌色延申定义）」——按示例结构
  （基底 / 表面 / 品牌浅背景 · 正文 / 品牌文字 / 次要文字 · 品牌描边 / 中性描边）
  以 `品牌色 × 权重 + 底色` 推导出浅色 / 深色两套令牌（浅色 `#FFFFFF/#F9FAFB/
  #E7F9F7/#0F1115/#26827B/#61666B/#B7ECE6`，深色镜像）；浅色品牌文字必须用
  `--brand-deep` 保证对比度。
- **v6 修订（2026-08-12）**：品牌色最终定版 `#33CCCC` / `#009AA2`
  （HSL 180–183°，S 60–100）；主题令牌按新品牌色重算（浅色品牌背景 `#E0F7F7`、
  品牌描边 `#A3E8E8`，深色镜像 `#081F1F` / `#0F3D3D`）；浅色品牌文字 `#009AA2`
  约 3.4:1，小号正文用文本专用加深 `#006F75`（非 logo 色）。
- **模块引用规范（无详细模块版）.md**：版本号保持 v9.16（设计定稿不升 driver 版本）。

> 对应 commit：driver `无变更` / cli `无变更` / docs `48c05e2`

## v9.17 — 2026-08-11

变更类型：`缺陷修复 + 结构重构 + 文档同步`（代码审查 13 项阻断级整改 + 依赖治理）

- **RT 零分配逻辑保证**：`object/apo/process.rs` 消除过渡/正常路径每帧
  `Box::new(Chain::new())` 占位分配（字段拆借用）；过渡缓冲 `Vec::resize` 改为
  容量断言 + `[..n].fill(0.0)`——**对应章节**：`object 7.1.11`、`主规范 十一`
- **引擎违约越界防护**：新增 `checked_interleaved_slice` 统一切片构造（先 clamp
  `frames ≤ max_frame_count` 再建切片），process 各分支与 panic 兜底补指针判空；
  `pipeline/process.rs` 同样先 clamp 再处理——**对应章节**：`object 7.1.11`、
  `pipeline 4.6`
- **控制型 COM 入口 panic 兜底**：`Reset/GetLatency/GetInputChannelCount/
  LockForProcess/UnlockForProcess` 统一 catch_unwind，控制路径锁改为
  `into_inner` 容忍中毒；`unsafe impl Send/Sync` 补 SAFETY 注释——**对应章节**：
  `object 7.1`、`主规范 十五`
- **热重载防覆盖修复**：`reloading` 在短锁检查后立即置位、RAII 保证任意提前返回
  路径复位；`diag_append` 全部移出 inner 锁（R5），`diag.log` 加 1MB 轮转——
  **对应章节**：`object 7.1.18`
- **事务回滚真实生效**：`DeleteKey` 回滚改 `delete_tree`（原“打开后传全路径”静默
  空操作）；新建安装信息区登记回滚；卸载删除信息区失败返回 Err；重启/停服失败
  改 best-effort 日志——**对应章节**：`install 5.5.2`
- **sys/registry 质量修复**：`.reg` 导出 MULTI_SZ 改 UTF-16LE hex(7) 双终止、
  QWORD 改完整 8 字节 hex(b)、根头按 root 参数输出、读值失败记警告；
  `delete_sub_key` 改相对句柄语义并修正调用方；补 `write_qword`——**对应章节**：
  `sys 3.4`
- **对象层护栏**：`object/factory.rs` 空指针先校验后写入、`lock_decrement` 零值
  CAS 保护；`aggregate.rs` x64 偏移加编译期断言并补 outer 契约 SAFETY——
  **对应章节**：`object 7.5/7.2`
- **死代码与探针清理**：删除全部 `*_probe.txt` 写盘探针（factory/dll_exports/
  apo/config/init/process）；删除 `parse_{aural,loudness,maximizer,wide,reverb}
  _params` 及旧命令解析测试；`pipeline/process.rs` 静音检测死分支删除——
  **对应章节**：`pipeline 4.x`、`object 7.x`
- **规范总表修订 + 单一事实源**：主规范第十一节按实际代码补录 10+ 行（含脚本新
  发现的 audiodg/select/state/apo_types 等缺口）；新增第三方依赖登记、D1–D8
  依赖铁律与自动校验说明；四份子规范引用约束总表改为指向主规范——
  **对应章节**：`主规范 十一`
- **依赖校验脚本**：新增 `vxapo-driver/scripts/check_deps.ps1`（按文件白名单
  断言 crate::/super:: 引用，忽略 cfg(test)，与总表三文件联动）——
  **对应章节**：`主规范 11.3`
- **测试**：全量 435 passed / 0 failed（注册表/回滚新测试在真实环境验证），
  release 零警告构建
- **模块引用规范（无详细模块版）.md**：版本号 v9.16 → v9.17。

> 对应 commit：driver `c05efde` / cli `无变更` / docs `待提交`（docs hash 按下
> changelog-rule v8.3 随下次自然变更本地回填）

## v9.16 — 2026-08-11

变更类型：`契约落地 + 文档同步`（UI 卡片模型段数契约落地）

- **单块段数下限 6 → 1**：`MIN_PEQ_BANDS = 1`，允许 1 段卡 / 无组裸 band
  （UI 设计规范 01 卡片模型）——**对应章节**：`PEQ 设计文档 2`、
  `config TOML 设计文档`、`config 模块规范`
- **全局段数校验**：所有 peq 块 band 合计 ≤ 31（预设块 + 无组裸 band 共享预算），
  超限整文件解析失败并保留旧链（`total 'peq' bands count N exceeds max 31`）——
  **对应章节**：`config 模块规范`、`intent.md`
- **测试**：全量 445 passed（4 个既有管理员权限用例除外），新增单段接受、
  全局 32 拒绝、全局 31 放行 3 个用例
- **UI 设计规范 01 同步**：移除“待落地”标注，契约调整标记 v9.16 已落地
- **模块引用规范（无详细模块版）.md**：版本号 v9.15 → v9.16。

> 对应 commit：driver `f95d8f8` / cli `无变更` / docs `a471c21`

## UI 设计规范 v1 — 2026-08-11

变更类型：`文档新增（设计定稿）`——首篇 APP-Driver 集成定调文档；driver / cli 无变更。

- **新建 `UI 设计规范/` 文件夹**：旧三 TAB 草案归档至 `UI 设计规范/_归档/`
  （设计令牌保留复用）。
- **《01 APP-Driver 集成与预设·图形EQ定调.md》**：定调 APP↔Driver 数据契约
  （块 ↔ `[[effects]] peq`、name/group 仅 APP、保序、enabled 仅卡片级）、
  预设视图 / 高级视图**双视图**语义（非模式）、拖拽成组与自定义预设闭环、
  TOML 导入双选项（完整 / 仅结构，丢弃 fc）、31 段全局预算。
- **Driver 契约调整登记（实现另立任务）**：`MIN_PEQ_BANDS` 6→1；新增全局
  peq band 总数 ≤ 31 校验。
- **模块引用规范（无详细模块版）.md**：版本号保持 v9.15（设计定稿不升 driver 版本）。

> 对应 commit：driver `无变更` / cli `无变更` / docs `d2cbee6`

## v9.15 — 2026-08-11

变更类型：`缺陷修复 + 简化重构`（脏静音缓冲自我反馈爆音修复 + 热重载回归 EAPO 语义）

- **脏静音缓冲自我反馈爆音修复（“浏览器音效菜单嗡声”根因闭环）**：引擎在流切换/
  静音时会发 `BUFFER_SILENT` 标志但**复用上一帧缓冲**（内存残留本 APO 上一帧输出）；
  旧实现把 SILENT 当有效数据处理 → 输出再作为下一帧输入 → 自我反馈放大（日志实证
  输出峰值 17–95401 倍满刻度，PostMix 直通实例同样记录）。v9.15 起 SILENT 标志权威：
  **去交织前按全零填充、绝不读取残留内容，输出强制清零 + BUFFER_SILENT**（对齐 EAPO
  C11 `memset` 清零语义）；热重载过渡路径同规则。真机验证：浏览器音效菜单开关不再
  嗡声/爆音——**对应章节**：`pipeline 4.2`、`object 7.1.11`、`Equalizer 行为文档 C11`
- **RT 诊断增强**：APOProcess 每次调用记录输入/输出峰值 + 缓冲标志（环形 8 条），
  UNLOCK 日志输出峰值高水位 `hot=(out,in,secs)` 与脏静音缓冲计数
  `silent_dirty=(calls,max_in)`——用于区分输入侧 vs DSP 侧爆音源，定位本次根因
- **热重载简化（回归 EAPO 语义）**：移除 v9.4 的 (mtime,size) 文件级预检/去重表与
  实验性进程级 `RELOAD_LOCK` 串行锁/try 变体；恢复“每实例独立 watcher + 10ms 去重 +
  spec 指纹短路 + transition/pending 协调”（EAPO `notificationThread` 同构）；保留
  diag 写锁（防多线程日志花屏）。消除对象地址复用撞旧去重条目、Unlock join 被重载
  队列拖住两类隐患——**对应章节**：`object 7.1.9/7.1.18`、`Equalizer 行为文档 C27/C32`
- **测试**：全量 442 passed（4 个既有管理员权限用例除外），新增
  `silent_buffer_with_garbage_outputs_silence` 回归用例（SILENT 标志 + 残留大数值 →
  输出必须静音）
- **模块引用规范（无详细模块版）.md**：版本号 v9.14 → v9.15。

> 对应 commit：driver `0801d19` / cli `无变更` / docs `7d9ee30`

## v9.14 — 2026-08-11

变更类型：`结构重构 + 文档同步`（删除 v9.11 遗留死代码 + 模块引用规范全面审阅修正）

- **死代码清理**：删除 `FilterFactory` / `FilterCreateResult` / `ConfigLoader`
  （filter.rs）与四个效果器模块的旧 `XxxFactory` 实现（v9.11 静态分派后
  遗留未删）；dsp.rs 测试支撑同步精简——**对应章节**：`pipeline 4.9/4.10/4.22`
- **文档全面审阅修正**：config / CLI / App / intent / apo 线程安全模型 / sys
  模块规范与主文档中的 `config.txt` / EAPO 命令残留全部更新为 v9.11 TOML 模型
  现状；`config 6.1` 旧逐行分发细节与 `6.3–6.16` 旧命令章节加废弃横幅；
  pipeline 模块树 / 4.9 / 4.19 / 4.20 同步；Equalizer 汇总表加 PEQ 现状行；
  `config convert`、Lock 降级 passthrough、复用链、延迟不上报等现状落档——
  **对应章节**：各模块规范
- **测试**：全量 441 passed（4 个既有管理员权限用例除外），release 零警告构建。
- **模块引用规范（无详细模块版）.md**：版本号 v9.13 → v9.14。

> 对应 commit：driver `778bcd4` / cli `无变更` / docs `7630073`

## v9.13 — 2026-08-11

变更类型：`缺陷修复 + 模块规范更新`（PEQ 衔接/瞬态/端点重协商系列修复）

- **跨频点段归属判据（宽 Q）**：`Fc>200` 且分频点处 |dB|>0.25 的段进 IIR——
  宽 Q 段的低频泄漏由 biquad 解析精确实现，避免 FIR 低频分辨率不足；
  T/L 频响统一计算路径（同段精确归零）；`cep_min` 奈奎斯特 bin 补齐——
  **对应章节**：`pipeline 4.22`、`PEQ 设计文档 3.1/3.3`
- **latency 修正**：直接 FIR 延迟改为最小相位群延迟保守估计 `fir_len/4`
  （1024 → 256），不再用 N-1 虚高值——**对应章节**：`PEQ 设计文档 5`
- **配置容错**：`config.toml` 缺失 = passthrough；Lock 解析失败降级 passthrough
  （配置坏/保存中间态时不再无声）；热重载失败保留旧链并记录具体错误——
  **对应章节**：`config 6.0`
- **端点重协商无缝化**：同 config/格式的 Relock 复用现有链（保留滤波器状态），
  Unlock 不再清指纹，热重载同步复用键；LOCK 日志带 `reuse=` 标记——
  消除重建瞬态
- **静音恢复淡入（“关网页哔声+断流”修复）**：输入确认静音时输出强制 0
  （抑制 IIR 振铃“嗡”），恢复时 8 ms 线性淡入（抑制阶跃）；静音判定带
  64 帧保持（避免低频正弦过零误判）——真机验证：关闭网页不再哔、音乐不断
- **测试**：peq_hybrid 9 用例（新增跨频点衔接、静音恢复淡入无阶跃）；
  全量 445 passed（4 个既有管理员权限用例除外），release 零警告构建。
- **模块引用规范（无详细模块版）.md**：版本号 v9.12 → v9.13。

> 对应 commit：driver `5009fb2` / cli `无变更` / docs `2344f97`

## v9.12 — 2026-08-11

变更类型：`缺陷修复 + 模块规范更新`（分块 FFT 块边界修复 + 延迟补偿回退 + IIR 叠加预期确认）

- **分块 FFT 块边界修复**：移除 per-call flush（打碎块边界导致切设备/切歌
  断续、无法播放），改为**块跨调用累积 + 输出驱动 + Reset 清空**（语义对齐
  EAPO libHybridConv）；新增变帧数（128/64/200/108 混合）跨调用一致性测试——
  **对应章节**：`pipeline 4.22`、`fir.rs`
- **延迟补偿回退**：v9.11 激活 `latency_frames_atomic` 引擎帧数补偿后，实测
  热重载/切歌播放卡住（帧协商错位；2026-08-10 实证复现），v9.12 回退为
  恒 0 + `GetLatency` 0（隐藏延迟不上报，音频整体滞后不可闻）——
  **对应章节**：`Equalizer 行为文档 2.10 C56`、`PEQ 设计文档 5`
- **IIR 低频叠加确认为设计预期**：低频段（fc<200）IIR 级联在邻近频点的
  叠加属正常 biquad 行为，不做 FIR 低频反向补偿（调音避让）；FIR 抽头规则
  保持 `next_pow2(sr×0.0213)` 夹 [1024, 8192]。
- **测试**：全量 442 passed（4 个既有管理员权限用例除外），release 零警告
  构建成功；真机验证：热重载、切歌、切设备均正常。
- **模块引用规范（无详细模块版）.md**：版本号 v9.11 → v9.12。

> 对应 commit：driver `87f6e87` / cli `无变更` / docs `167f638`

## v9.11 — 2026-08-11

变更类型：`结构重构 + 模块规范更新`（TOML 模型驱动 + 混合式 PEQ + 精简 DSP）

- **config TOML 模型驱动（P1-7 完成）**：`config.txt` 逐行命令体系整体移除
  （`config/commands/*` 删除），改为 `config.toml` + serde 双模型
  （FileModel → ChainModel，转换/校验在 config 层）；`[[effects]]` 保序链 +
  `[[effects.bands]]` 浮动段；`name`/`group`/`meta` 为 APP 元数据
  （driver 忽略、不参与指纹）；spec 指纹 = `EffectConfig::spec()`；
  watcher 监控 config.toml；`vxapo-cli config convert` 提供旧 txt 一次性
  转换——**对应章节**：`config 6.0`、`config TOML 设计文档.md`
- **混合式 PEQ（P1-6 完成）**：`pipeline/dsp/peq_hybrid.rs`——200 Hz 分频
  （Fc<200 段 IIR biquad 级联，Fc≥200 段最小相位 FIR，200.0 归 FIR）；
  FIR 目标 = 总目标 − IIR 频响（级联精确拟合）；抽头
  `next_pow2(sr×0.0213)` 夹 [1024, 8192]，≤2048 直接 FIR（AVX2）/ >2048
  分块 FFT（输出驱动 + 补块 flush，修复切换“嗡声”）——
  **对应章节**：`pipeline 4.22`、`PEQ 设计文档.md`
- **DSP 精简与工厂静态分派**：移除 graphic_eq / peq / copy / delay / vst /
  hp_lp / convolution；`biquad.rs` 保留为 loudness 内部工具（不再作为
  config 类型）；`factory.rs` 改为 `match EffectType` 穷尽分派，删除
  FilterRegistry / FilterCreateResult / OutcomeKind / index；保留集 =
  peq / preamp / aural / reverb / maximizer / wide / loudness——
  **对应章节**：`pipeline 4.10`
- **延迟补偿激活**：`latency_frames_atomic = chain.total_latency()`
  （Lock 时写入），`CalcInputFrames/CalcOutputFrames` 生效；
  `MAX_APO_LATENCY_SAMPLES` 提升到 8192；`GetLatency` 仍返回 0——
  **对应章节**：`Equalizer 行为文档 2.10 C56`
- **通道语义**：per-effect `channels` → `ChannelScopedFilter` +
  `Filter::fixed_channel_indices`；Chain 初始化优先使用固定槽位。
- **测试**：TOML 解析/校验/指纹、PEQ 频响拟合（跨 200 Hz 多采样率 ±0.5 dB）、
  分块 FFT 与朴素卷积逐样本一致 + 尾部 flush；全量 441 passed
  （4 个既有管理员权限用例除外），release 零警告构建。
- **模块引用规范（无详细模块版）.md**：版本号 v9.10 → v9.11；树与依赖表更新。

> 对应 commit：driver `a5f89e8` / cli `b2d72d1` / docs `f7ce544`

## v9.10 — 2026-08-10

变更类型：`结构重构 + 模块规范更新`（Wide 分频改为线性相位 FIR）

- **Wide FIR 分频重构（v9.10）**：200 Hz 分频由 IIR（LR4/Butterworth）改为
  1024 点线性相位 FIR（Hamming 窗理想低通 + 互补高通，
  `hp = 延迟 center 帧原信号 − lp`），两路逐样本完美重建、无 IIR 相位旋转/
  群延迟差——解决低频散；延迟 511 采样并按实际值上报 `latency()`——
  **对应章节**：`pipeline 4.22`、`config 6.16`
- **Wide 参数面迭代汇总（v9.9→v9.10）**：低频支路（<200 Hz）完全不处理；
  幂指数映射 `i' = Intensity^0.6`，`gHigh = 1+2.3·i'`（甜点 0.5 ≈2.52×、
  满档 3.3×）、`gComp = 1-0.10·i'`（缓解中频能量不足）；高频支路 tanh
  软限幅 `headroom_db = 0.2+0.8·(1-Intensity)`——**对应章节**：`pipeline 4.22`、`config 6.16`
- **复用 SIMD FIR 基础设施**：`convolution.rs` 的 `dot`/`init_fir_simd`
  提升为 `pub(crate)`，Wide 与 GraphicEQ 共用 AVX2/FMA 点积——**对应章节**：
  `pipeline 4.18/4.22`
- **测试**：Wide 18 用例（FIR 完美重建、脉冲延迟 = center、latency = 511、
  低频直通、高频宽度单调、极端反相有界等）；全量 `cargo test --lib`
  575 passed（4 个既有管理员权限用例除外），release 构建成功。
- **模块引用规范（无详细模块版）.md**：版本号 v9.9 → v9.10。

> 对应 commit：driver `31efd6d` / cli `无变更` / docs `af34b5f`

## v9.9 — 2026-08-10

变更类型：`结构重构 + 模块规范更新`（Wide/Aural 独立实现 + fxsound 目录平铺 dsp/）

- **Wide 算法替换（v9.9）**：移除 FxSound `Wide32.c` 移植与 AGPL 版权头，改为
  双频段加宽独立实现——4 阶 Linkwitz-Riley 分频（500 Hz，低频宽度为高频 25%）+
  每声道独立 velvet 去相关（25 ms、对数间隔、能量归一）+ `cosβ·dry + sinβ·decorr`
  混合（参考 DAFx24 StereoWidener）；Intensity=0 位精确直通，单声道直通
  （不再按 C 语义减半）——**对应章节**：`pipeline 4.22`、`config 6.16`
- **Aural 算法替换（v9.9）**：移除 FxSound `Auralp.c` 移植与 AGPL 版权头，改为
  「HP + 峰值电平跟随（瞬时 attack / 120 ms release，声道共享）+ tanh 软饱和
  奇次 + 半波整流 DC 阻塞偶次」的电平独立激励（参考 Jatin Chowdhury / FAUST 类
  设计）；谐波占比不再随输入电平变化，大 Drive 不发刺——**对应章节**：
  `pipeline 4.22`、`config 6.16`
- **目录结构重构**：`pipeline/dsp/fxsound/`（含唯一 `mod.rs`）移除，四个效果器
  平铺到 `pipeline/dsp/`，由 `pipeline/dsp.rs` 直接 `pub mod` 声明，与项目
  「目录平铺 + 上级文件聚合」命名风格对齐；测试支撑 `test_ctx/test_loader` 移至
  `pipeline/dsp.rs` `test_support`；parser 测试名 `fxsound_*` → `effect_*`——
  **对应章节**：`pipeline 4.10/4.22`、`模块引用规范（无详细模块版）.md`
- **测试**：Aural 谐波频域验证（偶次 2nd / 奇次 3rd）、电平独立性、Wide 去相关
  对称性/低频紧实/确定性等新用例；全量 `cargo test --lib` 568 passed（4 个既有
  管理员权限用例除外），release 构建成功。
- **模块引用规范（无详细模块版）.md**：版本号 v9.8 → v9.9；模块树效果器平铺更新。

> 对应 commit：driver `016ed9c` / cli `无变更` / docs `cd70e4f`

## v9.8 — 2026-08-10

变更类型：`结构重构 + 模块规范更新`（Maximizer 替换为独立 lookahead 峰值限幅器）

- **Maximizer 算法替换（v9.8）**：移除 FxSound `Maxi16.c` 移植与 AGPL 版权头，
  改为独立实现的「自动增益 + lookahead 峰值限幅」——全声道单极点 RMS 电平估计
  （τ≈250 ms）+ `Target` 增益回退；环形延迟（`Lookahead` 兼作 attack 时长）+
  多峰事件队列调度（参考 FFmpeg alimiter 思想）+ 线性 release + 输出硬钳位；
  抖动改为独立 xorshift64* PRNG（Uniform/Triangular/Shaped，16-bit 量化）；
  命令名与全部参数（GainBoost/MaxOutput/Release/Target/Lookahead/Dither/Wet/Dry）
  保持不变，config 兼容、spec 指纹不变——**对应章节**：`pipeline 4.22`、`config 6.16`
- **测试**：新增延迟线对齐、超限脉冲钳位、release 恢复、静音、自动增益满增益等
  用例，Maximizer 模块 18 passed；全量 `cargo test --lib` 561 passed（4 个既有
  管理员权限用例除外），release 构建成功。
- **模块引用规范（无详细模块版）.md**：版本号 v9.7 → v9.8；模块树 fxsound 注释更新。

> 对应 commit：driver `30a1a2e` / cli `无变更` / docs `f6f6fa1`

## v9.7 — 2026-08-10

变更类型：`性能优化 + 模块规范更新`（GraphicEQ 1024 点 FIR + AVX2/FMA 向量化，移除启动静音）

- **GraphicEQ 1024 点直接 FIR + SIMD**：直接 FIR 改为“旧→新连续段切片 + 逆序系数点积”，
  AVX2+FMA 8 路向量化（运行时探测，回退标量 mul_add）——1024 点 CPU 低于旧 512 点
  标量实现，200Hz 以下低频分辨率回到 ≈46.9Hz bin @48k（用户要求低频升 1024）——
  对应 `pipeline 4.18`、`Equalizer 行为文档 2.10 C51/C53`。
- **移除启动静音**：用户实测切回开头轻微断续为 Windows 自带行为，无需掩盖；
  保留字段与机制但置 0 直通——对应 `object 7.1.9`。
- **测试**：新增分段点积与朴素卷积逐点比对、AVX2/标量共用断言；
  全量 `cargo test --lib` 556 passed（4 个既有管理员权限用例除外），release 构建成功。
- **模块引用规范（无详细模块版）.md**：版本号 v9.6 → v9.7。

> 对应 commit：driver `7216d2d` / cli `无变更` / docs `277f017`

## v9.6 — 2026-08-10

变更类型：`缺陷修复 + 性能优化 + 模块规范更新`（GraphicEQ 直接 FIR 修复切换嗡声、启动静音、实例/IR/FFT 缓存）

- **GraphicEQ 改回 512 点直接 FIR（v9.6 实锤）**：v9.5 的分块 FFT 在流停止时会把最后
  ≤128 采样压在块缓冲里被引擎硬停丢弃 = 硬切尾音 → “M16+ 正常播放中切走时嗡一声”
  （音量随播放大小变化；空链无此问题，对照实验定位）。直接 FIR 无块缓冲、不丢尾音，
  切换嗡声消失——对应 `pipeline 4.18`、`Equalizer 行为文档 2.10 C51/C53`。
- **启动静音 100ms**（用户定稿）：新流建立后先保持 100ms 静音再正常出声，掩盖
  切回 M16+ 开头引擎加载的轻微断续（无淡入）；PreMix/PostMix 均应用——对应
  `object 7.1.9`。
- **批量实例化降本**：GraphicEQ FIR 按（频段,采样率）进程内缓存、rustfft FFT 计划
  缓存、`DisableProtectedAudioDG` 检查只做一次、MSFX 自愈扫描改直接构造路径
  （首次切换端点从数百次注册表打开降到几次）——设置页打开/设备切换时不再卡顿，
  audiodg CPU 峰值显著下降——对应 `pipeline 4.18`、`install 5.5.2`。
- **热重载补载修复**：文件级预检改为“只比较不记录”，过渡期间改配置不再丢补载；
  `FindNextChangeNotificationW` 失败自动重建监控句柄——对应 `object 7.1.18`。
- **诊断**：`C:\ProgramData\VxAPO\diag.log` 记录 LOCK/INIT/RELOAD/UNLOCK 事件、
  实例创建/销毁计数、停止前最后几次调用（帧数/标志/输出峰值），供真机问题定位。
- **测试**：全量 `cargo test --lib` 553+ passed（4 个既有管理员权限用例除外；
  全局计数用例在并行下有偶发 flaky），release 构建成功。
- **模块引用规范（无详细模块版）.md**：版本号 v9.5 → v9.6。

> 对应 commit：driver `b2c7636` / cli `无变更` / docs `ffff490`

## v9.5 — 2026-08-10

变更类型：`缺陷修复 + 性能优化 + 模块规范更新`（热重载补载修复、GraphicEQ 分块 FFT、空命令语义）

- **热重载补载修复**：文件级预检改为“只比较、不记录”，记录只在真正处理后写入——
  修复过渡（10ms）期间改配置时补重载被预检吞掉的问题；`FindNextChangeNotificationW`
  重置失败改为**重建监控句柄**（重建失败才退出），不再一次性杀死热重载——对应
  `object 7.1.9/7.1.18`、`config 6.2`。
- **GraphicEQ 改为分块 FFT 卷积**（块 128，v9.5）：频响与 1024 点直接 FIR 完全一致，
  单实例 CPU 约降 3~4 倍——多路音频流各自跑 PreMix 实例时不再吃满 audiodg
  （实测 ~57% → 预期 <20%），声音设置页卡顿随之缓解；隐藏延迟 128 采样不上报
  （与既有策略一致）——对应 `pipeline 4.18`、`Equalizer 行为文档 2.10 C51/C53`。
- **`GraphicEQ:` 空参数 = 显式移除 EQ**：不再报语法错误；不产生滤波器（passthrough），
  且产出 spec 指纹（与“无此命令”不同）——热重载必然触发，空命令不再是“改不动”——
  对应 `config 6.10`、`pipeline 4.18`。
- **运行诊断日志**：`C:\ProgramData\VxAPO\diag.log`（控制线程）记录 LOCK/RELOAD
  事件（实例 CLSID、滤波器数、spec 数、应用/跳过/失败原因），供真机问题定位。
- **测试**：空 GraphicEQ 解析/指纹用例 + 既有全量（551 passed，4 个既有管理员
  权限用例除外），release 构建成功。
- **模块引用规范（无详细模块版）.md**：版本号 v9.4 → v9.5。

> 对应 commit：driver `f7dc314` / cli `无变更` / docs `dd586c1`

## v9.4 — 2026-08-10

变更类型：`缺陷修复 + 模块规范更新`（双实例重复处理、默认效果运行期自愈、watcher 防自旋）

- **PostMix 实例默认直通**：Windows 对渲染设备同时挂 SFX(PreMix)+EFX(PostMix) 两个
  VxAPO 实例，此前都加载同一 config → GraphicEQ 双重卷积（音量异常偏低 + 双倍
  隐藏延迟/CPU，设备切换后帧协商更易错位）。v9.4 起 PostMix 不再解析用户 config
  （空链直通），保留 child APO 委托（前任 EFX APO 仍生效），且不启动 watcher——
  对应 `object 7.1.9`。
- **运行期默认效果自愈**：`Initialize` 时对已装 VxAPO 的端点调用
  `ensure_takeover_for_endpoint`——Windows 重新枚举/重启后从驱动模板灌回的微软
  CAPX 由 DLL 自动再接管（只动微软 CLSID、不覆盖第三方、幂等、失败降级），
  同时删除 DisableEnhancements / Disable_SysFx 强制启用增强链——对应
  `object 7.1.8`、`install 5.5.2`、`Equalizer 行为文档 2.10 C54`。
- **watcher 防自旋**：`FindNextChangeNotificationW` 重置失败 → 关闭句柄退出循环
  （不再无限重载）；`hot_reload` 增加 config.txt (mtime,size) 文件级预检，目录级
  事件由无关文件触发时直接跳过——修复 audiodg CPU 持续高位、声音设置页卡顿——
  对应 `object 7.1.9/7.1.18`。
- **测试**：`msfx_heal_action` 纯决策单测（微软 CAPX 替换/删除、第三方不动）；
  全量 `cargo test --lib` 551 passed（4 个既有管理员权限用例除外），release 构建成功。
- **模块引用规范（无详细模块版）.md**：版本号 v9.3 → v9.4。

> 对应 commit：driver `a106e10` / cli `无变更` / docs `7f5249b`

## v9.3 — 2026-08-10

变更类型：`算法替换 + 模块规范更新`（Reverb 由 Lexicon 移植替换为 Dattorro 板式混响）

- **`pipeline/dsp/fxsound/reverb.rs` 重写**：移除 FxSound `Lex16.c` 移植（AGPL 版权头一并移除），
  改为按 Jon Dattorro 1997 论文独立实现的板式混响——输入 4 级 AllPass 扩散
  （142/107/379/277）、双槽交叉反馈环路（672/908 正交 LFO 调制 APF + 4453/4217 主延迟 +
  1800/2656 扩散 APF + 3720/3163 尾延迟）、论文 Table 2 的 14 抽头输出（每抽头 0.6）；
  正确性经论文原文与 ValleyRackFree / johnhw 参考实现交叉核对，本文件为原创代码——
  对应 `pipeline 4.22`。
- **命令与参数完全兼容**：`Reverb:` 语法与取值范围不变；语义重映射——RoomSize → 槽内延迟缩放、
  Decay → 环路反馈（内部 0.25..0.95）、Damping/Bandwidth → 槽内/输入低通截止、
  Density → 扩散系数（1.0 = 论文默认 0.75/0.625/0.7/0.5）、Lat5/Lat6 → 早反射/尾音电平、
  MotionRate/MotionDepth → 调制 LFO 频率/深度（2 ms = 论文 EXCURSION 16 采样@29761 Hz）——
  对应 `pipeline 4.10/4.22`、`config 6.16`。
- **测试**：解析/静音/dry 精确直通/有限性/44.1k·48k·96k 最大参数/单声道尾音/工厂注册；
  全量 `cargo test --lib` 550 passed（4 个既有管理员权限用例除外），release 构建成功。
- **roadmap**：P1-1 更新为「四个效果已接入，v9.3 起逐个替换为更优算法」；新增 P1-3 算法升级项。
- **模块引用规范（无详细模块版）.md**：版本号 v9.2 → v9.3；模块树 fxsound 注释更新。

> 对应 commit：driver `cef16ef` / cli `无变更` / docs `7b2222c`

## v9.2 — 2026-08-10

变更类型：`新增能力 + 模块规范更新`（FxSound Wide 立体声加宽接入）

- **新增 `pipeline/dsp/fxsound/wide.rs`**：移植 FxSound `Wide32.c`
  （Theremino V2.0.3 简化环绕版，AGPL-3.0-or-later，保留版权头）——
  M/S 分解 + 侧信号放大（`1+3·Intensity`）+ 中央补偿（`1-0.3·Intensity`），
  无滤波/延迟；单声道按 C 语义输出减半。
- **`Wide:` 命令接入 config**：`Wide: Intensity 0.354331`（Intensity [0, 1]，
  0 时严格直通）；工厂注册追加，`FACTORY_COUNT` 18 → 19，`index::WIDE = 18`；
  parser 默认分支无需改动（`try_create_named` 精确分派）——
  对应 `pipeline 4.10/4.22`、`config 6.16`。
- **测试**：解析/直通/纯侧放大/纯中央补偿/单声道语义/有限性/工厂与 parser 集成。
- **roadmap P1-1**：Wide 落地后四个 FxSound 效果全部实现（听感验证留手动）。
- **模块引用规范（无详细模块版）.md**：版本号 v9.1 → v9.2；模块树新增 `wide.rs`。

> 对应 commit：driver `0eb5075` / cli `无变更` / docs `c7fef40`

## v9.1 — 2026-08-10

变更类型：`新增能力 + 模块规范更新`（FxSound DSP 效果器拆分：Aural Enhancer / Reverb / Maximizer）

- **新增 `pipeline/dsp/fxsound/` 子模块**：移植 FxSound `Auralp.c` / `Lex16.c` / `Maxi16.c`
  （AGPL-3.0-or-later，保留版权头与来源注释）——
  - `aural.rs`：二阶 Butterworth 高通（`filtDesign2ndButHighPass`）+ sin 奇偶谐波 + Wet/Dry；
  - `reverb.rs`：预延迟 + 四级 Lattice AllPass + 调制延迟网络（延迟线长度与 C 端
    `MasterLen` 一致，按实际 RoomSize + 最大 2 ms 调制预留）；
  - `maximizer.rs`：0.1 Hz 单极点电平估计 + lookahead 峰值限幅 + LCG 抖动 +
    16-bit 量化（`Dither None` 不量化）。
- **三个 EAPO 风格命令接入 config**：`AuralEnhancer:` / `Reverb:` / `Maximizer:`，
  参数解析后 clamp 到原始 c_* 区间；默认值取原 Quick preset 1 精神（Wet/Dry 可覆盖）。
- **工厂注册**：`factory.rs` 新增 `AuralEnhancerFactory` / `ReverbFactory` /
  `MaximizerFactory`，`FACTORY_COUNT` 15 → 18，`index` 新增 15/16/17；
  parser 默认分支改为 `try_create_named` 精确分派——修复宽容工厂（Convolution）
  吞掉已知命令非法参数的问题（v7.12「命令无效」契约真正生效）——对应
  `pipeline 4.10`、`config 6.1/6.16`。
- **测试**：解析/初始化/处理/RT 安全覆盖（44.1k/48k/96k、最大参数、静音、
  峰值不超 MaxOutput、Dither 复现、单声道语义、dry 直通、spec 指纹变化）。
- **模块引用规范（无详细模块版）.md**：版本号 v9.0 → v9.1；模块树新增
  `pipeline/dsp/fxsound/`；依赖表同步。

> 对应 commit：driver `99aad5b` / cli `无变更（lockfile 未动）` / docs `a9e3328`

## v9.0 — 2026-08-10

变更类型：`行为修正 + 模块规范更新`（CAPX 设备默认效果接管、GraphicEQ 卷积化、多流稳定性）

- **install/device/sysfx.rs（新增）**：接管 Windows CAPX「设备默认效果」`MSFX\N` 模板——
  替换微软 StreamEffect 为 VxAPO PreMix、删除微软 ModeEffect；端点 FxProperties 残留 MFX 同步清理；
  原始值备份到 `SysFxBackups`，卸载恢复——对应 `install 模块规范.md 5.5.2 Step 8/卸载 Step 4`
- **install Step 7 对齐 EAPO**：删除 `{1da5d803-...},5`（PKEY_AudioEndpoint_Disable_SysFx）强制启用增强
- **pipeline/dsp/graphic_eq.rs 重写**：弃用多段 biquad 级联（负增益叠加导致 -11~-12 dB），
  改为 EqualizerAPO 对齐的对数频率插值 + 最小相位 FIR + 1024 点直接时域卷积——对应 `pipeline 模块规范.md 4.18`
- **pipeline/dsp/convolution.rs**：新增 `with_ir` / `with_ir_direct` 内存 IR 注入
- **延迟策略**：GraphicEQ 直接 FIR 无分区块延迟，GetLatency 无 child 返回 0（EAPO 对齐）；
  `Chain::initialize` 末尾重算 total_latency
- **多流稳定性**：`process_audio` / `process_chain_interleaved` 增加输入/输出/临时缓冲
  长度校验与逐元素旁通（memmove 语义，修复 in-place `copy_from_slice` 重叠 UB）；
  RT 路径锁污染后 `into_inner` 不再二次 panic
- **Equalizer 行为文档.md**：新增 2.10（GraphicEQ 卷积实现 + CAPX MSFX），修正未确认清单
- **模块引用规范（无详细模块版）.md**：版本号 v9.0；模块树/依赖表对齐实际 driver 树
  （含 `object/apo/` 拆分、`pipeline/dsp/math.rs`、`install/device/sysfx.rs`、`utils/guid.rs` 等）

> 对应 commit：driver `9d4e1b5` / cli `f2a4aca` / docs `53edd94`

## v8.15 — 2026-08-06

变更类型：`新增规范`（App 引用规范——Tauri 前端架构草案落地为规范文档）

- **新增 `App 引用规范.md`**：App 定位与三层分离、现状（vxapo-app 脚手架 + 1249 行单体草案）、
  目标模块结构（src/ 目录树逐文件职责）、状态管理与数据流、types.ts 类型规范、
  后端 Tauri 命令接口、分阶段路线——对应章节 `App 引用规范 一~十`
- **已定决策固化**：响度补偿不做可视化只留开关（`Loudness: on|off`，默认 on）；
  超范围主动限幅在 App 写回时做（[-120,+48]、滤波深切地板 -60、NaN/inf 拒绝）；
  调音组件开关/总开关后续——对应 `App 引用规范 六`
- **主规范版本**：v8.9 → v8.15（头部版本滞后补齐，与 changelog v8.10-v8.14 对齐）

> 对应 commit：a4916bf（本地回填，随下次自然变更一并提交）

## v8.14 — 2026-08-05

变更类型：`模块结构对齐`（apo.rs → apo.rs + apo/ 子模块，与 install/config/pipeline 同风格）

- **`object/apo.rs` 改为模块入口**：保留 `ApoObject`、四个 COM `_Impl` 接口实现与公开 API
- **新增 `object/apo/` 子模块**：`state.rs`（状态机）、`inner.rs`（ApoObjectInner）、`config.rs`（配置路径/热重载）、`negotiate.rs`（格式协商）
- **聚合外壳收口**：`object/aggregate.rs` → `object/apo/aggregate.rs`，`factory.rs` 引用同步更新
- **行为不变**：452 passed，外部引用路径仅 `factory.rs` 的 aggregate 路径更新

## v8.13 — 2026-08-05

变更类型：`COM 引用边界收口`（`windows::Win32::System::Com` 统一经 prelude 重导出）

- **prelude 收口 COM 类接口/入口**：`IClassFactory` / `IClassFactory_Impl` / `CoCreateInstance` / `CoInitializeEx` / `CoTaskMemAlloc` / `CoTaskMemFree` / `CLSCTX_*` / `COINIT_MULTITHREADED`
- **业务层不再直接引用 `System::Com`**：`apo.rs` / `child.rs` / `factory.rs` / `dll_exports.rs` / `operation.rs` 全部改走 prelude
- **`IUnknown` / `Interface` / `GUID` / `HRESULT` / `implement` 也从 prelude 引入**，object/install 层不再出现 `windows::core::GUID` 等全限定 COM 类型

## v8.12 — 2026-08-05

变更类型：`引用边界收口`（系统 crate 类型只经 `sys/com/*` 重导出，HRESULT 常量集中于 prelude）

- **`windows::Win32::Media::Audio::Apo` 不再被业务层直接引用**：接口 IID 统一走 `sys/com/apo_interfaces` 的 `IID_*` 常量；`APO_CONNECTION_*` / `APO_REG_PROPERTIES` / `WAVEFORMATEX` 走 `sys/com/apo_types` 重导出
- **APOERR 错误码定义移至 `sys/com/prelude.rs`**：`apo_types.rs` 仅 re-export，消除 HRESULT 常量分散定义
- **`SELFREG_E_CLASS` 收入 prelude**：`dll_exports.rs` 不再本地定义
- **业务层原始 `HRESULT(0x...)` 清零**：`factory.rs`/`apo.rs`/`child.rs` 全部改用 prelude 常量

## v8.11 — 2026-08-05

变更类型：`代码质量`（Rust 架构优化：安全/可读性/零成本抽象）

- **aggregate vtable 转发去重**：IAPO/RT/CFG stub 统一由 `forward_method!` 宏生成，消除重复的 vtable 索引/transmute 样板——对应 commit `b50ea9a`
- **硬编码值提取常量**：AudioEngine 注册键（Flags/MaxInstances/连接数/接口 GUID）、MMDevices PKEY 值名、槽位 PID 均命名常量——对应 commit `b50ea9a`
- **重复代码抽取**：`ChildApo::resolve_supported` 统一输入/输出格式协商；`make_reg_props` 统一 Pre/Post 注册属性——对应 commit `b50ea9a`
- **可靠性修复**：`INST_COUNT::decrement` 改用 CAS 防下溢（并行测试 reset 交错不再把全局计数写成 u32::MAX）；移除 DSP process 路径里的 RT 探针 I/O——对应 commit `b50ea9a`

## v8.10 — 2026-08-05

变更类型：`实现对齐 + 模块边界优化`（P0-7 父槽位加载/配置生效全链路修复 + utils/guid 下沉 + driver 安装卸载全流程收口）

- **P0-7 父槽位加载闭环（driver 38a943b / cli ef1a0f4）**：聚合引用计数按 EAPO NonDelegating 语义修正；格式协商识别 WAVEFORMATEXTENSIBLE/IEEE_FLOAT；端点 GUID 从 `pAPOEndpointProperties` 读取并兼容 VT_LPWSTR；热重载过渡期 pending 不丢修改；`Chain::initialize()` 预计算 DSP 系数/状态——对应章节：`object 7.1.8/7.1.18`、`pipeline 4.5`、`install 5.5.2`
- **driver 安装/卸载全流程收口**：`install_endpoint` 内置 DisableProtectedAudioDG、全局 APO 注册刷新、AudioSrv 重启；`uninstall_endpoint` 内置停服/重启——对应章节：`install 5.5.2`、`CLI 引用规范.md 5.1/5.3`
- **`utils/guid.rs` 新增**：`guid_from_bytes` / `is_zero_guid` / `parse_guid_string` 从 `install/device/slots.rs` 下沉，安装层不再自持 GUID 解析——对应章节：`utils 8.3`、`install 5.3`
- **模块边界表更新**：`install/selector/operation.rs` 允许依赖 `object/dll_exports::register_apo_with_path`；`Chain::initialize` 写入 pipeline/object 规范——对应章节：`install 引用约束总表`、`pipeline 4.5`、`object 7.1.8/7.1.18`

> 对应 commit：`driver 38a943b / cli ef1a0f4`

## v8.9 — 2026-08-04

变更类型：`实现对齐`（P0-7 端到端调试驱动——install/卸载/快照/EAPO 对齐 + config 路径系统级修正）

- **EAPO 三档安装模式自动探测（执行端落地，先于规范）**：`slots::detect_install_mode`（纯逻辑三档：Win<8.1→LfxGfx / 仅 LFX+GFX→LfxGfx / 蓝牙容器 ID→SfxMfx / 否则 SfxEfx）+ `info::detect_mode_for_device/guid`（CLI 缺省 --mode 入口、APP 调用）——**对应章节**：`install 5.3/5.4`、`Equalizer 行为文档.md C41-C44`
- **install 5.4 模式检测改「VxAPO CLSID 成对」判定**（EDIFIER 实证：旧实现按任意 GUID 占槽误判 SfxMfx，实际 SFX/EFX 被 EAPO 占、MFX 被系统占）——**对应章节**：`install 5.4`
- **install 5.3 槽位 API 补全**：`registry_pid`（实证 0/3/5/6/7，非连续）+ `read_slot_value` REG_SZ/Binary 双格式兼容 + 全零 GUID 归一 NoValue + `detect_install_mode`——**对应章节**：`install 5.3`
- **install 5.5.2 安装实现对齐（2026-08-04 实证）**：FxProperties 已存在用 `open_for_write` 最小写权限（KEY_SET_VALUE|KEY_QUERY_VALUE，避开 MMDevices ACL 0x80070005）；verify 前 `CoInitializeEx`（修 0x800401F0）；子 APO 配置写独立信息区（删 FxProperties\ChildApoKeys 死代码）；self-preserve 过滤（重装不把 VxAPO 自己当子 APO）；槽位写 REG_SZ（GUID 字符串，防 EAPO wrong type）；EAPO 互斥删槽位（SfxEfx 保 MFX、SfxMfx 保 EFX）；capture 只装 PreMix——**对应章节**：`install 5.5.2`
- **快照恢复语义（install/卸载）**：被覆盖槽位名+原值无条件备份（PreMixSlot/PostMixSlot + Value）；卸载只删 VxAPO CLSID → **删空才写回备份**（接管者不覆盖）→ 恢复后删信息区键——**对应章节**：`install 5.5.2`
- **config 路径改系统级 `C:\ProgramData\VxAPO\{GUID}\config.txt`（用户指示修正）**：audiodg 是 SYSTEM 服务，`documents_folder()` 拿到 SYSTEM 的 Documents 读不到 CLI（用户进程）写入的文件——改用全用户共享 ProgramData（与快照目录同根）——**对应章节**：`object 7.1.8/7.1.9`、`CLI 引用规范.md`
- **sys/registry 补 open_for_write**（KEY_SET_VALUE|KEY_QUERY_VALUE）+ `SAM_SET_VALUE` 常量 + create 文档警示 MMDevices ACL——**对应章节**：`install 5.5.2`
- **ref_count 测试 flaky 修复（附带）**：`decrement` 改 `saturating_sub` 防测试并行 reset_for_test 交错下溢 panic
- **feedback：P0-7-1 已修订**（detect_install_mode 实现先行 + 规范同步 + config 路径系统级）——**对应章节**：`feedback.md #P0-7-1`
- **roadmap：P0-7 落点补充**（detect_mode_for_device/guid + 快照恢复 + ProgramData 路径；依赖 P0-4/P0-5/P0-6 均 Done）——**对应章节**：`roadmap P0-7`

> 对应 commit：`d7e41da`

## v8.8 — 2026-08-04

变更类型：`文档同步`（P0-7 CLI 端到端验证——《CLI 引用规范.md》定稿）

- **新建《CLI 引用规范.md》（v8.8 定稿，源码实读驱动）**：CLI 现状（纯交互诊断，仅 winreg 依赖）/ driver 可复用 API（install_endpoint/uninstall_endpoint/enumerate_devices/slots 全接口源码实读确认）/ CLI 边界（三层分离，不触碰 pipeline-RT）/ 修改路线 Phase A-D（依赖接入→核心命令→快照→P1 扩展）——**对应章节**：`CLI 引用规范.md`
- **命令参数明确**：`<device>` 接受 GUID（推荐）/ 枚举序号，统一经 `resolve_device` 映射三元组；`<file>` = 源文件绝对/相对路径，CLI 原样写入 per-device config.txt——**对应章节**：`CLI 引用规范.md 4.4.1`
- **GUID 友好名称**：EAPO PreMix `{EACD2258-...}`/PostMix `{EC1CC9CE-...}`（源码实读）+ VxAPO PreMix/PostMix（v7.5 正式 GUID）+ premix/postmix 用法二次明确（混音前流 vs 最终混合输出）——**对应章节**：`CLI 引用规范.md 4.5`
- **快照 = 变更对比**：`snapshot_device` 捕获**注册表**（FxProperties 5 槽位 + childApoPath + DisableEnhancements，**不含 config**——config 比对归 driver 目录监控 + filter_spec 序列对齐非哈希）；`snapshot diff` 红绿列示 + 统计行；**基线保持**（卸载用最开始的基线对比，重装才替换）；`snapshot restore` 显式恢复——**对应章节**：`CLI 引用规范.md Phase C`
- **命令状态流**：U/B/I/L 四态（DeviceInfo + 快照存在性判定）+ 命令×状态矩阵（前置判断拒绝+可读错误）+ 状态切换逻辑 + **每步骤绝对严格错误检验**（错误码+消息+所属步骤，失败提示恢复路径）——**对应章节**：`CLI 引用规范.md 5.4`
- **行为流 5.1-5.5**：安装/验证/卸载具体到函数（层次：CLI 命令→CLI 辅助→driver API→driver 内部）+ 端到端验证索引 ①-⑥（含 P0-6 child 委托链联调）——**对应章节**：`CLI 引用规范.md 五`
- **roadmap：P0-7 → Spec-Finalized**（落点回填《CLI 引用规范.md》；依赖 P0-4/P0-5/P0-6 均 Done）——**对应章节**：`roadmap P0-7`

> 对应 commit：`2ba7c8c`

## v8.7 — 2026-08-04

变更类型：`实现对齐`（P0-5/P0-6 执行端实现完成 → 规范侧合规核对通过 → Done）

- **P0-5 RT 入口 panic 防护 → Done（v8.7 合规核对通过）**：执行端实现完成报告（e2fb954）DoD 全勾——
  431 passed + 2 panic 防护测试；RT 无违规（catch_unwind 不跨函数边界 + panic 兜底零分配）；仅 object/apo.rs ⊆ 影响模块；
  **三层防护语义落地**（编译期约束 → debug catch_unwind → release abort 兜底）——**对应章节**：`roadmap P0-5`、`object 7.1.11/7.1.12`
- **P0-6 子 APO 委托实现 → Done（v8.7 合规核对通过）**：执行端实现完成报告（e2fb954）实现全勾——
  object/child.rs（三接口类型化持有 + 全部委托 + v8.6 Option 参数）、object/apo.rs（child_apo 字段 + Initialize 反查 + APOProcess 前置 + GetLatency/Lock/Unlock 委托）、
  install/device/slots.rs（CHILD_APO_PATH_ROOT + child_apo_key_exists + read_child_apo_guid + ChildApoKind）；
  RT 无违规（child 前置独立锁短持 + 委托不分配）；431 passed；
  child 委托链完整测试需真实 COM + 已注册 APO 无法单测——**非实现缺口**，参照 P0-1/P0-4 先例判定 Done，端到端联调留 P0-7 CLI——
  **对应章节**：`roadmap P0-6`、`object 7.2/7.1.8/7.1.11`、`install 5.3/5.5.2`
- **P0-6 缺陷说明**：原 7 个 null 接口防御测试因类型化方案 Drop 对 null Release 解引用 vtable 崩溃（STATUS_STACK_BUFFER_OVERRUN）删除——**类型化安全边界**，注释已留（执行端 e2fb954）——**对应章节**：`roadmap P0-6`、`object 7.2`

> 对应 commit：`9d29947`

## v8.6 — 2026-08-04

变更类型：`实现对齐`（child.rs 格式协商委托参数签名采纳执行端建议）

- **object 7.2 `is_input/output_format_supported` 输入参数 `*mut IAudioMediaType` → `Option<&IAudioMediaType>`**（执行端建议采纳）：
  p_opposite 可 null（无对端）/ p_requested 由父转发非空——可空借用语义用安全引用表达，与 windows-rs
  `#[interface]` 对可空接口参数的 `Option<&>` 风格一致；输出 `pp_supported: *mut *mut` 为 COM 输出必须保留
  裸指针（方法仍 unsafe）；7.1.16 父接口为 COM vtable 约束不改——**对应章节**：`object 7.2`、`feedback.md #P0-6-4`

> 对应 commit：`90b4a3d`

## v8.5 — 2026-08-03

变更类型：`实现对齐`（槽位失守检测产品化 + 全量/非全量备份判定定稿 + driver 同步）

- **intent.md 七节「槽位失守检测」v8.5（产品意图，用户指示补充）**：App/CLI 启动/切换设备时检测**安装模式用到的 fx 槽位**（非全部 5 槽）非 VxAPO CLSID → 提示重装；重装前把被夺占槽位当前值**覆盖备份**为 childapo（最新前任）；备份语义区分（全量备份回退基线 vs 槽位覆盖备份 childapo）；全量判定唯一依据 = `HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}` 键存在性（不存在=全量，存在=非全量）——**对应章节**：`intent.md 七/十三`
- **全量判定收敛（用户指示）**：childapo 键**存在 = 非全量**、**不存在 = 全量**（初始/完全卸载后安装等价）；「第三方破坏安装信息区」不构成场景（VxAPO 私有路径，无第三方会写——EAPO 只写自己的 `HKLM\SOFTWARE\EqualizerAPO\Child APOs`）；**卸载必删 childapo 键**（用户确认）——**对应章节**：`intent.md 七`、`feedback.md #P0-6-3`
- **install 5.3 slots.rs v8.5**：补 `CHILD_APO_PATH_ROOT` + `child_apo_key_exists`（全量判定依据检测 API）——**对应章节**：`install 5.3`
- **install 5.5.2 v8.5**：安装流程补「0. 全量判定」+ Step 4 覆盖语义（失守时 childapo 覆盖为被夺占槽位值）；卸载流程补「删整个键」——**对应章节**：`install 5.5.2`
- **object 7.2 v8.5**：槽位失守检测精确定性（安装模式槽位 + 覆盖备份 + 全量/非全量判定 + 卸载删键）——**对应章节**：`object 7.2`

> 对应 commit：`b743bb1`

## v8.4 — 2026-08-03

变更类型：`实现对齐`（P0-6 子 APO GUID 来源三处矛盾消解 + 路径隔离修正）

- **P0-6 三处规范内部冲突消解（用户指示确认实现方向）**：
  ① object 7.1.8「从 APOInitSystemEffects 提取子 APO CLSID」不可行（该结构无子 APO 字段）；
  ② object 7.2「childApoPath」vs install 5.5.2「FxProperties childGuid」位置矛盾；
  ③ **路径隔离**（用户补充指示）：EAPO `APP_REGPATH = HKLM\SOFTWARE\EqualizerAPO`（RegistryHelper.h 33）——
  **VxAPO 不得复用该路径**（污染 EAPO 安装信息区）、改用独立 `HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}`
  ——**对应章节**：`feedback.md #P0-6-2`
- **object 7.1.8 修正**：子 APO GUID 来源 = 端点 GUID → `HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}\{PreMixChild|PostMixChild}`
  （APOInitSystemEffects 仅取端点 GUID）；引用来源新增 `install/device/slots` 依赖 + 引用约束总表同步——
  **对应章节**：`object 7.1.8`、`object 引用约束总表`
- **object 7.1.2/7.2 修正**：`child_apo_clsid` 注释 + 子 APO 来源（VxAPO 独立路径 + 禁止读写 EAPO 路径）——
  **对应章节**：`object 7.1.2/7.2`
- **install 5.3/5.5.2 修正**：slots.rs 补子 APO 安装信息区读取职责 + 路径隔离注；
  安装流程 Step 1/4 + 卸载流程 Step 3 更新（`PreMixChild`/`PostMixChild` 值名，EAPO DeviceAPOInfo.cpp 558-563 对齐；
  废弃含糊「childGuid」表述）——**对应章节**：`install 5.3/5.5.2`

> 对应 commit：`88b3384`

## v8.3 — 2026-08-03

变更类型：`实现对齐`（P0-5 panic 语义重评 + EAPO 源码逐行查验 + 治理流程废止 hash 回填循环）

- **P0-5 策略重评（三层防护语义澄清）**：原「杜绝 panic 跨 FFI unwind 到 audiodg 崩溃」表述失准——修正为「**杜绝 UB 传播**」；三层防护各司其职（编译期 O1 不 panic 为源头 → debug catch_unwind 验证防御路径 → release abort 确定性兜底，panic hook 为诊断工具非防线）——**对应章节**：`roadmap P0-5`、`object 7.1.11/7.1.12`、`主规范 十五`
- **EAPO 源码逐行查验（`D:\Source_Code\equalizerapo-code`）→ 新建《Equalizer 行为文档.md》**：EqualizerAPO.cpp / DeviceAPOInfo.cpp / FilterEngine.cpp/.h / FilterConfiguration.cpp / RegistryFunctions.cpp / IFilter.h 全量逐行核对——**发现 5 处规范偏差（S1-S5）** 并同步修订
- **S1：主规范 18.2 D2「协商期锁死 in==out」表述失准**——EAPO 实际是**拒绝下混**（in>out → S_FALSE + outFormat）+ **仅 mono→stereo 上混补做**（126-128）；VxAPO「等价立场」同步修正为「拒绝下混 + 仅 mono→stereo 上混」——**对应章节**：`主规范 18.2 D2`、`roadmap P0-6 ③`
- **S2：E3.4 testAPOInstallation 描述与源码不符**——实际激活 `IAudioClient`（GetDevice → Activate → GetMixFormat → Initialize 共享模式 100ms）做**音频管线自检**，非「CoCreateInstance 验证 DLL」；失败抛 DeviceException——**对应章节**：`install 5.5.2`（E3.4）
- **S3：child 委托失败销毁语义未覆盖**——EAPO `IsInputFormatSupported` 委托失败会 `resetChild()` 销毁 child 降级（258-283）；主规范 18.2 D5 修订注明——**对应章节**：`主规范 18.2 D5`
- **S4：GetLatency 无 child 返回 0**——EAPO `*pTime=0` 后仅 child 委托（82-95），VxAPO 误读「无 child 走自身」；object 7.1.14 伪代码对齐为「无 child 返回 0」——**对应章节**：`object 7.1.14`、`object 7.2`
- **S5：无冒号行语义澄清**——EAPO 静默跳过（FilterEngine.cpp 329-330）vs VxAPO 拒绝报错 = **有意差异**（VxAPO 更严格，intent「语法严格性」）——**对应章节**：`config 6.1`、`intent.md`
- **治理流程废止 hash 回填循环（v8.3）**：changelog 每版本 `对应 commit` 改为**一次性回填**（版本提交完成时填写；此后禁止回填/追加提交；已发布记录冻结）——**对应章节**：`.clinerules/changelog-rule.md`

> 对应 commit：`854cdc4`

## v8.2 — 2026-08-03

变更类型：`实现对齐`（P0-5 RT 入口 panic 防护定稿 + 治理流程补强）

- **P0-5 RT 入口 panic 防护定稿**：`APOProcess` / `CalcInputFrames` / `CalcOutputFrames` RT 三入口 `catch_unwind` 包裹——
  debug（`panic="unwind"`）跨 FFI unwind 防御第一道（捕获 → 安全降级输出不向 audiodg 传播）；
  release（`panic="abort"` O3）空操作、真防线 telemetry/panic.rs hook；捕获行为（输出清零 + BUFFER_SILENT +
  stats.error_count++ / 保守帧数返回值）+ 测试要点——**对应章节**：`object 7.1.11/7.1.12`、`主规范 十五`、`telemetry 9.2`
- **治理流程补强（P0-5 教训）**：spec-finalize skill 定稿流程强制「**先落地规范正文（含子规范对应条目）→ 再递增版本号 → 写 changelog → 最后改 roadmap 状态**」——
  Step 4「先落地规范正文」强制步骤 + Step 9 子规范交叉核对自检；roadmap-rule 补「执行顺序（v8.2 补强）」节——
  子规范条目缺失 = 定稿不完整 = 执行端无法落地——**对应章节**：`.clinerules/skills/spec-finalize/SKILL.md`、`.clinerules/roadmap-rule.md`
- **roadmap：P0-5 → Spec-Finalized**（落点回填 + DoD 规范定稿 ☑）——**对应章节**：`roadmap P0-5`

> 对应 commit：`d7b7920`

## v8.1 — 2026-08-03

变更类型：`结构重构`（P0-6 子 APO 实现对齐——EAPO 源码精读 + 主规范新增「EAPO 对齐度与差异化」章节）

- **P0-6 开放决策收敛（EAPO 源码对齐）**：实读 EAPO `EqualizerAPO.cpp`/`FilterEngine`/`FilterConfiguration`——
  ① Unlock 失败语义（默认 VxAPO 容错）；② 双链过渡下 childRT 委托**前置每帧一次**（双链共享同一份 child 输出；child 不在任一链内）；③ 通道约束**方案 A**（child 输出==父链输入==最终输出；Lock 校验三方一致）——**对应章节**：`roadmap P0-6`
- **主规范新增「十八、EAPO 对齐度与差异化」**：对齐度 A1-A5（创建/QI/Initialize/降级/APOProcess 时序/GetLatency/就地缓冲）；
  差异化 D1-D5（父内双链 vs EAPO 单链配置级过渡 / realChannelCount 维度切换 vs VxAPO 显式约束 / child 输出通道数不可知的约定显式化 /
  Unlock 失败语义差异 / IsInputFormatSupported 委托失败回落）+ 差异化带来的设计影响（RT 成本叠加 3 次 DSP / 维度稳定前提 / 校验前置）——**对应章节**：`主规范 十八`

> *v8.1 补正说明：撤销「显式约束/显式优于 EAPO 隐式信任」表述（语义保留于 roadmap P0-6、主规范 18.2/18.3，已同步重写为「等价立场」）——经推演，EAPO 靠「协商期锁死 in==out + 协商委托 child + mono→stereo 补系统默认」构成完整通道语义；VxAPO 与之**等价**，唯一区别是不引入 realChannelCount 维度切换机制（协商后冗余/死路径），属实现简化非更严谨。*
>
> *P0-6 定稿补记（v8.1，EAPO 源码精读闭环 + 用户决策）*：
> - **子 APO 来源** = 安装时被 VxAPO 接管槽位的**前任 APO**（`PreMixChild/PostMixChild` 存 `childApoPath\{deviceGuid}`，对齐 EAPO DeviceAPOInfo；备份全部槽位供回退、子 APO 仅对应实际装入槽位）——**对应章节**：`object 7.2`
> - **槽位失守检测 = 应用层**（CLI/GUI 启动/切换设备时检测槽位非 VxAPO CLSID → 提示重装 → 重装前把**当前**槽位备份为新 childapo「最新前任」）；**watcher 不负责**（对齐 EAPO Configurator）——**对应章节**：`object 7.2`
> - **无需注册表监视**：EAPO `watchRegistryKey` 是 `readRegString/readRegDWORD` 配置命令副作用（RegistryFunctions.cpp 52/92）；VxAPO config 纯文件无 readReg → 不需要——**对应章节**：`object 7.2`
> - **运行期委托**：childRT->APOProcess **前置每帧一次**（双链共享其输出）；child 不在 current/outgoing 链内——**对应章节**：`object 7.1.11/7.2`
> - **Unlock 容错 + 重置防御**：child 解锁失败 → 父继续解锁（`UnlockForProcess(void)` 无重试语义已核证）+ child 标记需重置 → 下次 Lock 前 reset/重建——**对应章节**：`object 7.1.10`、`roadmap P0-6 开放决策①`
> - **P0-6 状态 → Spec-Finalized**（roadmap），DoD 规范定稿 ☑——**对应章节**：`roadmap P0-6`
>
> 对应 commit：`8e5c54f`（v8.1 主体）+ `b4763fb`（补正）+ `3d501c1`（P0-6 定稿补记）

## v8.0 — 2026-08-03

变更类型：`结构重构`（治理机制 major 升级——反馈外置 feedback.md + feedback-rule）

- **反馈外置（roadmap 精简）**：执行端反馈统一记录于根目录 `feedback.md`（11 条迁移：P0-2-1/2/3、P0-3-1/2、P0-4-0/1/2/3/4）；
  `roadmap.md` 删除全部反馈/修订记录正文，条目只留状态/DoD/规范落点 + 引用（`> 反馈记录：feedback.md #PX-X`）——
  **对应章节**：`roadmap.md`、`feedback.md`（新建）
- **feedback-rule（新建规则）**：`.clinerules/feedback-rule.md`——反馈触发条件 / 记录格式（PX-X + 影响版本 + 问题 + 规范侧判定 + 修订记录 + 状态）/ 生命周期 / 与 roadmap-rule、主规范十七、changelog-rule 衔接——**对应章节**：`.clinerules/feedback-rule.md`
- **roadmap-rule 同步**：补「反馈外置（v8.0）」节——反馈只留引用，状态机/DoD 不受影响——**对应章节**：`.clinerules/roadmap-rule.md`
- **主规范十七更新**：反馈闭环语义不变（执行端不改状态 + 零容忍绕过 + 规范侧修订），反馈落点由 roadmap → feedback.md——**对应章节**：`主规范 十七`
- **P0-4 执行端待办汇总保留**：对象层接线 + v7.11/v7.12 严格化（roadmap P0-4 条目）——DoD 未全勾，状态保持 Spec-Finalized

> 对应 commit：`106729f`

## v7.12 — 2026-08-03

变更类型：`缺陷修复`（P0-4 二次反馈——「未知命令」判定失效 → 命令关键字白名单三段式）

- **命令关键字白名单（三段式）**：`split_command_value`（冒号数量）→ **命令名 ∈ 白名单**（静态命令 / REW `Filter N:` 前缀 / `registry.factory_names()`）→ `try_create`——
  修复 v7.11「Unmatched → SyntaxError」失效：`BogusCommand: x` 有冒号被 Convolution 宽容解析（任意非空=IR 路径）接管 → 永远到不了 Unmatched → 「未知命令」永不触发——
  **对应章节**：`config 6.1`（白名单小节）
- **Unmatched 语义收窄**：`try_create` Unmatched 仅表示「已知命令的参数无效」（命令已过白名单）——文案 `命令无效 'X'：参数无法解析`（原「未知命令」由白名单分支判定）——**对应章节**：`config 6.1`
- **VSTPlugin 特判**：白名单命中但功能未启用 → 不过 try_create，直接 `SyntaxError「命令无效 'VSTPlugin'：该命令当前未启用（预留）」`（诚实且准确定位）——**对应章节**：`config 6.1`、`pipeline 4.20`

> 对应 commit：`b26a324`

## v7.11 — 2026-08-03

变更类型：`结构重构`（P0-4 潜在问题反馈② → config 语法严格化 + 错误报告方案定稿）

- **config 语法严格化（用户产品决策）**：每行必须 `命令关键字: 参数`——无冒号行拒绝（SyntaxError「缺少冒号」）、
  一行仅一个冒号（SyntaxError「多余冒号」）；**v7.9 裸命令可达性修正反转**（无冒号不再 `try_create(cmd)`）——
  `BogusCommand` 不再被 Convolution 宽容语义误接——**对应章节**：`config 6.1`、`intent.md「config.txt 语法严格性」`
- **Unmatched → SyntaxError**：`registry.try_create` 返回 Unmatched（未知命令）→ 不再 `log::warn` 跳过，
  改为 `SyntaxError「未知命令」`整体失败（保留旧链）；「配置写错必有反馈」——**对应章节**：`config 6.1`
- **Convolution 参数严格化**：`parse_convolution_params` 从 `Option` → `Result`——≥3 tokens / 第 2 个非数值
  → `ParseError`（`ir.wav -6 abc` 不再静默忽略 `abc`）——**对应章节**：`pipeline 4.19`
- **VST 静默 NoMatch 对齐**：`VSTPlugin:` v7.11 起与未知命令同样落 SyntaxError（诚实反馈，不再静默跳过）——
  **对应章节**：`pipeline 4.20`
- **诊断日志归属（intent 固化）**：config 解析错误摘要写入**软件安装根目录 `log/`**（非 Documents\VxAPO 设备目录），
  由应用层读取呈现；DLL 只写日志（纯落盘，非状态回传通道），watcher 天然不监控 log/——**对应章节**：
  `intent.md「诊断日志归属」`

> 对应 commit：`b71f1f8`

## v7.10 — 2026-08-03

变更类型：`实现对齐`（P0-4 实现完成报告反馈①——watcher 线程模型澄清 + 对象层接线缺口确认）

- **ConfigWatcher 外部驱动模型澄清（执行端反馈①）**：`config 6.2` 原「new 启动 watcher 线程」与 `wait_and_handle`
  外部驱动矛盾——统一为 **`ConfigWatcher` 不自启线程**（new 只建句柄；线程由调用方 `object/apo.rs::start_watcher`
  创建并循环驱动），`shutdown` 不 join（join 由 `stop_watcher` 负责：SetEvent → join → close）——
  **对应章节**：`config 6.2`、`object 7.1.9`（start_watcher 定义）、`object 7.1.10`（stop_watcher 定义）、`object 7.1.3`（字段补 shutdown_event/watcher_thread）
- **P0-4 对象层接线缺口确认（执行端遗留 1）**：`apo.rs` 尚未创建 watcher 线程（LockForProcess 末尾 start_watcher
  + UnlockForProcess stop_watcher）——**属执行端 P0-4 剩余项**，规范本文件已完备，执行端补做后回归；
  P0-4 **不标记 Done**（热重载链路未实际接通）——**对应章节**：`object 7.1.9/7.1.10`、`roadmap P0-4`

> 对应 commit：`f860c10`

## v7.9 — 2026-08-02

变更类型：`结构重构`（P0-4 配置变更检测方案定稿——filter_spec 指纹 + 目录级事件驱动语义落定）

- **目录级语义澄清（执行端反馈 → 方案定型）**：`FindFirstChangeNotificationW` 是**目录级通知**、不提供具体文件名——
  旧 6.2「校验文件名 == config.txt」无法实现；统一为 `DirectoryChanged(watch_dir)`，hot_reload 内 spec 指纹比对决定是否真正切换——
  **对应章节**：`config 6.2`、`object 7.1.8`
- **filter_spec 配置指纹（用户方案）**：parser 分发层统一产出 `produce_spec(cmd, value)`（命令名小写 + `\x1F` 分隔 + token 级规范化）；
  `FilterSpec = String`（单一规范化字符串，含无冒号裸命令整行处理）；`parse_file_with_spec` 双返回 `(滤波器列表, SpecChain)`——
  **对应章节**：`config 6.1`
- **配置变更检测行为链**：目录变更 → 128KB 逐文件闸门 → 重新解析 + spec 比对（与 active_spec）→ 相同幂等跳过 / 不同构建新链 + 双链过渡；
  **Include 失败 = 整体解析失败**（不更新 active_spec）；active_spec 构建成功即更新——
  **对应章节**：`object 7.1.18`、`object 7.1.3`、`object 7.1.9`
- **watcher 生命周期随锁定周期**：Initialize 不再启动（未锁定无可放新链）；`LockForProcess` 末尾启动、`UnlockForProcess` 停止（对齐 EAPO startMonitorThread）——
  **对应章节**：`object 7.1.8/7.1.9/7.1.10`
- **裸命令可达性修正**：无冒号行（`PK Fc 1000`）当前 value="" → try_create 收空串 → Unmatched 到不了工厂；值空且非配置关键字时 `try_create(cmd)`（整行作参数）——
  **对应章节**：`config 6.1`
- **产品意图文档**：新建 `intent.md`（产品定位 / 用户行为模型——正常路径 CLI/UI 管理、手动改文件为边缘降级 / 驱动-应用层边界 / 治理地位：规范上位）——
  **对应章节**：`intent.md`（根目录新文档）

> 对应 commit：`f042966`

## v7.8 — 2026-08-02

变更类型：`外部借鉴`（EqualizerAPO 源码二次检查 → P0-4 热重载事件驱动 + 容错对齐）

- **ConfigWatcher 事件驱动**：`FindFirstChangeNotificationW` + `WaitForMultipleObjects` 替代 2000ms 轮询（延迟 10ms 级、无空闲 CPU）；**监控目录**（非文件——文件删除重建时句柄失效）；去重窗口 500ms→10ms（对齐 EAPO + Note 72）；shutdown_event 退出 + join——**对应章节**：`config 6.2`、`object 7.1.8`
- **热重载失败保留旧链**：解析失败不再变空链直出（EQ 消失），保留 current_chain + log::warn（对齐 EAPO loadConfig 失败不替换）——**对应章节**：`object 7.1.18`
- **过渡缓冲预分配**：LockForProcess 按 `max_frame_count × max_ch` 预分配 temp_buffer_old/new，杜绝 RT 首次过渡 `resize()` 扩容——**对应章节**：`object 7.1.9`
- **EAPO 对比澄清**：R1（退役链零析构）与 EAPO `previousConfig` 为**等价对齐**（EAPO 也在控制线程 loadConfig 析构，非我此前误述的"我们更优"）——**对应章节**：`object 7.1.3`（R1 注）

> 对应 commit：`2c304e4`

## v7.7 — 2026-08-02

变更类型：`缺陷修复`（实现缺陷反馈：IsInputFormatSupported 时序错误）

- **补关键时序约束（object 7.1.16）**：`IsInputFormatSupported`/`IsOutputFormatSupported` 在 `LockForProcess` 之前被引擎调用，此时 `pipeline_context` 为全零 `PipelineContext::new()`——**禁止依赖 pipeline_context 做等值比较**（真实格式 vs 全零永远不等 → 拒绝所有格式、APO 无法协商）。正确做法是对请求格式做**独立属性检查**（浮点格式 + 44.1k~192k + 1~8 通道），属性在协商时已确定、与锁定后上下文无关——**对应章节**：`object 7.1.16`

> 对应 commit：`5da8b60`

## v7.6 — 2026-08-02

变更类型：`实现对齐`（P0-3 实现反馈闭环 → APOInitSystemEffects 提取路径实测化）

- **APOInitSystemEffects 提取路径修订**：`pSystemEffectsProperties->pEndpointGuid` → `pAPOSystemEffectsProperties`（`IPropertyStore`）取 `PKEY_AudioEndpoint_GUID`（PROPVARIANT VT_CLSID 的 `puuid`）——windows-rs 0.62.2 实测结构（P0-3 实现反馈①）——**对应章节**：`object 7.1.8`、`sys 3.3.1b`
- **无 GUID 兜底描述同步**：`PKEY_AudioEndpoint_GUID` 提取失败/为空时回退 `_default`（原 `pEndpointGuid` 残留修正）——**对应章节**：`object 7.1.8`
- **apo.rs 二次检查对齐（规范侧实读 881 行）**：① 7.1.11 过渡完成不再先置 `reloading=true` 再调 hot_reload（会短锁拦截自身），改为不置位直接调用 + pending 残留 bypass 防御 + advance None→factor=1.0；② 7.1.8 Initialize 非法数据降级默认配置仍返回 Ok（非 E_INVALIDARG，SDK 容错）；③ 7.1.8 补 `documents_folder()` 失败→固定 `C:\ProgramData\VxAPO\config.txt` 二级兜底——**对应章节**：`object 7.1.8/7.1.11`
- **P0-3 per-device 配置路径进入 Done**：实现完成（441 passed，含 known_folder/config_path 测试）+ 合规核对通过；watcher 接线留待 P0-4（v7.3 约定，属 P0-4 职责）

> 对应 commit：`ed8557b`

## v7.5 — 2026-08-02

变更类型：`实现对齐`（正式 CLSID GUID 落定）

- **正式 GUID 替换占位值**：`CLSID_VXAPO_PRE_MIX = 41C34613-D391-459D-A039-72B2B15A1A1D`、
  `CLSID_VXAPO_POST_MIX = B4A97313-ABC0-45ED-9C33-428B20D39428`（用户确定）；
  涉及 `vxapo.def` / `sys/com/apo_types` / `install/device/slots` / `object/vx_reg_props` 同步更新——**对应章节**：`object 7.5`

> 对应 commit：`a243e7f`

## v7.4 — 2026-08-02

变更类型：`缺陷修复`（P0-1/P0-2 实现反馈闭环 → config 规范修订）

- **实现反馈闭环机制**：新增「十七、实现反馈闭环」——执行端实现中发现规范不可行/遗漏 → 追加反馈段（不改状态）→ 规范侧修订 → 合规核对后标记 Done；零容忍绕过——**对应章节**：`主规范 十七`
- **config 6.1 修订 current_file**：`ParseContext.current_file: &'a Path` → `PathBuf`（Include 子解析需独立持有子文件路径，借用跨递归层不安全，旧 `Box::leak` 致泄漏）——**对应章节**：`config 6.1`
- **config 6.1 补充 REW 动态命令名分发**：REW `Filter N:`（如 `Filter 12:`）命令关键字动态，静态 match 无法命中，补充 `starts_with("filter ")` 前缀分支——**对应章节**：`config 6.1`
- **config 6.3 注册意图澄清**：`register_all_commands` 只注册 DSP 工厂——纯配置命令由 6.1 静态分发（`FilterFactory::create_filter` 只收 value 不含命令关键字，config 命令工厂注册后永远无法命中）——**对应章节**：`config 6.3`

> 对应 commit：`bf7a08e`

## v7.3 — 2026-08-01

变更类型：`实现对齐`（P0-4 配置热重载全链路规范补齐）

- **watcher 启动约定**：Initialize 中按 `config_path` 父目录启动 `ConfigWatcher`（轮询 2000ms、去重 500ms，`config 6.2`）；`ConfigFileChanged/Deleted` 事件经 `hot_reload`（R2 阻塞式 + R1 退役链）处理——**对应章节**：`object 7.1.8`
- **watcher 生命周期**：与 APO 实例一致（`ApoObject.watcher` 字段持有）；`UnlockForProcess`/`Reset` 不停止 watcher（热重载跨锁定周期持续生效）——**对应章节**：`object 7.1.8`

> 对应 commit：`a2c8d6c`

## v7.2 — 2026-08-01

变更类型：`实现对齐`（P0-3 per-device 配置路径规范补齐）

- **sys 新增 known_folder**：`sys/known_folder.rs` 封装 `SHGetKnownFolderPath(FOLDERID_Documents)` 已知文件夹解析（RAII 释放 CoTaskMem 内存），只做 FFI 收窄不拼接路径——**对应章节**：`sys 3.6`
- **APOInitSystemEffects re-export**：`sys/com/apo_types` 增加 `APOInitSystemEffects`（Initialize 初始化数据，提取端点 GUID）——**对应章节**：`sys 3.3.1b`
- **Initialize 补全 per-device 路径解析**：`APOInitSystemEffects` 反查端点 GUID → `guid_to_string` 大写格式化 → `Documents\VxAPO\{GUID}\config.txt`；目录自动创建、config 缺失写默认 passthrough、无 GUID 兜底 `_default`——**对应章节**：`object 7.1.8`
- **引用约束总表同步**：主规范增加 `sys/known_folder.rs` 行、`object/apo.rs` 增 `sys/known_folder` 依赖——**对应章节**：`主规范 十一`

> 对应 commit：`8dd42e2`

## v7.1 — 2026-08-01

变更类型：`实现对齐`（P0-1 DllRegisterServer 规范补齐）

- **regsvr32 职责边界澄清**：`DllRegisterServer` 无设备参数，只做全局 COM 类注册（2 个 CLSID 的 COM 类键 + ThreadingModel）；设备挂载/FxProperties 绑定归 `install_endpoint`，经 `vxapo-cli install -d` 触发——**对应章节**：`object 7.6`
- **DllRegisterServer 完整流程**：注册顺序（PostMix→PreMix）+ 幂等覆盖 + 失败逆序回滚（`SELFREG_E_CLASS`）+ 禁止触碰 MMDevices/FxProperties——**对应章节**：`object 7.6`
- **DllUnregisterServer 幂等**：键不存在视为成功（重复 `regsvr32 /u` 安全），尽力清理——**对应章节**：`object 7.6`
- **dll_exports 依赖补全**：引用约束总表增加 `sys/registry`（CLSID 键写入）——**对应章节**：`主规范 十一`、`object 7.6`

> 对应 commit：`895d9cd`

## v7.0 — 2026-08-01

变更类型：`结构重构`（引入路线清单治理机制，v6.9 → v7.0 major 递增）

- **路线清单驱动**：新增 `roadmap.md`（"要做什么"的规划），登记 P0-1..4 / P1-1..4 首版条目；任何新功能必须先登记，禁止"规范外直接实现"——**对应章节**：`主规范 十六`
- **状态机 + 硬门禁**：`Backlog → Spec-Drafting → Spec-Finalized → Implementing → Done`；仅 `Spec-Finalized` 可进入 `Implementing`（roadmap-rule 强制）——**对应章节**：`主规范 十六`、`.clinerules/roadmap-rule.md`
- **三文件联动**：一次版本变更须同时满足 roadmap 状态更新 + 规范章节落地 + changelog 记录，changelog 版本号 == 主规范版本号 == roadmap 落点版本——**对应章节**：`主规范 十六`、`.clinerules/roadmap-rule.md`
- **配套 skill 修正**：`roadmap-add-item`（仅登记，Backlog 不写 changelog，依赖检查推迟到 Spec-Drafting 前）；`spec-finalize`（次版本默认递增、changelog 严格走 changelog-rule、补三文件一致性校验、无法合规停留 Spec-Drafting）——**对应章节**：`.clinerules/skills/roadmap-add-item/SKILL.md`、`.clinerules/skills/spec-finalize/SKILL.md`

> 对应 commit：`f59f623`

## v6.9 — 2026-08-01

变更类型：`外部借鉴`（EqualizerAPO FilterEngine 过渡/重载机制深度分析）

- **R1 退役链延迟析构**：过渡完成帧 RT 线程仅 `retired_chain = outgoing_chain.take()`（零析构），控制线程锁内统一 drop——重型滤波器析构绝不留在 RT 线程——**对应章节**：`object 7.1.3/7.1.9/7.1.10/7.1.11/7.1.13`
- **R2 阻塞式重载**：hot_reload 检测到过渡在途/加载中即返回不构建，过渡完成由 APOProcess 触发重载；`reloading` 标志防覆盖——**对应章节**：`object 7.1.3/7.1.11/7.1.18`
- **R3 空链快路径**：`Chain::is_empty()` 时 `process_audio` 直接复制去交织结果跳过链遍历——**对应章节**：`pipeline 4.5/4.6`
- **R4 过渡周期 10ms**：`default_smoothing_length` 由 `sample_rate/20`（50ms）→ `sample_rate/100`（10ms）——**对应章节**：`pipeline 4.11`

> 对应 commit：`7aa6f16`

## v6.8 — 2026-08-01

变更类型：`外部借鉴`（EqualizerAPO Device 层）

- **E3.1 默认设备判定边界**：driver 层 `enumerate_devices()` 仅返回合法安装容器，不判定默认设备；用户层经 COM `GetDefaultAudioEndpoint` 对应——**对应章节**：`install 5.4`
- **E3.2 设备物理状态谓词**：`DeviceInfo::is_disabled()` / `is_unplugged()`（由已有 `EndpointState` 推导）——**对应章节**：`install 5.4`
- **E3.3 autoAdjust 独立字段**：`InstallConfig::auto_adjust`（默认 false），Step 4 写注册表读取——**对应章节**：`install 5.5.2`
- **E3.4 安装自检**：`install_endpoint(..., verify)`——commit 后 `CoCreateInstance` 验证 DLL 可实例化——**对应章节**：`install 5.5.2`
- 修复 install 5.5.2 多余代码围栏（用户修复）——**对应章节**：`install 5.5.2`

> 对应 commit：`bac344e`（+ 用户围栏修复 `b162890`）

## v6.7 — 2026-07-31

变更类型：`外部借鉴`（EqualizerAPO `IFilter::getInPlace`）

- **E1 就地处理声明**：`Filter::is_in_place()`（默认 true）+ `Chain::is_fully_in_place()`——全链就地时 `temp_buffers` 即最终输出，零拷贝快路径——**对应章节**：`pipeline 4.9/4.5/4.6`
- 与 v6.6 的 O1（RealtimeContext）/ 去交织架构完全兼容——**对应章节**：`pipeline 4.7/4.9`

> 对应 commit：`415ef53`

## v6.6 — 2026-08-01

变更类型：`外部借鉴`（tympan-apo）

- **O1 RT 编译期见证**：`RealtimeContext` 零尺寸标记 + `DspContext::rt_marker`（`PhantomData<RealtimeContext>`）——编译期能解决的问题绝不拖到运行时——**对应章节**：`pipeline 4.7/4.9`
- **O2 StateCell 补全**：`release()`（任意态→Created）+ `TransitionError{expected, attempted, actual}` + 语义化转换——**对应章节**：`object 7.1.4`
- **O3 生产构建约束**：release 必须 `panic="abort"` + `codegen-units=1`（RT 跨 FFI unwind = UB）——**对应章节**：`主规范 十五`
- **O4 AEC 接口预留**：`feature = "aec"` 门控 3 个 AEC 接口 + IID 常量——**对应章节**：`sys 3.2`

> 对应 commit：`a6ca7b5`

> **范围说明**：本 changelog 自 v6.6 开始记录（v6.0-v6.5 不补录）。维护规则见 `.clinerules/changelog-rule.md`。
