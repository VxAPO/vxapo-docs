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
| `DeviceAPOInfo.cpp` | 249/396-413（安装模式探测：LfxGfx/SfxMfx/SfxEfx 三档） | install 5.1 |
| `DeviceAPOInfo.cpp` | 498-645（install 三模式互斥写 + per-device 备份） | install 5.5.2 / object 7.2 |
| `DeviceAPOInfo.cpp` | 500-576（槽位备份 + childApoPath） | object 7.2 |
| `DeviceAPOInfo.cpp` | 777-815（testAPOInstallation） | install 5.5.2（E3.4） |
| `filters/GraphicEQFilter.cpp` | 44-101（最小相位 FIR 卷积生成） | pipeline 4.18 |
| `helpers/GainIterator.cpp` | 30-98（对数频率线性插值） | pipeline 4.18 |
| `wdma_usb.inf` | `USBAudio.SysFx.Render`（CAPX MSFX 模板） | install 5.5.2（v9.0） |
| `Setup/Setup.nsi` | 35（`RequestExecutionLevel admin` 安装器提权） | CLI 引用规范 六 |
| `DeviceSelector/DeviceSelector.vcxproj` | 169-248（6 处 `<UACExecutionLevel>RequireAdministrator</UACExecutionLevel>`） | CLI 引用规范 六 |
| `helpers/TaskSchedulerHelper.cpp` | 34-186（登录计划任务跑 UpdateChecker，非提权） | CLI 引用规范（更新检查预留） |
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
| C11 | **BUFFER_SILENT 输入清零**：输入为 SILENT 时先 `memset` 输入缓冲（468-470）→ childRT 处理 → silent 输出判定（allowSilentBufferModification 时 486-499 / 否则输出清零 + BUFFER_SILENT 503-505） | EqualizerAPO.cpp 468-511 | **对齐 + v9.15 严格化**：VxAPO 对 SILENT 输入**不读残留、按全零处理、输出强制 SILENT**（pipeline 4.2 / object 7.1.11）——修复“引擎复用脏静音缓冲 → 自我反馈爆音” |
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

### 2.8 安装提权机制（EAPO 安装会话）

> **结论：EAPO 全程用「编译期 manifest 声明式提权」，无任何「运行时检测→提权重启」代码。**

| # | 行为（源码确认） | 源码位置 | VxAPO 借鉴 |
|---|------------------|----------|------------|
| C38 | **NSIS 安装器整体提权**：`RequestExecutionLevel admin`——安装器启动即触发 UAC，整流程管理员令牌（regsvr32 + DeviceSelector.exe /i 静默安装 + 写 HKLM） | Setup.nsi 35 | VxAPO CLI 安装器可对齐（installer 需管理员） |
| C39 | **Configurator/DeviceSelector manifest `RequireAdministrator`**：**6 个构建配置全部** `<UACExecutionLevel>RequireAdministrator</UACExecutionLevel>`——**每次启动**即要求管理员，写 FxProperties 无需运行时提权 | DeviceSelector.vcxproj 169/184/200/216/233/248 | **VxAPO CLI 关键借鉴**：写 HKLM（install/uninstall）需管理员，应 manifest 声明或显式提权 |
| C40 | **登录计划任务（非提权）**：`scheduleAtLogon` 注册「登录触发 + 网络可用」任务跑 **UpdateChecker**（`TASK_LOGON_INTERACTIVE_TOKEN`），`singleInstance` 用于更新检查——与提权无关 | TaskSchedulerHelper.cpp 34-186 | VxAPO 若做更新检查可复用计划任务模式 |

**补充（C39 佐证）**：程序内无 `ShellExecuteEx(runas)` / `IsUserAnAdmin()` / `CreateProcess` 提权调用（源码搜索确认），**全部依赖 manifest**。
程序外：安装器 `RequestExecutionLevel admin`（Setup.nsi 35）+ Configurator `RequireAdministrator`（DeviceSelector.vcxproj 6 处）。

### 2.9 安装槽位机制（模式探测 + 互斥写）

> **行为流**：`load()` 探测安装模式（Win8.1+ 三档）→ `install()` 按选定模式互斥写目标槽位 + 删本模式外槽位 + per-device 备份。
> **会话边界**：DeviceSelector.exe 启动时 UAC → 载入 `loadAllInfos` → 用户对每台设备 `install()`（管理员令牌，无运行时提权）。

