# VxAPO Driver 配置与 DSP 设计

## 1. config.toml 模型

每个设备一份配置文件：

```text
C:\ProgramData\VxAPO\{GUID}\config.toml
```

配置为 TOML v1，顶层包含：

```toml
version = 1
enabled = true             # false = 整链 passthrough，内容保留

[meta]
app = "vxapo"
schema = 1

[[effects]]
type = "peq"
enabled = true
crossover_hz = 200
name = "示例"              # APP 元数据，driver 忽略
group = "示例组"           # APP 元数据，driver 忽略
[[effects.bands]]
fc = 1000
gain_db = 3.0
q = 1.0
```

### 1.1 模型分层

- `FileModel`（`config/model.rs`）：TOML 文件格式，包含 APP 元数据。
- `ChainModel`（`pipeline/dsp/model.rs`）：DSP 语义模型，无 APP 元数据。
- 转换在 `config` 层完成，依赖方向为 `config -> pipeline/dsp`。

### 1.2 校验规则

- 未知 `type`、未知键、缺必填键、超范围、类型错误都会导致整文件解析失败。
- 解析失败时保留旧链。
- PEQ 段数：单块 1-31，跨块全局（按声道/共享）合计不超过 31。
  `channels` 明确指定声道的块计入对应声道预算，未指定声道的块计入共享预算。
- PEQ 单段取值范围：`fc 20..20000`、`gain_db -30..+30`、`q 0.1..12`、
  `crossover_hz 20..20000`（缺省 200）。
- 通用增益范围 `[-120, +48]` dB 只适用于 `preamp.gain_db`；各效果器参数按 §1.3 各自的范围校验。
- 旧配置兼容映射：`type = "maximizer"` / `"leveler"` 解析为 `compressor`，
  其旧参数键（`gain_boost_db` / `max_output_db` / `target` / `lookahead_ms` / `dither` /
  `target_rms_db` / `response_s` / `max_gain_db` / `dynamic_preserve` / `noise_gate_db` /
  `peak_limit_db`）被接受但忽略，统一走 `CompressorParams` 默认值。
- `wide` 的旧键 `intensity` / `depth` 依次回退映射到 `air`（仅在该段未显式写 `air` 时生效）。
- `enabled = false` 可旁路单个效果器。

### 1.3 效果器类型

| type | 参数（范围） | 缺省值 |
|------|--------------|--------|
| `peq` | `crossover_hz`（20..20000）、`bands`（fc/gain_db/q/type） | `200` / 必填 |
| `preamp` | `gain_db`（-120..48） | 必填 |
| `aural` | `tune_hz`(500..10000)、`drive`(0..4.25)、`odd`(0..1.5)、`even`(0..0.75)、`wet`/`dry`(0..1) | 见下表 |
| `reverb` | `room_size`(0.5..1.5)、`decay`/`damping`/`bandwidth`/`density`/`lat5`/`lat6`(0..1)、`pre_delay_ms`(0..100)、`motion_rate`(0.05..2)、`motion_depth`(0..2，旧键 `motion_depth_ms`)、`low_cut_hz`(20..250)、`wet`/`dry`(0..1) | 见下表 |
| `compressor` | `threshold_db`(-48..0)、`ratio`(1..20)、`lift`(0..1)、`attack_ms`(1..100)、`mix`(0..1) | 见下表 |
| `wide` | `gain`/`air`/`side_itd`(0..1)、`crossover_hz`(200..1000) | 见下表 |
| `loudness` | `phon`(0..120，必填)、`reference_phon`(0..120) | 见下表 |

各效果器缺省值（`pipeline/dsp/*.rs` 的 `Default`）：

| type | 缺省值 |
|------|--------|
| `aural` | `tune_hz 1760` / `drive 1.76993` / `odd 1` / `even 0.2` / `wet 0.5` / `dry 0.5`（odd/even 原 1.5/0.25，odd 即量程上限、谐波过量已收紧） |
| `reverb` | `room_size 1.0` / `decay 0.41` / `damping 0.408290` / `bandwidth 0.350110` / `density 1.0` / `lat5 0.70` / `lat6 0.50` / `pre_delay_ms 0` / `motion_rate 0.110871` / `motion_depth 0.63` / `low_cut_hz 100` / `wet 0.27` / `dry 0.73` |
| `compressor` | `threshold_db -12` / `ratio 3` / `lift 0` / `attack_ms 10` / `mix 1.0` |
| `wide` | `gain 0.05` / `air 0.2` / `side_itd 0.2` / `crossover_hz 200` |
| `loudness` | `reference_phon 80`（`phon` 必填） |

