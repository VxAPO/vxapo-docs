# Equalizer 行为文档（源码确认版）

> **目的**：记录已**通读源码确认过**的 EqualizerAPO（EAPO）行为，作为 VxAPO 对齐/差异化决策的**事实依据**。
> **状态**：随源码精读持续更新；**严格区分「已确认」（有源码位置）与「推断/待确认」**。
> **依据**：2026-08-03 实读 `D:\Source_Code\equalizerapo-code`；本轮已逐行查验 EqualizerAPO.cpp / DeviceAPOInfo.cpp / FilterEngine.cpp/.h / FilterConfiguration.cpp / parser/RegistryFunctions.cpp / IFilter.h。

---

## 一、源码清单（已通读文件 + 实际行号）

| 文件 | 已确认行为区间 | 对应 VxAPO 落点 |
|------|----------------|-----------------|
| `EqualizerAPO/EqualizerAPO.cpp` | 82-95（GetLatency） | object 7.2 |
| `EqualizerAPO/EqualizerAPO.cpp` | 149-226（Initialize + 子 APO 创建） | object 7.1.8 / 7.2 |
| `EqualizerAPO/EqualizerAPO.cpp` | 228-306（IsInputFormatSupported 委托/回落/下混拒绝） | object 7.1.7 / 7.2 |
| `EqualizerAPO/EqualizerAPO.cpp` | 308-389（LockForProcess + realChannelCount） | object 7.1.9 / 主规范 18.2 |
| `EqualizerAPO/EqualizerAPO.cpp` | 391-404（UnlockForProcess） | object 7.1.10 / 7.2 |
| `EqualizerAPO/EqualizerAPO.cpp` | 406-425（resetChild） | object 7.2 |
| `EqualizerAPO/EqualizerAPO.cpp` | 457-517（APOProcess） | object 7.1.11 |
| `DeviceAPOInfo.cpp` | 82-233（枚举/默认设备） | install 5.4（E3.1） |
| `DeviceAPOInfo.cpp` | 250-260（installPostMix=!input / useOriginal 默认） | object 7.2 |
| `DeviceAPOInfo.cpp` | 500-576（槽位备份 + childApoPath） | object 7.2 |
| `DeviceAPOInfo.cpp` | 777-815（testAPOInstallation） | install 5.5.2（E3.4） |
| `FilterEngine.cpp` | 79（loadSemaphore）、141-213（initialize + 通知线程创建） | object 7.1.9 / 主规范 18.2 |
| `FilterEngine.cpp` | 215-269（loadConfig + previousConfig 析构） | object 7.1.18 |
| `FilterEngine.cpp` | 271-373（loadConfigFile：读失败返回 + 无冒号跳过） | config 6.1 |
| `FilterEngine.cpp` | 375-378（watchRegistryKey）、381-455（process/doTransition 调用） | object 7.2 / pipeline 4.11 |
| `FilterEngine.cpp` | 464（getInPlace 调用）、556-624（notificationThread） | config 6.2 / pipeline 4.9 |
| `FilterEngine.h` | 129-135（realChannelCount 字段注释） | 主规范 18.2 |
| `FilterConfiguration.cpp` | 31-52（缓冲预分配）、121-128（mono→stereo）、157-176（doTransition） | pipeline 4.11 / 主规范 18.2 |
| `parser/RegistryFunctions.cpp` | 30-58（readRegString→watchRegistryKey）、70-98（readRegDWORD→watchRegistryKey） | object 7.2 |
| `IFilter.h` | 41（getInPlace 声明） | pipeline 4.9（E1） |

---

## 二、已确认行为清单（每条均经源码逐行核实）

### 2.1 子 APO 生命周期

