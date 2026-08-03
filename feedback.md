# Feedback 记录

> 执行端实现中发现的规范问题反馈，统一记录于此；**禁止写入 `roadmap.md`**（roadmap 只留引用）。
> 维护规则见 `.clinerules/feedback-rule.md`。
> 逆序插入（最新反馈在最上）；每条含 PX-X + 影响版本 + 问题 + 规范侧判定 + 修订记录 + 状态。

---

### [P0-6-2] 子 APO GUID 来源三处规范内部冲突（v8.4 + 用户指示确认实现方向）

**影响版本**：规范 v8.1 起（object 7.1.8/7.2 + install 5.5.2 涉及）

**问题**：
- 分类：用户决策/规范侧（执行端实现前置——子 APO GUID 来源存在规范内部冲突，需先消解再实现）
- ① **object 7.1.8 步骤 3「从 APOInitSystemEffects 提取子 APO CLSID」不可行**——`APOInitSystemEffects` **无任何子 APO 字段**（仅 APOInit/pAPOEndpointProperties/pAPOSystemEffectsProperties/pReserved/pDeviceCollection；EAPO 也从 APOInitSystemEffects 只取端点 GUID，再查注册表）
- ② **object 7.2「读 childApoPath{deviceGuid}」vs install 5.5.2「FxProperties 下 childGuid」**——两处存储位置矛盾，且 EAPO 实际用**独立** `childApoPath` 路径（DeviceAPOInfo.cpp 43/332-337），非 FxProperties
- ③ **路径冲突（用户补充指示）**：EAPO `APP_REGPATH = HKLM\SOFTWARE\EqualizerAPO`，`childApoPath = ...\Child APOs`（RegistryHelper.h 33）——**VxAPO 不能复用该路径**（污染 EAPO 安装信息区，EAPO 读到 VxAPO 写的值会混乱）

**规范侧判定**：
- 已确认缺陷（对应章节）——采取**方案 B**：规范侧修订 7.1.8/7.2 明确运行期读取路径，且**路径隔离**（VxAPO 用独立 `HKLM\SOFTWARE\VxAPO\Child APOs`，对齐 EAPO 机制、不共用其路径）

**修订记录**：
- v8.4：object 7.1.8 步骤 3 修正（APOInit 无子 APO 字段 → 端点 GUID → `HKLM\SOFTWARE\VxAPO\Child APOs\{deviceGuid}\{PreMixChild|PostMixChild}`）+ 引用来源新增 `install/device/slots` 依赖声明 + 引用约束总表同步；object 7.1.2 child_apo_clsid 注释修正；object 7.2 子 APO 来源更新（**VxAPO 独立路径** + 禁止读写 EAPO `HKLM\SOFTWARE\EqualizerAPO`）；install 5.3 slots.rs 补子 APO 安装信息区读取职责 + 路径隔离注；install 5.5.2 安装流程 Step 1/4 + 卸载流程 Step 3 更新（VxAPO 独立路径值名 PreMixChild/PostMixChild，废弃含糊「childGuid」）——**对应章节**：`object 7.1.2/7.1.8/7.2`、`install 5.3/5.5.2`

**状态**：已修订（v8.4）

---

### [P0-6-1] EAPO 源码逐行查验揭示 5 处规范偏差（v8.3 + 用户要求查验 `D:\Source_Code\equalizerapo-code`）

**影响版本**：规范 v8.1 起（主规范 18 / install 5.5.2 / object 7.x 涉及）