PEQ 段类型（`bands[].type`，缺省 `peaking`）：

| type | 语义 |
|------|------|
| `peaking` | 峰值滤波（RBJ peaking EQ） |
| `low_shelf` / `high_shelf` | 低/高架滤波，`gain_db` 为目标电平 |
| `low_pass` / `high_pass` | 低/高通滤波，RBJ 公式本身无增益项：`gain_db` 线性乘到**通带**上，`0 dB` 时保持纯滤波行为 |

效果器实现要点：

- `reverb`：Dattorro 板式混响；`low_cut_hz` 为低频瞬态保护分频点，分频点以下逐声道旁路混响、
  原样直通（`20` ≈ 关闭），分频点以上进混响。
- `compressor`：全声道联动**两级检测**。主检测为**慢轴响度**（mean-square 非对称极点：
  上升 0.5s 保音符尺度平移、回落 0.2s 保段落结束后恢复不拖沓（对称 0.5s 时深压
  恢复要 1.5s+，是「慢吞吞闷感」来源）；+ 10dB crest 补偿，对齐原峰值口径、
  threshold 旋钮触发点基本不变；材料 crest 偏离典型值时漂移 ±几 dB，节奏型素材
  因非对称偏置读数偏低 ~2dB）：增益在音符/瞬态尺度近似恒定，峰与峰身**等比
  平移**、crest 原样保留——旧的峰值包络直驱会让 attack 漏过峰尖、峰身被全额压掉
  （同击峰/身偏移差可达 9dB，crest 被吹大），即「强行压下去、不自然」的来源。
  **峰值包络**（瞬时 attack + 深度加权速率释放，深压 30dB/s、浅压 180dB/s）降级为
  **lift 封顶专用**：抬升量 = `u × min(max(阈值-峰值包络, 0), 24dB)`，恒不过阈值，
  且电平突增的瞬态瞬间撤销抬升（绝不对瞬时大信号做提升）。曲线为固定 3dB 软膝
  静态曲线（**判定按慢轴读数**：阈值右侧按 `ratio` 压大、左侧按 `lift` 朝阈值抬升，
  膝区 Hermite C1 连续、`lift=0` 逐位退化为原插值）+ dB 域 attack/release 平滑
  （**释放固定自动、无参数**：双极点 100ms/600ms，慢极点权重随压制深度**立方**
  退到 0——深压保持 0.3/0.7 混合（600ms 尾巴，持续段与低频不泵浦），浅压纯走
  快极点）+
  **有界输出**（无 makeup：lift=0 恒 `|out| ≤ |in|`；lift>0 时抬升量按峰值包络
  封顶、输出电平恒 ≤ 阈值，两种情况都不可能削波；干湿由 `mix` 等比混合，
  `dry = 1 - mix`）；
  取代原 `maximizer` / `leveler`，旧 `knee_db` / `makeup_gain_db` / `wet` / `dry` 键静默忽略；
  `release_ms` 已随手动释放一并移除（未知键，出现即拒载）。
- `wide`：线性相位 FIR 分频；`gain`（高频补偿）只决定**低频降低**的深度——RBJ 低架，
  Q 固定 0.707、拐点 = `crossover_hz`、深度 = 6dB × `gain`，且加在**重建后的干声和**上
  （只在低通支路做会破坏分频两路对称，让线性相位的前后振铃露出来）；`air` 为中置空气吸收；
  `side_itd`（侧向去相关）控制 1.5kHz 以上高通段的**分数延迟长度**（纯延迟，不改幅度）：
  0 = 不动相位，1 = 最大时间差 0.10 / 0.15 ms（Δτ = 0.05 ms，按采样率折算，换采样率时间量恒定）。
  侧通道不提升、也不单独衰减；干路取该分离器的低通段，中置参照与干声各平移其群延迟
  （48k 下 127 采样），因此链路总延迟比主分频多 D2 ≈ 2.6 ms；处理增量过截止 = 分频点的
  一阶高通，输出端软膝限幅兜底。