| # | 行为（源码确认） | 源码位置 | VxAPO 对齐 |
|---|------------------|----------|------------|
| C1 | **Initialize 读取子 APO GUID**：`APOInitSystemEffects.APOInit.clsid` 判断 PreMix/PostMix（122）；`DeviceAPOInfo` 读 `PreMixChildGuid/PostMixChildGuid`（160-163） | EqualizerAPO.cpp 122/149-163 | **完全对齐**：object 7.1.8 子 APO 来源 = 接管槽位前任 |
| C2 | **创建流程**：`CLSIDFromString` → `CoCreateInstance(childGuid, NULL, CLSCTX_INPROC_SERVER, IID_IAudioProcessingObject)` → QI `IAudioProcessingObjectRT` → QI `IAudioProcessingObjectConfiguration` | EqualizerAPO.cpp 178-209 | **完全对齐**：object/child.rs create 三步 |
| C3 | **Initialize 透传**：`childAPO->Initialize(cbDataSize, pbyData)` **同传父 APOInit 数据** | EqualizerAPO.cpp 211 | **完全对齐**：object 7.1.8 |
| C4 | **创建失败降级**：任一步失败（CoCreate/QI RT/QI Cfg/Initialize）→ `resetChild()` → **返回 S_OK**（不阻塞父） | EqualizerAPO.cpp 186-217 | **完全对齐**：child_apo=None 降级（object 7.1.8 Note 57） |
| C5 | **GetLatency**：`*pTime=0` → **有 child 才委托** `childAPO->GetLatency(pTime)` → 返回 S_OK；**无 child 时返回 0**（**非**「走自身延迟」） | EqualizerAPO.cpp 82-95 | **对齐（修正）**：object 7.2 GetLatency 委托；无 child 返回 0 |
| C6 | **IsInputFormatSupported 委托**：有 child → `childAPO->IsInputFormatSupported`（260）；**child 失败 → resetChild() 销毁 child**（268）→ 回落自身检查（272-283） | EqualizerAPO.cpp 258-283 | **对齐 + 补充**：child 失败会**销毁**（非仅回落）；VxAPO 需对齐 resetChild 语义 |
| C7 | **LockForProcess 委托**：有 childCfg → `childCfg->LockForProcess`（结果**仅 Trace 不 return**，343-346）→ 随后父自身锁定 | EqualizerAPO.cpp 341-350 | **对齐**：object 7.1.9 子 APO 拒绝锁定时降级 |
| C8 | **UnlockForProcess 失败语义（EAPO 侧）**：`childCfg->UnlockForProcess` FAILED → **return hr**（父解锁失败） | EqualizerAPO.cpp 393-401 | **VxAPO 容错差异**（object 7.1.10）：子解锁失败不阻塞父 |
| C9 | **resetChild 语义**：三接口（childAPO/childRT/childCfg）逐个 `Release()` + 置 NULL | EqualizerAPO.cpp 406-424 | **对齐**：ChildApo Drop（object 7.2） |

### 2.2 APOProcess 委托与时序

| # | 行为（源码确认） | 源码位置 | VxAPO 对齐 |
|---|------------------|----------|------------|
| C10 | **APOProcess 委托时序**：**`childRT->APOProcess` 先跑**（作用于输入缓冲）→ `engine.process(outputFrames, outputFrames, ...)`（**输出缓冲就地处理**） | EqualizerAPO.cpp 472-477 | **完全对齐**：object 7.1.11 child 前置每帧一次 |
| C11 | **BUFFER_SILENT 输入清零**：输入为 SILENT 时先 `memset` 输入缓冲（468-470）→ childRT 处理 → silent 输出判定（allowSilentBufferModification 时 486-499 / 否则输出清零 + BUFFER_SILENT 503-505） | EqualizerAPO.cpp 468-511 | **对齐**：VxAPO buffer_flags 语义（object 7.1.11） |
| C12 | **child 输出通道语义**：`realChannelCount` = 有 child 时 **outFormat 通道数**；无 child 时 **inFormat 通道数** | EqualizerAPO.cpp 365-369 | **等价立场**：VxAPO 不引入 realChannelCount 机制（主规范 18.2 D2） |
| C13 | **realChannelCount 含 child 输出**：字段注释「处理时输入通道数（含 child APO 输出）」 | FilterEngine.h 131-132 | **等价**：主规范 18.2 D2/D3 |

### 2.3 槽位安装与备份

| # | 行为（源码确认） | 源码位置 | VxAPO 对齐 |
|---|------------------|----------|------------|
| C14 | **installPostMix 默认 = !input**（渲染设备才装 PostMix） | DeviceAPOInfo.cpp 254 | **对齐**：object 7.2 |
| C15 | **useOriginal 默认**：`useOriginalAPOPreMix = true`、`useOriginalAPOPostMix = !input` | DeviceAPOInfo.cpp 255-256 | **对齐**：object 7.2 前任 APO 备份 |
| C16 | **备份全部槽位**：FxProperties 各槽位（allGuidValueNames）现值 → 写 `childApoPath\{deviceGuid}`（567-577 同区）——**含 NOKEY/NOVALUE 占位** | DeviceAPOInfo.cpp 529-548 | **对齐**：object 7.2 备份全部槽位供回退 |
| C17 | **原 APO 备份 `.reg` 文件**：FxProperties 存在时 `saveToFile` 写 `backup_*.reg`（551-553） | DeviceAPOInfo.cpp 550-553 | **VxAPO 差异**：VxAPO 无 `.reg` 文件备份（object 7.2 仅注册表备份）——留 P1 按需 |
| C18 | **autoAdjust 写入**：`autoAdjust=true` → 删 `disableAutoAdjust` 键；**false → 写 `disableAutoAdjust="true"`**（默认） | DeviceAPOInfo.cpp 567-575 | **对齐 + 保守默认**：VxAPO 默认 false（install 5.5.2 E3.3） |
| C19 | **allowSilentBuffer 写入**：写 `allowSilentBufferValueName`（true/false） | DeviceAPOInfo.cpp 565-566 | **对齐**：object 7.1.2 allow_silent_buffer_modification |

