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
- `enabled = false` 可旁路单个效果器。

### 1.3 效果器类型

| type | 主要参数 |
|------|----------|
| `peq` | `crossover_hz`、`bands`（fc/gain_db/q/type） |
| `preamp` | `gain_db` |
| `aural` | `tune_hz/drive/odd/even/wet/dry` |
| `reverb` | `room_size/decay/damping/bandwidth/density/lat/pre_delay/motion/wet/dry` |
| `maximizer` | `gain_boost_db/max_output_db/release_ms/target/lookahead_ms/dither/wet/dry` |
| `wide` | `intensity` |
| `loudness` | `phon/reference_phon` |

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

## 5. EqualizerAPO 行为参考

`Equalizer 行为文档` 的价值在于为安装/子 APO/格式协商等行为提供对照。当前实现已不再兼容 EAPO 的 `config.txt` 命令语法，但在以下方面保留了可对照的机制：

- 子 APO 创建与委托。
- 安装模式探测（LfxGfx / SfxMfx / SfxEfx）。
- 槽位备份与恢复。
- 安装后自检（`CoCreateInstance` 验证）。

详细源码位置可参考原文档，但本文档以现有代码为准。

> 更详细的 `config/` 与 `pipeline/` 模块规范见 `driver/zh/模块引用规范/`。