- `aural`：二阶 Butterworth 高通 + 电平跟随 + tanh 奇次软饱和 + 半波整流偶次，Wet/Dry 混合。

## 2. 配置指纹与热重载

- 指纹由 `EffectConfig::spec()` 生成，只包含 DSP 字段。
- `name`、`group`、`meta` 不参与指纹。
- watcher 监控 `C:\ProgramData\VxAPO\{GUID}` 目录，检测到变化后触发 `hot_reload`。
- 热重载在锁外解析新配置，锁内替换链指针，并通过 10ms 升余弦过渡避免爆音。

## 3. PEQ 混合架构

PEQ 使用 200 Hz 分频的混合架构：

- `Fc < 200 Hz`：IIR biquad 路径。
- `Fc >= 200 Hz`：FIR 路径。
- 若高频段在分频点处偏差超过 0.25 dB，也归入 IIR 以保证低频精度。
- 段类型：`peaking` / `low_shelf` / `high_shelf` / `low_pass` / `high_pass`（缺省 peaking）。
  `low_pass` / `high_pass` 的 `gain_db` 线性乘到通带（0 dB 时与纯滤波完全一致），
  因此可用同一段同时表达"截止 + 通带电平"。

处理链：

```text
输入 -> IIR 级联 -> 最小相位 FIR -> 输出
```

FIR 抽头数按采样率自适应：

```text
N = next_pow2(round(sr * 0.0213))，限制在 [1024, 8192]
```

- `N <= 2048`：直接时域 FIR。
- `N > 2048`：分块 FFT，块大小 128。
- 延迟不上报：`latency_frames_atomic` 恒 0。

## 4. DSP 数值安全

### 4.1 问题背景

极端参数（如 ±1000 dB）会导致：

- `db_to_linear` 溢出为 `inf`。
- biquad 极点贴近单位圆，产生持续振荡。
- 增益跳变产生爆音。

### 4.2 三层防线

1. 解析层：将参数 clamp 到安全范围。
2. 系数层：biquad 使用 f64 计算，做有限性与稳定性校验。
3. RT 层：设置 FTZ/DAZ，冲刷次正规数，并对 NaN/Inf 兜底。

### 4.3 关键常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `GAIN_DB_MIN` | -120.0 | 最小增益 |
| `GAIN_DB_MAX` | 48.0 | 最大增益 |
| `FILTER_CUT_FLOOR_DB` | -60.0 | 滤波深切地板 |
| `Q_MIN` / `Q_MAX` | 0.05 / 18.0 | Q 值安全范围 |
| `MAX_GAIN_STEP_RATIO` | 0.05 | 增益平滑步长 |
| `MAX_PEQ_BANDS` | 31 | PEQ 段数上限 |
| `crossover_hz` | 20.0 / 20_000.0 | PEQ 分频点范围（缺省 200） |
| PEQ `fc` | 20.0 / 20_000.0 | 单段中心/截止频率范围 |
| PEQ `gain_db` | -30.0 / 30.0 | 单段增益范围 |
| PEQ `q` | 0.1 / 12.0 | 单段 Q 范围 |

## 5. EqualizerAPO 行为参考

本节为安装/子 APO/格式协商等行为提供与 EAPO 的对照基准（原 `Equalizer 行为文档` 自 v9.11 起退役、v9.15 后随文档重组移除，本节是唯一保留的对照来源）。当前实现已不再兼容 EAPO 的 `config.txt` 命令语法，但在以下方面保留了可对照的机制：

- 子 APO 创建与委托。
- 安装模式探测（LfxGfx / SfxMfx / SfxEfx）。
- 槽位备份与恢复。
- 安装后自检（`CoCreateInstance` 验证）。

详细源码位置可参考原文档，但本文档以现有代码为准。

以下条件清单已对照 EAPO 源码逐行核实（2026-08-03 实读 `D:\Source_Code\equalizerapo-code`：EqualizerAPO.cpp / DeviceAPOInfo.cpp / FilterEngine.cpp/.h / FilterConfiguration.cpp / parser/RegistryFunctions.cpp / IFilter.h）；已确认行为均标注源码位置，并与推断严格区分。

### 5.1 子 APO 生命周期

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

### 5.2 APOProcess 委托与时序