**问题**：
- 分类：规范侧自查（用户要求查验 EAPO 源码后确认）
- S1：「协商期锁死 in==out」表述失准——EAPO 实际是**拒绝下混**（`in>out 且 >2ch` → 创建 outFormat + S_FALSE；EqualizerAPO.cpp 277-282）+ **仅 mono→stereo 上混补做**（FilterConfiguration.cpp 126-128）
- S2：E3.4 `testAPOInstallation` 描述与源码不符——实际激活 `IAudioClient`（GetDevice → Activate → GetMixFormat → Initialize 共享模式 100ms；DeviceAPOInfo.cpp 777-815），非「CoCreateInstance 验证 DLL 可实例化」
- S3：child 委托失败销毁语义未覆盖——EAPO `IsInputFormatSupported` 委托失败会 `resetChild()` 销毁 child 降级（EqualizerAPO.cpp 258-283）
- S4：GetLatency 无 child 返回 0——EAPO `*pTime=0` 后仅 child 委托改写（82-95）；VxAPO「无 child 走自身」误读
- S5：无冒号行语义——EAPO 静默跳过（FilterEngine.cpp 329-330）vs VxAPO 拒绝报错 = **有意差异**（VxAPO 更严格，intent「语法严格性」）需在 config 6.1 标注

**规范侧判定**：
- 已确认缺陷（对应章节）——S1-S4 属规范文本/语义失准需修订；S5 属有意差异需标注 EAPO 行为事实

**修订记录**：
- v8.3：新建《Equalizer 行为文档.md》（实际行号版 + 已确认/推断分离）；主规范 18.1 A4/A5 修正（GetLatency 无 child=0 / child 就地由 is_in_place 决定）、18.2 D2 修正（拒绝下混 + 仅 mono→stereo 上混）、18.2 D5 补 child 失败 resetChild 销毁、18.2 补正说明 v8.3；install 5.5.2 E3.4 修正（IAudioClient 管线自检）；object 7.1.14 GetLatency 伪代码修正（无 child 返回 0）；object 7.2 补 GetLatency 委托语义——**对应章节**：`Equalizer 行为文档.md`、`主规范 18.1/18.2`、`install 5.5.2`、`object 7.1.14/7.2`

**状态**：已修订（v8.3）

---

### [P0-5-1] P0-5 定稿说明「杜绝 panic 跨 FFI 崩溃」语义失准（v8.3 + 用户质疑策略合理性）

**影响版本**：规范 v8.2 起

**问题**：
- 分类：用户决策/规范侧（用户无法理解「P0-5 目的是杜绝崩溃，但 release 下 catch_unwind 为空操作」）
- 原表述「杜绝 panic 跨 FFI unwind 到 audiodg 崩溃」失准——release（`panic="abort"`）下 panic 即**进程终止**（audiodg 由 Windows 音频服务重启恢复），非「崩溃」而是「确定性终止 + 无 UB」；真正危害是 unwind 跨 FFI = **UB（静默内存损坏）**——目标应为「杜绝 UB 传播」
- 「真防线是 panic hook + abort」表述错误：abort 是**兜底语义**；panic hook 仅 abort 前记录现场，为**诊断工具非防线**

**规范侧判定**：
- 已确认缺陷（对应章节）——P0-5 策略本身合理（三层防护组合），但定稿说明表述需重写为准确语义

**修订记录**：
- v8.3：roadmap P0-5 重写（三层防护各司其职：编译期 O1 不 panic → debug catch_unwind 验证防御路径 → release abort 确定性兜底；「杜绝 UB 传播」）；object 7.1.11/7.1.12 修正（三层防护 + panic hook 定位为诊断工具非防线）——**对应章节**：`roadmap P0-5`、`object 7.1.11/7.1.12`

**状态**：已修订（v8.3）

---

### [P0-4-4] v7.12 「未知命令」判定失效：Convolution 宽容解析接管未知命令（v7.11 + 执行端反馈）

**影响版本**：规范 v7.11 起

**问题**：
- 分类：实现缺陷发现
- 场景：`BogusCommand: x` 有冒号、value 非空 → parser `_` 分支落 `try_create(x)` → Convolution 的 `parse_convolution_params` 对任意非空字符串返回 Ok（路径即合法）→ 创建成功 → 永远到不了 Unmatched → v7.11「Unmatched → SyntaxError『未知命令』」判定失效
- 根因：parser 缺少「命令关键字白名单」校验；`try_create` 的 Unmatched 语义承载了「未知命令」判定，但该判定无法在 registry 内完成