### 2.4 格式协商与通道语义

| # | 行为（源码确认） | 源码位置 | VxAPO 对齐 |
|---|------------------|----------|------------|
| C20 | **下混拒绝**：`inFormat > 2ch && inFormat.dwSamplesPerFrame > outFormat.dwSamplesPerFrame` → 创建 outFormat 媒体类型 + `S_FALSE`（**父不执行下混**） | EqualizerAPO.cpp 277-282 | **等价立场（修正表述）**：VxAPO 协商期锁死「收到的输入==输出通道数」（父不转换）；**「拒 in>out」即下混拒绝，非「锁死 in==out」** |
| C21 | **mono→stereo 上混特例**：`realChannelCount==1 && outputChannelCount>=2` → 复制 mono 到声道 1（注释「模拟无 APO 时 Windows 音频系统自动上混」） | FilterConfiguration.cpp 126-128 | **等价立场**：仅此上混特例，其他上混/下降混父均不执行（主规范 18.2） |
| C22 | **协商委托 + child 失败销毁回退**：child 成功用 child 判定；child 失败 **resetChild 销毁** + 回落自身 | EqualizerAPO.cpp 258-283 | **对齐 + 收窄**：VxAPO 白名单三段式（主规范 18.2 D5） |

### 2.5 配置解析行为

| # | 行为（源码确认） | 源码位置 | VxAPO 对齐 |
|---|------------------|----------|------------|
| C23 | **无冒号行跳过**：`pos = line.find(':')`；**`pos==-1` → 整行跳过不解析**（无错误、不拒绝） | FilterEngine.cpp 329-330 | **对齐（修正）**：VxAPO v7.11「无冒号行拒绝」**比 EAPO 更严格**——EAPO 静默跳过，VxAPO 显式报错（intent「语法严格性」有意差异） |
| C24 | **读文件失败返回**：`CreateFile` 失败（非共享冲突）→ `return`（保留当前配置不替换） | FilterEngine.cpp 285-287 | **对齐**：object 7.1.18 解析失败保留旧链 |
| C25 | **配置文件等待语义**：写文件过程中 `ERROR_SHARING_VIOLATION` → `Sleep(1)` 重试（非报错） | FilterEngine.cpp 282-292 | **VxAPO 差异**：VxAPO 128KB 闸门 + spec 比对（config 6.1）；EAPO 无大小闸门 |

### 2.6 配置加载与过渡（FilterEngine / FilterConfiguration）