| # | 行为（源码确认） | 源码位置 | VxAPO 对齐 |
|---|------------------|----------|------------|
| C10 | **APOProcess 委托时序**：**`childRT->APOProcess` 先跑**（作用于输入缓冲）→ `engine.process(outputFrames, outputFrames, ...)`（**输出缓冲就地处理**） | EqualizerAPO.cpp 472-477 | **完全对齐**：object 7.1.11 child 前置每帧一次 |
| C11 | **BUFFER_SILENT 输入清零**：输入为 SILENT 时先 `memset` 输入缓冲（468-470）→ childRT 处理 → silent 输出判定（allowSilentBufferModification 时 486-499 / 否则输出清零 + BUFFER_SILENT 503-505） | EqualizerAPO.cpp 468-511 | **对齐 + v9.15 严格化**：VxAPO 对 SILENT 输入**不读残留、按全零处理、输出强制 SILENT**（pipeline 4.2 / object 7.1.11）——修复“引擎复用脏静音缓冲 → 自我反馈爆音” |
| C12 | **child 输出通道语义**：`realChannelCount` = 有 child 时 **outFormat 通道数**；无 child 时 **inFormat 通道数** | EqualizerAPO.cpp 365-369 | **等价立场**：VxAPO 不引入 realChannelCount 机制（主规范 18.2 D2） |
| C13 | **realChannelCount 含 child 输出**：字段注释「处理时输入通道数（含 child APO 输出）」 | FilterEngine.h 131-132 | **等价**：主规范 18.2 D2/D3 |

### 5.3 槽位安装与备份

| # | 行为（源码确认） | 源码位置 | VxAPO 对齐 |
|---|------------------|----------|------------|
| C14 | **installPostMix 默认 = !input**（渲染设备才装 PostMix） | DeviceAPOInfo.cpp 254 | **对齐**：object 7.2 |
| C15 | **useOriginal 默认**：`useOriginalAPOPreMix = true`、`useOriginalAPOPostMix = !input` | DeviceAPOInfo.cpp 255-256 | **对齐**：object 7.2 前任 APO 备份 |
| C16 | **备份全部槽位**：FxProperties 各槽位（allGuidValueNames）现值 → 写 `childApoPath\{deviceGuid}`（567-577 同区）——**含 NOKEY/NOVALUE 占位** | DeviceAPOInfo.cpp 529-548 | **对齐**：object 7.2 备份全部槽位供回退 |
| C17 | **原 APO 备份 `.reg` 文件**：FxProperties 存在时 `saveToFile` 写 `backup_*.reg`（551-553） | DeviceAPOInfo.cpp 550-553 | **VxAPO 差异**：VxAPO 无 `.reg` 文件备份（object 7.2 仅注册表备份）——留 P1 按需 |
| C18 | **autoAdjust 写入**：`autoAdjust=true` → 删 `disableAutoAdjust` 键；**false → 写 `disableAutoAdjust="true"`**（默认） | DeviceAPOInfo.cpp 567-575 | **对齐 + 保守默认**：VxAPO 默认 false（install 5.5.2 E3.3） |
| C19 | **allowSilentBuffer 写入**：写 `allowSilentBufferValueName`（true/false） | DeviceAPOInfo.cpp 565-566 | **对齐**：object 7.1.2 allow_silent_buffer_modification |

### 5.4 格式协商与通道语义

| # | 行为（源码确认） | 源码位置 | VxAPO 对齐 |
|---|------------------|----------|------------|
| C20 | **下混拒绝**：`inFormat > 2ch && inFormat.dwSamplesPerFrame > outFormat.dwSamplesPerFrame` → 创建 outFormat 媒体类型 + `S_FALSE`（**父不执行下混**） | EqualizerAPO.cpp 277-282 | **等价立场（修正表述）**：VxAPO 协商期锁死「收到的输入==输出通道数」（父不转换）；**「拒 in>out」即下混拒绝，非「锁死 in==out」** |
| C21 | **mono→stereo 上混特例**：`realChannelCount==1 && outputChannelCount>=2` → 复制 mono 到声道 1（注释「模拟无 APO 时 Windows 音频系统自动上混」） | FilterConfiguration.cpp 126-128 | **等价立场**：仅此上混特例，其他上混/下降混父均不执行（主规范 18.2） |
| C22 | **协商委托 + child 失败销毁回退**：child 成功用 child 判定；child 失败 **resetChild 销毁** + 回落自身 | EqualizerAPO.cpp 258-283 | **对齐 + 收窄**：VxAPO 白名单三段式（主规范 18.2 D5） |