| # | 行为（源码确认） | 源码位置 | VxAPO 对齐 |
|---|------------------|----------|------------|
| C41 | **LfxGfx 模式判定**：Win8.1+ 且 FxProperties **只有 LFX/GFX 值、无 SFX/MFX/EFX 值** → 驱动仅支持 Legacy | DeviceAPOInfo.cpp 396-408 | **对齐**：install 5.1 模式检测（Legacy 独占） |
| C42 | **SfxMfx 模式判定**：检测到 `combinedDeviceValueName`（**Win11 蓝牙组合设备，EFX 无效**）——值名精确为 `{b3f8fa53-0004-438e-9003-51a46e139bfc},41`（**PKEY_Device_ContainerId，设备容器 ID**，WT_DEVICE 属性集 PID 41），位于端点 `Properties` 子键（`keyPath\Properties`，非 FxProperties） | DeviceAPOInfo.cpp 51/410-411 | **对齐**：install 5.1 模式检测（蓝牙组合）——**执行端 `detect_install_mode` 输入：端点 Properties 子键下值名 `{b3f8fa53-...},41` 存在性** |
| C43 | **SfxEfx 默认模式**：否则（现代驱动默认） | DeviceAPOInfo.cpp 412-413 | **对齐**：SfxEfx 为默认（install 5.1） |
| C44 | **旧 Windows（<8.1）默认 LfxGfx**：`installMode` 初始即 `INSTALL_LFX_GFX`（249）；8.1+ 才探测 | DeviceAPOInfo.cpp 249/396 | **对齐**：install 5.1 模式默认 |
| C45 | **三模式互斥写**：`install()` 按 mode 写目标槽位 + **删除本模式外槽位**（LfxGfx 删 SFX/MFX/EFX；SfxMfx 删 LFX/GFX 保留 EFX；SfxEfx 删 LFX/GFX 保留 MFX） | DeviceAPOInfo.cpp 578-640 | **对齐**：install 5.5.2 按 mode 写槽位 |
| C46 | **处理模式值按槽位写**：新装槽位写 `{sfx/mfx/efx}ProcessingModesValueName`（已存在不覆盖），值 = `defaultProcessingModeValue` | DeviceAPOInfo.cpp 603-605/611-613/627-629 | **对齐**：install 5.5.2 Step 6（默认处理模式） |
| C47 | **FxProperties 备份先于写入**：存在 → 逐个值（NOKEY/NOVALUE 占位或实际值）写 childApoPath + `.reg` 备份；不存在 → 创建 + 全 5 槽位 NOKEY 占位 | DeviceAPOInfo.cpp 512-554 | **对齐**：object 7.2 备份全部槽位 |
| C48 | **采集设备只装 PreMix**：`installPostMix && !input`（PostMix 仅渲染设备） | DeviceAPOInfo.cpp 583/607/632 | **对齐**：object 7.2 installPostMix=!input |
| C49 | **已安装设备保留模式**：`load()` 检测 `foundAt`（LFX/SFX→premix、GFX/MFX/EFX→postmix）→ 保留现有安装状态 | DeviceAPOInfo.cpp 339-352 | **对齐**：install 5.1 已装保留 |
| C50 | **版本升级**：`canBeUpgraded()` = `installed && version != installVersion`（"2"） | DeviceAPOInfo.cpp 421-423 | **对齐**：install 5.4 can_be_upgraded |

### 2.10 GraphicEQ 与 CAPX「设备默认效果」（2026-08-10 源码确认）

