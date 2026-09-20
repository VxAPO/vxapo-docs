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
| `compressor` | `threshold_db`(-60..0)、`ratio`(1..20)、`knee_db`(0..12)、`attack_ms`(0.1..100)、`release_ms`(10..1000)、`makeup_gain_db`(0..24)、`wet`/`dry`(0..1) | 见下表 |
| `wide` | `gain`/`air`/`air_side`/`mix`(0..1)、`crossover_hz`(200..1000) | 见下表 |
| `loudness` | `phon`(0..120，必填)、`reference_phon`(0..120) | 见下表 |

各效果器缺省值（`pipeline/dsp/*.rs` 的 `Default`）：

| type | 缺省值 |
|------|--------|
| `aural` | `tune_hz 1760` / `drive 1.76993` / `odd 1.5` / `even 0.25` / `wet 0.5` / `dry 0.5` |
| `reverb` | `room_size 1.0` / `decay 0.41` / `damping 0.408290` / `bandwidth 0.350110` / `density 1.0` / `lat5 0.70` / `lat6 0.50` / `pre_delay_ms 0` / `motion_rate 0.110871` / `motion_depth 0.63` / `low_cut_hz 100` / `wet 0.27` / `dry 0.73` |
| `compressor` | `threshold_db -18` / `ratio 4` / `knee_db 3` / `attack_ms 10` / `release_ms 100` / `makeup_gain_db 6` / `wet 1.0` / `dry 0.0` |
| `wide` | `gain 0` / `air 0.354331` / `air_side 0` / `mix 0.6` / `crossover_hz 200` |
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
- `compressor`：全声道联动 RMS 检测 + 含软膝静态曲线 + dB 域 attack/release 平滑 + makeup 增益；
  取代原 `maximizer` / `leveler`。
- `wide`：线性相位 FIR 分频后只处理高频支路；`gain` 控制侧通道增强量，`air` 为中置空气吸收，
  `air_side` 为侧通道空气吸收（默认 0 = 关闭），`mix` 为处理增量干湿比。
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

`Equalizer 行为文档` 的价值在于为安装/子 APO/格式协商等行为提供对照。当前实现已不再兼容 EAPO 的 `config.txt` 命令语法，但在以下方面保留了可对照的机制：

- 子 APO 创建与委托。
- 安装模式探测（LfxGfx / SfxMfx / SfxEfx）。
- 槽位备份与恢复。
- 安装后自检（`CoCreateInstance` 验证）。

详细源码位置可参考原文档，但本文档以现有代码为准。

> 更详细的 `config/` 与 `pipeline/` 模块规范见 `driver/zh/模块引用规范/`。