**规范侧判定**：
- 已确认缺陷（对应章节）——需补「命令关键字白名单」三段式：冒号数量校验 → 命令名 ∈ 白名单（静态命令 / REW `Filter N:` 前缀 / `registry.factory_names()`）→ `try_create`；Unmatched 语义收窄为「已知命令的参数无效」

**修订记录**：
- v7.12：config 6.1 补白名单校验——命令名不在白名单 → `SyntaxError「未知命令」`不落 registry；Unmatched 文案改 `命令无效 'X'：参数无法解析`；VSTPlugin 特判「该命令当前未启用（预留）」不过 try_create；`split_command_value(trimmed)?` 补 `?` ——**对应章节**：`config 6.1`、`pipeline 4.20`

**状态**：已修订（v7.12）

---

### [P0-4-3] v7.11 语法严格化：无冒号行被 Convolution 宽容语义误接 + 参数多余 token 静默忽略（v7.11 + 用户产品决策）

**影响版本**：规范 v7.9 起

**问题**：
- 分类：实现完成报告随附潜在问题 + 用户产品决策
- （1）`Convolution: ir.wav -6 abc` 第 3 token 非数字，`parse_convolution_params` 只取前 2 token 静默忽略 `abc`——配置写错无反馈
- （2）v7.9 裸命令可达性（无冒号 → `try_create(cmd)`）让 `BogusCommand`（无冒号整行）被 Convolution 宽容语义接住 → 变「路径加载失败」而非「未知命令」；用户定性：**无冒号行不应被解析**——冒号前字符串决定解析目标，必须是严格关键字（对齐 EAPO）；「输入严格保证解析宽容」写错必有反馈

**规范侧判定**：
- 已确认缺陷（对应章节）——config 语法严格化：每行必须 `命令关键字: 参数`，无冒号行拒绝、一行仅一个冒号；Unmatched → SyntaxError（后续 v7.12 再收窄为白名单三段式）；Convolution 参数严格化；诊断日志归属软件安装根目录 `log/`（应用层呈现，DLL 只写非状态回传）

**修订记录**：
- v7.11：config 6.1 单冒号校验（0 冒号「缺少冒号」/ ≥2 冒号「多余冒号」）+ 裸命令修正反转；Unmatched → SyntaxError「未知命令」；pipeline 4.19 `parse_convolution_params` `Option`→`Result`（≥3 tokens 报错）；pipeline 4.20 VST 静默 NoMatch 对齐；intent.md 补「config.txt 语法严格性」+「诊断日志归属」（log/ 安装根目录）——**对应章节**：`config 6.1`、`pipeline 4.19/4.20`、`intent.md`
- v7.12：白名单三段式补充（见 [P0-4-4]）——**对应章节**：`config 6.1`

**状态**：已修订（v7.11 + v7.12 补充）

---

### [P0-4-2] v7.10 watcher 线程模型矛盾：new「启动线程」 vs 外部驱动（v7.10 + 执行端反馈）

**影响版本**：规范 v7.9 起

**问题**：
- 分类：规范文本矛盾
- config 6.2 `new` 注释「启动 watcher 线程」，但 `wait_and_handle`（"等待并处理一个事件……线程应退出"）暗示外部线程循环驱动；v7.9 生命周期声明「ConfigWatcher 由 ApoObject 持有、UnlockForProcess SetEvent+join」——若自启线程则 wait_and_handle 无调用者、shutdown 应 join 内部线程
- 附带：实现报告遗留「apo.rs 尚未创建 watcher 线程」——对象层接线缺口

**规范侧判定**：
- 已确认规范矛盾（对应章节）——确立**外部驱动模型**：`ConfigWatcher` 不自启线程，线程由调用方 `start_watcher` 创建并循环驱动；`shutdown` 不 join（join 由 `stop_watcher` 负责）
- 对象层接线：确认为 P0-4 剩余项（待执行端补做）