| # | 行为（源码确认） | 源码位置 | VxAPO 对齐 |
|---|------------------|----------|------------|
| C26 | **previousConfig 延迟析构**：`loadConfig` **开头**先析构 previousConfig（控制线程） | FilterEngine.cpp 219-224 | **对齐**：R1 退役链延迟析构（object 7.1.3/7.1.11） |
| C27 | **loadSemaphore 阻塞式重载**：创建 1/1 信号量（79）；过渡完成 `ReleaseSemaphore`（413/452）→ 通知线程等待 semaphore 再 loadConfig（606-614） | FilterEngine.cpp 79/413/452/606-614 | **对齐**：R2 阻塞式重载（object 7.1.18） |
| C28 | **transitionLength = sampleRate/100（10ms）** | FilterEngine.cpp 154 | **对齐**：R4（pipeline 4.11） |
| C29 | **EAPO 过渡是配置级**：新旧两套 allSamples 平面按帧混合 `doTransition`（升余弦）；**`transitionCounter >= transitionLength` 后释放下一配置**（407-414） | FilterConfiguration.cpp 157-176 + FilterEngine.cpp 402/407-414 | **VxAPO 差异（D1）**：父内双链过渡（outgoing/current mix_buffers） |
| C30 | **缓冲预分配**：FilterConfiguration 构造按 `maxFrameCount` 预分配 allSamples/allSamples2（RT 零分配） | FilterConfiguration.cpp 31-52 | **对齐**：object 7.1.9 LockForProcess 预分配 temp_buffer_old/new |
| C31 | **通知线程创建时机**：`engine.initialize` 末尾、`configPath != ""` 时创建（**LockForProcess 路径调用 initialize**） | FilterEngine.cpp 201-205 | **对齐**：object 7.1.9 watcher 生命周期随锁定周期 |
| C32 | **目录变更通知**：`FindFirstChangeNotificationW(configPath, 递归, 文件名|最后写入)` + **10ms 去重**（599-604）+ 等 loadSemaphore 后 loadConfig（606-614） | FilterEngine.cpp 556-624 | **对齐**：config 6.2 目录级事件驱动 + 10ms 去重 |
| C33 | **watchRegistryKey 是 readReg 副作用**：`readRegString`（52）/`readRegDWORD`（92）成功读取后 → `engine->watchRegistryKey(key)` → notificationThread `RegNotifyChangeKeyValue` 监视（577） | parser/RegistryFunctions.cpp 52/92 + FilterEngine.cpp 375-378/577 | **VxAPO 不需要**：config 纯文件无 readReg 命令（object 7.2） |
| C34 | **isEmpty 空链快路径**：`currentConfig->isEmpty() && nextConfig==NULL` → 直接 memcpy（避免去交织成本） | FilterEngine.cpp 383-393/419-431 | **对齐**：R3 `Chain::is_empty()` 快路径（pipeline 4.5/4.6） |
| C35 | **getInPlace 就地声明**：`IFilter::getInPlace()`（默认 true）；addFilters 读取 `filterInfo->inPlace`（464） | IFilter.h 41 + FilterEngine.cpp 464 | **对齐**：E1 `Filter::is_in_place()`（pipeline 4.9） |

### 2.7 安装与设备（E3.x）

| # | 行为（源码确认） | 源码位置 | VxAPO 对齐 |
|---|------------------|----------|------------|
| C36 | **默认设备判定边界（E3.1）**：driver 层枚举（`loadAllInfos`）仅返回安装容器；默认设备由 `GetDefaultAudioEndpoint`（**用户层/API 层**）取得后与 driver 结果对应 | DeviceAPOInfo.cpp 82-87/106-116 | **对齐**：install 5.4 E3.1 |
| C37 | **testAPOInstallation（E3.4 修正）**：激活 `IAudioClient`（Activate + GetMixFormat + Initialize 共享模式 100ms）——**全音频管线自检**，**非**「CoCreateInstance 验证 DLL 可实例化」；失败 `fail()` 抛 `DeviceException`（异常上抛） | DeviceAPOInfo.cpp 777-815 | **VxAPO 需修订 E3.4 描述**：install 5.5.2「CoCreateInstance 验证 DLL」**与源码不符**——EAPO 实际激活 IAudioClient 做管线自检 |

---

## 三、VxAPO 对齐结论汇总（源码确认依据）

| 维度 | VxAPO 立场 | 源码依据 |
|------|-----------|----------|
| 子 APO 创建/降级/Initialize 透传/GetLatency | 完全对齐（GetLatency 无 child=0） | C1-C6、C9 |
| IsInputFormatSupported 委托 + child 失败销毁 | 对齐 + 补充 resetChild 语义 | C6、C22 |
| APOProcess 委托时序（child 前置 + 输出缓冲就地处理） | 完全对齐 | C10-C11 |
| 通道语义（下混拒绝 + mono→stereo 特例 + realChannelCount） | 等价（**修正「锁死 in==out」表述为「拒 in>out 下混」**） | C12-C13、C20-C21 |
| 槽位安装/备份/autoAdjust/allowSilentBuffer | 完全对齐 | C14-C19 |
| 过渡机制 | 差异有意（父内双链 vs EAPO 配置级单链） | C29 |
| 配置解析/加载/失败语义/预分配/监控 | 对齐（无冒号行 VxAPO 更严格为有意差异） | C23-C25、C26-C32 |
| 注册表监视 | **VxAPO 不需要**（无 readReg 命令） | C33 |
| 安装自检 | **E3.4 描述需修订**（实际是 IAudioClient 管线自检） | C37 |

---

## 四、推断/待确认（非独立源码事实——不视为 EAPO 行为，仅供 VxAPO 决策参考）

> 以下条目**无源码位置直接证实**，来自 VxAPO 规范对齐声明或推断，**不得**作为「EAPO 事实」引用。