### 5.5 配置解析行为

| # | 行为（源码确认） | 源码位置 | VxAPO 对齐 |
|---|------------------|----------|------------|
| C23 | **无冒号行跳过**：`pos = line.find(':')`；**`pos==-1` → 整行跳过不解析**（无错误、不拒绝） | FilterEngine.cpp 329-330 | **对齐（修正）**：VxAPO v7.11「无冒号行拒绝」**比 EAPO 更严格**——EAPO 静默跳过，VxAPO 显式报错（intent「语法严格性」有意差异） |
| C24 | **读文件失败返回**：`CreateFile` 失败（非共享冲突）→ `return`（保留当前配置不替换） | FilterEngine.cpp 285-287 | **对齐**：object 7.1.18 解析失败保留旧链 |
| C25 | **配置文件等待语义**：写文件过程中 `ERROR_SHARING_VIOLATION` → `Sleep(1)` 重试（非报错） | FilterEngine.cpp 282-292 | **VxAPO 差异**：VxAPO 128KB 闸门 + spec 比对（config 6.1）；EAPO 无大小闸门 |

### 5.6 配置加载与过渡（FilterEngine / FilterConfiguration）

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

### 5.7 安装与设备（E3.x）

| # | 行为（源码确认） | 源码位置 | VxAPO 对齐 |
|---|------------------|----------|------------|
| C36 | **默认设备判定边界（E3.1）**：driver 层枚举（`loadAllInfos`）仅返回安装容器；默认设备由 `GetDefaultAudioEndpoint`（**用户层/API 层**）取得后与 driver 结果对应 | DeviceAPOInfo.cpp 82-87/106-116 | **对齐**：install 5.4 E3.1 |
| C37 | **testAPOInstallation（E3.4 修正）**：激活 `IAudioClient`（Activate + GetMixFormat + Initialize 共享模式 100ms）——**全音频管线自检**，**非**「CoCreateInstance 验证 DLL 可实例化」；失败 `fail()` 抛 `DeviceException`（异常上抛） | DeviceAPOInfo.cpp 777-815 | **VxAPO 需修订 E3.4 描述**：install 5.5.2「CoCreateInstance 验证 DLL」**与源码不符**——EAPO 实际激活 IAudioClient 做管线自检 |

### 5.8 安装提权机制（EAPO 安装会话）

> **结论：EAPO 全程用「编译期 manifest 声明式提权」，无任何「运行时检测→提权重启」代码。**

| # | 行为（源码确认） | 源码位置 | VxAPO 借鉴 |
|---|------------------|----------|------------|
| C38 | **NSIS 安装器整体提权**：`RequestExecutionLevel admin`——安装器启动即触发 UAC，整流程管理员令牌（regsvr32 + DeviceSelector.exe /i 静默安装 + 写 HKLM） | Setup.nsi 35 | VxAPO CLI 安装器可对齐（installer 需管理员） |
| C39 | **Configurator/DeviceSelector manifest `RequireAdministrator`**：**6 个构建配置全部** `<UACExecutionLevel>RequireAdministrator</UACExecutionLevel>`——**每次启动**即要求管理员，写 FxProperties 无需运行时提权 | DeviceSelector.vcxproj 169/184/200/216/233/248 | **VxAPO CLI 关键借鉴**：写 HKLM（install/uninstall）需管理员，应 manifest 声明或显式提权 |
| C40 | **登录计划任务（非提权）**：`scheduleAtLogon` 注册「登录触发 + 网络可用」任务跑 **UpdateChecker**（`TASK_LOGON_INTERACTIVE_TOKEN`），`singleInstance` 用于更新检查——与提权无关 | TaskSchedulerHelper.cpp 34-186 | VxAPO 若做更新检查可复用计划任务模式 |

**补充（C39 佐证）**：程序内无 `ShellExecuteEx(runas)` / `IsUserAnAdmin()` / `CreateProcess` 提权调用（源码搜索确认），**全部依赖 manifest**。
程序外：安装器 `RequestExecutionLevel admin`（Setup.nsi 35）+ Configurator `RequireAdministrator`（DeviceSelector.vcxproj 6 处）。