**修订记录**：
- v7.10：config 6.2 澄清「创建监控器（不启动线程）」；object 7.1.9 补 start_watcher 定义（CreateEventW → ConfigWatcher::new → spawn 循环 wait_and_handle → hot_reload + 失败降级）；object 7.1.10 补 stop_watcher 定义（SetEvent → join → shutdown + 幂等）；object 7.1.3 补 watcher_thread/watcher_shutdown_event 字段——**对应章节**：`config 6.2`、`object 7.1.3/7.1.9/7.1.10`

**状态**：已修订（v7.10）

---

### [P0-4-1] v7.9 目录级语义矛盾：FindFirstChangeNotificationW 拿不到文件名（v7.9 + 执行端反馈）

**影响版本**：规范 v7.8 起

**问题**：
- 分类：规范不可行
- config 6.2 事件驱动流程「校验文件名 == config.txt」与 `FindFirstChangeNotificationW` 语义矛盾——目录级通知不提供具体文件名（文件名信息只有 `ReadDirectoryChangesW` 扩展才有），无法逐文件过滤

**规范侧判定**：
- 已确认缺陷（对应章节）——不做文件名校验；目录任何变更 → hot_reload → 128KB 闸门 → 重新解析 + `parse_file_with_spec` 产出 spec chain → 与 active_spec 比较 → 相同幂等跳过 / 不同建新链过渡（用户方案：比较解析后配置指纹而非文件内容哈希——"配置实质变了"真实语义）

**修订记录**：
- v7.9：config 6.2 `WatchEvent` 统一 `DirectoryChanged`；config 6.1 `FilterSpec=String` + `parse_file_with_spec` 双返回 + 128KB 逐文件闸门 + 裸命令可达性修正（后 v7.11 反转）；object 7.1.18 spec 短路 + active_spec 更新；object 7.1.8/7.1.9/7.1.10 生命周期随锁定；新建 intent.md——**对应章节**：`config 6.1/6.2`、`object 7.1.3/7.1.8/7.1.9/7.1.10/7.1.18`、`intent.md`

**状态**：已修订（v7.9）

---

### [P0-4-0] IsInputFormatSupported 时序错误：协商早于 LockForProcess（v7.7 + 用户反馈）

**影响版本**：规范 v7.1 起

**问题**：
- 分类：用户反馈（实现缺陷发现）
- `IsInputFormatSupported` 在 `LockForProcess` **之前**被引擎调用，此时 `pipeline_context` 为 `PipelineContext::new()`（全零）——实装的等值比较（`fmt.channels == ctx.input_channels` && `fmt.sample_rate == ctx.sample_rate`）**永远不成立** → 拒绝所有格式、APO 无法协商

**规范侧判定**：
- 已确认缺陷（对应章节）——协商阶段禁止依赖 pipeline_context；正确做法为对请求格式做独立属性检查（浮点格式 + 44.1k~192k + 1~8 通道）

**修订记录**：
- v7.7：object 7.1.16 补关键时序约束——协商阶段禁止依赖 pipeline_context，独立属性检查——**对应章节**：`object 7.1.16`

**状态**：已修订（v7.7）

---

### [P0-3-2] APOInitSystemEffects 实测结构不符：pAPOSystemEffectsProperties（v7.6 + 执行端反馈）

**影响版本**：规范 v7.2 起

**问题**：
- 分类：原型不可实现
- object 7.1.8 原型 `pSystemEffectsProperties->pEndpointGuid` 与 windows-rs 0.62.2 实测不符：`APOInitSystemEffects` 无 `pSystemEffectsProperties` 字段，而是 `pAPOSystemEffectsProperties: ManuallyDrop<Option<IPropertyStore>>`；端点 GUID 经 `IPropertyStore::GetValue(&PKEY_AudioEndpoint_GUID)` 返回 PROPVARIANT（VT_CLSID）的 `puuid` 提取