| # | 行为（源码确认） | 源码位置 | VxAPO 对齐 |
|---|------------------|----------|------------|
| C51 | **EAPO 的 GraphicEQ 不是 biquad 级联**：`GraphicEQFilter` 继承 `ConvolutionFilter`，在 `initializeFilters` 中把节点增益按频率插值后生成频响，用 FFT 做最小相位变换，得到 FIR 再卷积 | GraphicEQFilter.cpp 44-101 | **VxAPO v9.0 + v9.7 对齐**：`pipeline/dsp/graphic_eq.rs` 对数频率插值 + 最小相位 FIR + **1024 点直接时域卷积（AVX2/FMA 向量化）**（v9.5 分块 FFT 块缓冲流停止丢尾音 → 切换“嗡”声，v9.6/v9.7 定稿直接 FIR） |
| C52 | **节点间对数频率线性插值**：`GainIterator::gainAt` 在 `log(freq)` 上线性插值；低于首节点/高于末节点取端点增益（频带外平坦） | GainIterator.cpp 30-98 | **对齐**：`gain_at()` 同语义 |
| C53 | **GetLatency 无 child 恒返回 0**：即使内部使用卷积（有滤波器固有延迟）也不向引擎上报；VxAPO 采用 1024 点直接 FIR（v9.7，无块缓冲），继续对齐该行为 | EqualizerAPO.cpp 82-95 | **对齐**：object 7.2 / 2026-08-10 延迟策略 |
| C54 | **EAPO 不处理 CAPX `MSFX\N` 模板**：通用 USB 设备由 `wdma_usb.inf` 在设备接口注册「Microsoft Audio Home Theater Effects」（WMALFXGFX 两个 APO）；Windows 重启/重新枚举端点可能从模板恢复微软 APO，EAPO 不接管 | wdma_usb.inf `USBAudio.SysFx.Render` | **VxAPO v9.0 + v9.4 扩展**：install `device/sysfx.rs` 定位并替换 `MSFX\N` 的 StreamEffect/ModeEffect（卸载时恢复）；v9.4 起 DLL `Initialize` 时**运行期自愈**——设备重新枚举后被 Windows 灌回的微软 CAPX 由本 DLL 自动再接管（仅动微软 CLSID、幂等、失败降级） |
| C55 | **强制启用增强**：EAPO 安装时删除 `{1da5d803-d492-4edd-8c23-e0c0ffee7f0e},5`（PKEY_AudioEndpoint_Disable_SysFx）；`fxTitle` 仅在新建 FxProperties 时写入 | DeviceAPOInfo.cpp 642-645 / 527 | **对齐 + 扩展**：VxAPO 同步删除该值；fxTitle 不写（避免历史音量/格式问题） |
| C56 | **v9.11 起 VxAPO 不再对齐 EAPO EQ 体系**：`GraphicEQ:` 命令与 `graphic_eq.rs` 移除，由 TOML `[[effects]] type="peq"` 混合式 PEQ 取代（200 Hz 分频：Fc<200 段 IIR 级联、Fc≥200 段采样率自适应最小相位 FIR，1024–8192 抽头）；命令解析/工厂注册不再 EAPO 对齐；延迟策略（v9.12 定稿）——`GetLatency` 恒 0、`latency_frames_atomic` 恒 0（隐藏延迟不上报；v9.11 曾尝试激活引擎帧数补偿，实测热重载/切歌播放卡住，回退） | —（独立设计） | **VxAPO v9.11 + v9.12**：`config TOML 设计文档.md` + `PEQ 设计文档.md` + `pipeline 4.10/4.18/4.22` + `config 6.0` |
| C57 | **热重载监控无跨实例去重（v9.15 回归对齐）**：每实例独立通知线程 + 10ms 去重 + loadSemaphore 阻塞式重载（FilterEngine.cpp 556-624）；**无进程级全局锁、无 mtime 去重表** | FilterEngine.cpp 556-624 | **VxAPO v9.15 对齐**：移除 v9.4 (mtime,size) 预检与实验性全局锁，恢复“每实例独立重载 + spec 指纹短路 + transition/pending”（object 7.1.9/7.1.18）；保留 diag 写锁防日志花屏 |

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
| 热重载监控（现状） | **v9.15 回归对齐**：每实例独立通知线程 + 10ms 去重 + spec 指纹短路；无跨实例去重/全局锁（移除 mtime 预检） | C27、C32、C57 |
| 注册表监视 | **VxAPO 不需要**（无 readReg 命令） | C33 |
| 安装自检 | **E3.4 描述需修订**（实际是 IAudioClient 管线自检） | C37 |
| 安装提权（manifest 声明式） | **VxAPO CLI 借鉴**：安装器 `RequestExecutionLevel admin` + 程序 `RequireAdministrator`，无运行时提权代码 | C38-C40 |
| 安装槽位模式探测 + 互斥写 | 对齐（LfxGfx 独占/SfxMfx 蓝牙/SfxEfx 默认三档 + 按 mode 互斥写+删槽位） | C41-C50 |
| GraphicEQ 实现 | **v9.0 + v9.7 对齐**：对数频率插值 + 最小相位 FIR + 1024 点直接时域卷积（AVX2/FMA 向量化，非 biquad 级联） | C51-C52 |
| 混合式 PEQ（现状） | **v9.11+**：`graphic_eq.rs` 移除，TOML `peq` 混合架构（200 Hz IIR + 最小相位 FIR）；跨频点宽 Q 段进 IIR；静音恢复淡入；端点重协商复用链 | C56 |
| CAPX「设备默认效果」 | **EAPO 不处理**；VxAPO v9.0 扩展：接管 `MSFX\N` 模板，替换微软 APO | C54 |
| 延迟上报 | 对齐：GetLatency 无 child 返回 0；VxAPO 直接 FIR 无块延迟不上报 | C53 |

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

- 滤波器 DSP 内部的逐样本处理细节（BiQuad/IIR 等 filters/ 目录；**GraphicEQ 已确认，见 2.10**）
- VSTPlugin 加载路径（helpers/VSTPluginInstance.cpp）
- 默认设备判定边界的完整流程（E3.1 已确认 driver 枚举 + GetDefaultAudioEndpoint 边界，但 Configurator 交互未读）
- 其他文件（Editor/、Setup/、DeviceSelector/、Wiki/）

---

## 七、引用关系

- 主规范「十八、EAPO 对齐度与差异化」（18.1-18.4）——对齐度/差异化正式落点（**需按 S1-S5 修订**）
- roadmap P0-6（子 APO 委托实现）——开放决策 ①②③ 收敛依据
- 本文件第五节（S1-S5）——规范修订触发项，随版本递增落地