### 5.9 安装槽位机制（模式探测 + 互斥写）

> **行为流**：`load()` 探测安装模式（Win8.1+ 三档）→ `install()` 按选定模式互斥写目标槽位 + 删本模式外槽位 + per-device 备份。
> **会话边界**：DeviceSelector.exe 启动时 UAC → 载入 `loadAllInfos` → 用户对每台设备 `install()`（管理员令牌，无运行时提权）。

| # | 行为（源码确认） | 源码位置 | VxAPO 对齐 |
|---|------------------|----------|------------|
| C41 | **LfxGfx 模式判定**：Win8.1+ 且 FxProperties **只有 LFX/GFX 值、无 SFX/MFX/EFX 值** → 驱动仅支持 Legacy | DeviceAPOInfo.cpp 396-408 | **对齐**：install 5.1 模式检测（Legacy 独占） |
| C42 | **SfxMfx 模式判定**：检测到 `combinedDeviceValueName`（**Win11 蓝牙组合设备，EFX 无效**）——值名精确为 `{b3f8fa53-0004-438e-9003-51a46e139bfc},41`（**PKEY_Device_ContainerId，设备容器 ID**，WT_DEVICE 属性集 PID 41），位于端点 `Properties` 子键（`keyPath\Properties`，非 FxProperties） | DeviceAPOInfo.cpp 51/410-411 | **有意偏离（deliberate divergence）**：EAPO 以容器 ID 值名 `{b3f8fa53-...},41`（PKEY_Device_ContainerId）判定蓝牙组合设备——该值对所有设备都存在，会把 USB 设备误判为 SfxMfx；VxAPO 改以端点实例 ID 前缀 BTHENUM/BTHLE 判定（install/device/slots/read.rs 的 `detect_install_mode`，info.rs:251-285） |
| C43 | **SfxEfx 默认模式**：否则（现代驱动默认） | DeviceAPOInfo.cpp 412-413 | **对齐**：SfxEfx 为默认（install 5.1） |
| C44 | **旧 Windows（<8.1）默认 LfxGfx**：`installMode` 初始即 `INSTALL_LFX_GFX`（249）；8.1+ 才探测 | DeviceAPOInfo.cpp 249/396 | **对齐**：install 5.1 模式默认 |
| C45 | **三模式互斥写**：`install()` 按 mode 写目标槽位 + **删除本模式外槽位**（LfxGfx 删 SFX/MFX/EFX；SfxMfx 删 LFX/GFX 保留 EFX；SfxEfx 删 LFX/GFX 保留 MFX） | DeviceAPOInfo.cpp 578-640 | **对齐**：install 5.5.2 按 mode 写槽位 |
| C46 | **处理模式值按槽位写**：新装槽位写 `{sfx/mfx/efx}ProcessingModesValueName`（已存在不覆盖），值 = `defaultProcessingModeValue` | DeviceAPOInfo.cpp 603-605/611-613/627-629 | **对齐**：install 5.5.2 Step 6（默认处理模式） |
| C47 | **FxProperties 备份先于写入**：存在 → 逐个值（NOKEY/NOVALUE 占位或实际值）写 childApoPath + `.reg` 备份；不存在 → 创建 + 全 5 槽位 NOKEY 占位 | DeviceAPOInfo.cpp 512-554 | **对齐**：object 7.2 备份全部槽位 |
| C48 | **采集设备只装 PreMix**：`installPostMix && !input`（PostMix 仅渲染设备） | DeviceAPOInfo.cpp 583/607/632 | **对齐**：object 7.2 installPostMix=!input |
| C49 | **已安装设备保留模式**：`load()` 检测 `foundAt`（LFX/SFX→premix、GFX/MFX/EFX→postmix）→ 保留现有安装状态 | DeviceAPOInfo.cpp 339-352 | **对齐**：install 5.1 已装保留 |
| C50 | **版本升级**：`canBeUpgraded()` = `installed && version != installVersion`（"2"） | DeviceAPOInfo.cpp 421-423 | **对齐**：install 5.4 can_be_upgraded |

### 5.10 GraphicEQ 与 CAPX「设备默认效果」（2026-08-10 源码确认）