| # | 表述 | 来源 | 状态 |
|---|------|------|------|
| P1 | **child 无独立缓冲、就地改写输入缓冲** | 主规范 18.1 A5 推断 | **部分证实**：APOProcess 中 childRT 先跑作用于输入/输出缓冲（472-477）；「child 自身是否就地」由 child 的 is_in_place 决定，EAPO 不强制 |
| P2 | **child 无报告通道数接口，父按 outFormat 假设布局** | 主规范 18.2 D3 推断 | **部分证实**：realChannelCount=outFormat（365-369）隐含「父信任 child 已就位输出布局」；「child 无接口」未直接证实 |
| P3 | **过渡期 child 不变**（child 不参与 EAPO 配置级过渡） | 主规范 18.2 D1 推断 | **待确认**：doTransition 仅混合 allSamples（FilterConfiguration.cpp 157-176）；child 调用在 APOProcess 层（EqualizerAPO.cpp 474），过渡期间 child 确实**每帧仍被调用**——「不变」表述需谨慎 |
| P4 | **严格关键字语法：EAPO 无冒号行亦不解析** | v7.11 对齐 EAPO 声明 | **已证实（修正）**：EAPO 无冒号行**静默跳过**（FilterEngine.cpp 329-330）；VxAPO「拒绝报错」为**有意更严格**，非对齐 |
| P5 | **EAPO 所有缓冲区 initialize 阶段预分配，RT 零分配** | object 7.1.9 v7.8 对齐声明 | **已证实**：FilterConfiguration 构造按 maxFrameCount 预分配（FilterConfiguration.cpp 31-52）；RT 路径（read/process/write/doTransition）零分配 |
| P6 | **startMonitorThread 在 LockForProcess 内调用** | object 7.1.9 对齐声明 | **已证实**：engine.initialize 末尾创建通知线程（FilterEngine.cpp 201-205）；initialize 由 LockForProcess 路径调用 |
| P7 | **EAPO 失败「报告非回滚」** | install 5.5.2 E3.4 对齐声明 | **已证实 + 修正**：testAPOInstallation 失败抛异常（DeviceAPOInfo.cpp 817-822），非「静默报告」；且自检对象是 IAudioClient 非 DLL |

---

## 五、源码查验揭示的规范偏差（需修订 VxAPO 规范）

| # | 偏差 | 源码依据 | 需修订落点 |
|---|------|----------|------------|
| S1 | **「协商期锁死 in==out」表述失准**：EAPO 实际是**拒 in>out（下混）** + **mono→stereo 上混补做**；in<out 且非 mono→stereo 的上混父不执行——VxAPO「等价立场」表述应为「**拒绝下混 + 仅 mono→stereo 上混**」 | EqualizerAPO.cpp 277-282 + FilterConfiguration.cpp 126-128 | 主规范 18.2 D2 + roadmap P0-6 ③ |
| S2 | **E3.4 testAPOInstallation 描述不符**：实际激活 `IAudioClient`（全管线自检），非「CoCreateInstance 验证 DLL」 | DeviceAPOInfo.cpp 777-815 | install 5.5.2 E3.4 |
| S3 | **child 失败销毁语义未覆盖**：EAPO `IsInputFormatSupported` 委托失败会 `resetChild()`（销毁 child 降级），VxAPO object 7.1.7 未明确 | EqualizerAPO.cpp 258-283 | object 7.1.7 + 主规范 18.2 D5 |
| S4 | **GetLatency 无 child 返回 0**：EAPO `*pTime=0` 后仅 child 委托改写；VxAPO「无 child 走自身」需明确为「返回 0」 | EqualizerAPO.cpp 82-95 | object 7.2 |
| S5 | **无冒号行语义**：EAPO 静默跳过 vs VxAPO 拒绝报错——**有意差异**需在 config 6.1 标注（EAPO 行为事实 + VxAPO 严格化理由） | FilterEngine.cpp 329-330 | config 6.1 + intent「语法严格性」 |

---

## 六、未确认/待精读事项（不写入行为清单）

> 以下事项**尚未通读源码确认**，不视为 EAPO 行为事实：

- 滤波器 DSP 内部的逐样本处理细节（GraphicEQ/BiQuad/IIR 等 filters/ 目录）
- VSTPlugin 加载路径（helpers/VSTPluginInstance.cpp）
- 默认设备判定边界的完整流程（E3.1 已确认 driver 枚举 + GetDefaultAudioEndpoint 边界，但 Configurator 交互未读）
- 其他文件（Editor/、Setup/、DeviceSelector/、Wiki/）

---

## 七、引用关系

- 主规范「十八、EAPO 对齐度与差异化」（18.1-18.4）——对齐度/差异化正式落点（**需按 S1-S5 修订**）
- roadmap P0-6（子 APO 委托实现）——开放决策 ①②③ 收敛依据
- 本文件第五节（S1-S5）——规范修订触发项，随版本递增落地