**规范侧判定**：
- 已确认缺陷（对应章节）——更新提取路径描述

**修订记录**：
- v7.6：object 7.1.8 + sys 3.3.1b + sys 3.3 引用来源更新为 pAPOSystemEffectsProperties（IPropertyStore）→ PKEY_AudioEndpoint_GUID（VT_CLSID puuid）；无 GUID 兜底 `_default`；Documents 失败兜底 `C:\ProgramData\VxAPO\config.txt`——**对应章节**：`object 7.1.8`、`sys 3.3.1b`

**状态**：已修订（v7.6）

---

### [P0-3-1] APOProcess 过渡缺陷（v7.6 + 独立发现 + 用户确认）

**影响版本**：规范 v6.9 起

**问题**：
- 分类：实现缺陷发现（P0-3 实现完成报告遗留）
- ① finished 分支先置 `reloading=true` 再调 hot_reload → 短锁检查 reloading==true 直接 return → 延迟重载被自己拦截
- ② transition=None 但 pending 残留（无混合器）→ 本帧不写输出，违反 APO 契约
- ③ advance() 返回 None（过渡已达上限）→ 本帧不写输出

**规范侧判定**：
- 已确认缺陷（对应章节）——需修正过渡完成分支

**修订记录**：
- v7.6：object 7.1.11 修正——过渡完成不置位 reloading 直接调用 hot_reload；pending 残留 → bypass 输出 + 旧链退役 + 补重载；advance None → factor=1.0（纯新链）输出——**对应章节**：`object 7.1.11`

**状态**：已修订（v7.6）

---

### [P0-2-3] REW 动态命令名未覆盖：Filter N:（v7.4 + 执行端反馈）

**影响版本**：规范 v7.1 起

**问题**：
- 分类：规范遗漏
- 6.1 逐行分发逻辑未描述 REW `Filter N:` 动态命令名（如 `Filter 1:`/`Filter 12:`），静态 match 无法命中

**规范侧判定**：
- 已确认遗漏（对应章节）——补动态命令名分发路径

**修订记录**：
- v7.4：config 6.1 补充 `starts_with("filter ")` 前缀分支 → rew::handle——**对应章节**：`config 6.1`

**状态**：已修订（v7.4）

---

### [P0-2-2] ParseContext.current_file 借用冲突（v7.4 + 执行端反馈）

**影响版本**：规范 v7.1 起

**问题**：
- 分类：原型不可实现
- `ParseContext.current_file: &'a Path` 与 Include 递归冲突——子解析需独立持有子文件路径，借用 `&'a Path` 无法跨递归层安全表达（旧实现 Box::leak 绕过后泄漏）

**规范侧判定**：
- 已确认缺陷（对应章节）——改为所有权 PathBuf

**修订记录**：
- v7.4：config 6.1 `ParseContext.current_file: &'a Path` → `PathBuf`（消除泄漏）——**对应章节**：`config 6.1`

**状态**：已修订（v7.4）

---

### [P0-2-1] 规范 6.3 与 6.1 矛盾：register_all_commands 注册意图（v7.4 + 执行端反馈）

**影响版本**：规范 v7.1 起

**问题**：
- 分类：规范文本矛盾
- 6.3 `register_all_commands` 列出 7 个 config 命令工厂注册进 FilterRegistry；但 6.1 逐行分发逻辑明确 Device/Stage/Channel/Eval/Include 走 handle_* 静态分发、未知命令走 registry——`FilterFactory::create_filter` 只接收冒号后的 value（不含命令关键字），config 命令工厂注册后永远无法命中

**规范侧判定**：
- 已确认矛盾（对应章节）——register_all_commands 只注册 DSP 工厂，纯配置命令由 6.1 静态分发

**修订记录**：
- v7.4：config 6.3 改为只注册 DSP 工厂（9 个），纯配置命令静态分发——**对应章节**：`config 6.3`

**状态**：已修订（v7.4）