| # | 行为（源码确认） | 源码位置 | VxAPO 对齐 |
|---|------------------|----------|------------|
| C51 | **EAPO 的 GraphicEQ 不是 biquad 级联**：`GraphicEQFilter` 继承 `ConvolutionFilter`，在 `initializeFilters` 中把节点增益按频率插值后生成频响，用 FFT 做最小相位变换，得到 FIR 再卷积 | GraphicEQFilter.cpp 44-101 | **VxAPO v9.0 + v9.7 对齐**：`pipeline/dsp/graphic_eq.rs` 对数频率插值 + 最小相位 FIR + **1024 点直接时域卷积（AVX2/FMA 向量化）**（v9.5 分块 FFT 块缓冲流停止丢尾音 → 切换“嗡”声，v9.6/v9.7 定稿直接 FIR） |
| C52 | **节点间对数频率线性插值**：`GainIterator::gainAt` 在 `log(freq)` 上线性插值；低于首节点/高于末节点取端点增益（频带外平坦） | GainIterator.cpp 30-98 | **对齐**：`gain_at()` 同语义 |
| C53 | **GetLatency 无 child 恒返回 0**：即使内部使用卷积（有滤波器固有延迟）也不向引擎上报；VxAPO 采用 1024 点直接 FIR（v9.7，无块缓冲），继续对齐该行为 | EqualizerAPO.cpp 82-95 | **对齐**：object 7.2 / 2026-08-10 延迟策略 |
| C54 | **EAPO 不处理 CAPX `MSFX\N` 模板**：通用 USB 设备由 `wdma_usb.inf` 在设备接口注册「Microsoft Audio Home Theater Effects」（WMALFXGFX 两个 APO）；Windows 重启/重新枚举端点可能从模板恢复微软 APO，EAPO 不接管 | wdma_usb.inf `USBAudio.SysFx.Render` | **VxAPO v9.0 + v9.4 扩展**：install `device/sysfx.rs` 定位并替换 `MSFX\N` 的 StreamEffect/ModeEffect（卸载时恢复）；v9.4 起 DLL `Initialize` 时**运行期自愈**——设备重新枚举后被 Windows 灌回的微软 CAPX 由本 DLL 自动再接管（仅动微软 CLSID、幂等、失败降级） |
| C55 | **强制启用增强**：EAPO 安装时删除 `{1da5d803-d492-4edd-8c23-e0c0ffee7f0e},5`（PKEY_AudioEndpoint_Disable_SysFx）；`fxTitle` 仅在新建 FxProperties 时写入 | DeviceAPOInfo.cpp 642-645 / 527 | **对齐 + 扩展**：VxAPO 同步删除该值；fxTitle 不写（避免历史音量/格式问题） |
| C56 | **v9.11 起 VxAPO 不再对齐 EAPO EQ 体系**：`GraphicEQ:` 命令与 `graphic_eq.rs` 移除，由 TOML `[[effects]] type="peq"` 混合式 PEQ 取代（200 Hz 分频：Fc<200 段 IIR 级联、Fc≥200 段采样率自适应最小相位 FIR，1024–8192 抽头）；命令解析/工厂注册不再 EAPO 对齐；延迟策略（v9.12 定稿）——`GetLatency` 恒 0、`latency_frames_atomic` 恒 0（隐藏延迟不上报；v9.11 曾尝试激活引擎帧数补偿，实测热重载/切歌播放卡住，回退） | —（独立设计） | **VxAPO v9.11 + v9.12**：`config TOML 设计文档.md` + `PEQ 设计文档.md` + `pipeline 4.10/4.18/4.22` + `config 6.0` |
| C57 | **热重载监控无跨实例去重（v9.15 回归对齐）**：每实例独立通知线程 + 10ms 去重 + loadSemaphore 阻塞式重载（FilterEngine.cpp 556-624）；**无进程级全局锁、无 mtime 去重表** | FilterEngine.cpp 556-624 | **VxAPO v9.15 对齐**：移除 v9.4 (mtime,size) 预检与实验性全局锁，恢复“每实例独立重载 + spec 指纹短路 + transition/pending”（object 7.1.9/7.1.18）；保留 diag 写锁防日志花屏 |

> 更详细的 `config/` 与 `pipeline/` 模块规范见 `driver/zh/模块引用规范/`。
