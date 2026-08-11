# PEQ 设计文档（混合式架构，Spec-Drafting）

> 状态：已实现（v9.11，roadmap P1-6 完成）。核心内容已并入 `config 模块规范.md` /
> `pipeline 模块规范.md` / `Equalizer 行为文档.md` 对应章节。

## 1. 目标与范围

混合式现代图形化 PEQ：曲线型浮动段数（单块 1–31 段，跨块全局合计 ≤ 31）peaking，
每段可调 Fc / Gain / Q。

- **范围**：只针对 PEQ，不做空间/IR 卷积。`Convolution:` 类通用 IR 卷积能力
  不纳入本架构，也不为其做任何适配。
- **不再对齐 EAPO**：命令解析、工厂注册、IR 生成均不再考虑 EqualizerAPO 兼容；
  旧 `GraphicEQ:` 命令移除，由 `PEQ:` 取代。
- 保留与现有命令体系的一致性：`Key Value` 语法、单位可选、大小写不敏感、
  段间分号分隔。

## 2. 配置表达（TOML，现代语法）

```toml
[[effects]]
type = "peq"
crossover_hz = 200
name = "脚步声增强"            # 可选，APP 元数据，driver 忽略
group = "FPS 预设"             # 可选，APP 元数据，driver 忽略
[[effects.bands]]
fc = 1000
gain_db = 3.0
q = 1.0
[[effects.bands]]
fc = 2500
gain_db = -2.0
q = 2.0
```

- 配置格式全局迁移为 TOML（`config TOML 设计文档.md`）；`PEQ:` 文本命令
  不再存在，效果器统一为 `[[effects]]` 表 + `type = "peq"`；
- 段：`[[effects.bands]]` 数组表，每段恰好 `fc` / `gain_db` / `q` 三个键，
  顺序自由；单位固化（Hz / dB / 无量纲 Q），不再有可选单位解析；
- 范围与校验：
  - 单块段数 [1, 31]（少于 1 或多于 31 → 语法错误）；
  - 全局（所有 peq 块 band 合计）≤ 31（v9.16 新增）；
  - `Fc` [20, 20000] Hz，`Gain` [-30, +30] dB，`Q` [0.1, 12.0]；
  - 缺键、未知键、非法数值、同段重复键 → 整文件解析失败（保留旧链）；
- `enabled = false` 显式旁路 PEQ；无 `effects` 或 PEQ 缺席 = 无 EQ；
- 段数上限常量沿用 `MAX_GRAPHIC_EQ_BANDS = 31`，下限 `MIN_PEQ_BANDS = 1`
  （v9.16：UI 卡片模型允许 1 段卡 / 无组裸 band）；全局合计 ≤ 31 由 config 层校验。

## 3. 混合架构（200 Hz 分频）

### 3.1 段归属（边界判据，v0.1 定稿）

```text
Fc < 200 Hz               → IIR 路径
Fc ≥ 200 Hz               → FIR 路径（含 Fc = 200.0，分频点归属高频路径）
Fc > 200 Hz 且分频点处 |dB| > 0.25 → 也进 IIR（宽 Q 段低频泄漏由 biquad
                              解析精确实现，避免 FIR 低频分辨率不足）
```

- 判据用常量 `CROSSOVER_HZ = 200.0` 精确比较，无浮点容差；
- 补充判据（v9.12 修订）：Q 值低的 peaking 段没有“截止”在分频点，其影响
  范围会泄漏到低频——凡在分频点处频响偏离 > 0.25 dB 的段（如 fc=250/Q=0.6）
  由 IIR 主实现（biquad 低频解析精确），其余段（低频泄漏 < 0.25 dB）由 FIR
  主实现，误差无感；总响应仍由 T−L 全频段恒等保证衔接；
- 边界语义：分频点及以上的段由 FIR 统一承担相位特性；低于分频点的段由
  IIR 承担。无论归属哪边，整体幅度拟合都精确（见 3.3），归属只影响相位
  特征与约定清晰度。

### 3.2 处理链（级联）

每条通道：`输入 → IIR 级联（fc<200 段，按频率升序）→ 最小相位 FIR → 输出`。

- IIR：复用现有 `BiquadFilter` peaking 实现，每段一个二阶 biquad；
- FIR：由目标频响经最小相位变换生成（见 3.3），执行见第 4 节；
- 级联结构下所有信号都经过 FIR，总延迟由 FIR 路径决定（IIR 延迟为 0），
  段归属不影响最终延迟。

### 3.3 目标曲线与跨分频点拟合

```text
T(f) = Σ 所有段的解析频响（dB 域相加）
L(f) = Σ fc<200 段的解析频响（dB 域相加）
F(f) = T(f) − L(f)          // FIR 目标（dB 域相减）
```

- 总响应 = IIR 路径 × FIR 路径（级联，幅度相乘 = dB 相加）=
  `L(f) + F(f) = T(f)`，整体幅度精确等于目标曲线；
- **全频段衔接（非线性切断）**：F(f) 在全部频点（DC..Nyquist）合成，
  FIR 是**全频段补偿 IR**（不是 200 Hz 以上才生效的高通）——fc≥200 的段
  在 <200 Hz 的低频残余由 FIR 自己实现；fc<200 的段在 >200 Hz 的高频残余
  被 FIR 扣除；跨分频点的段由级联自然衔接；
- **主实现 vs 补偿（衔接机制）**：fc<200 的段由 IIR **主实现**（完整频响，
  含其 >200 Hz 残余）；fc≥200 的段由 FIR **主实现**（完整频响，含其
  <200 Hz 残余）。另一侧残余由 F(f) 自动处理：IIR 段的高频残余在 T−L 中
  扣除（不重复实现）；高频段在低频的残余由 F(f) 连续覆盖——远离中心后
  peaking 频响平滑缓变（非陡峭），1024+ 点 FIR 表达足够。总响应 = L·F = T
  全频段恒等，分频点两侧连续、无切断。
- 跨分频点的段自动处理：fc≈200 的段若进 IIR，其 >200 Hz 残余由 FIR 目标
  扣除；若进 FIR，IIR 不参与，FIR 目标即 T(f)；
- 最小相位变换复用 `graphic_eq.rs` 的 cepstrum 逻辑，改造成
  「任意频响数组 → 最小相位 IR」（对数插值节点不再需要，频响直接由
  peaking 解析公式逐频点合成）。

## 4. 采样率自适应 FIR 引擎

### 4.1 抽头数

```text
N = next_pow2(round(sr × 0.0213))，夹在 [1024, 8192]
```

| 采样率 | 抽头数 | 群延迟（≈N/2 @sr） |
|--------|--------|---------------------|
| 44.1k / 48k | 1024 | ≈10.7 ms @48k |
| 96k | 2048 | ≈10.7 ms |
| 192k | 4096 | ≈10.7 ms |
| 384k | 8192 | ≈10.7 ms |

### 4.2 执行方式

- `N ≤ 2048`：直接时域 FIR（复用 `convolution::dot` AVX2/FMA，运行时探测）；
- `N > 2048`：分块 FFT（块大小沿用 `CONVOLUTION_PARTITION_SIZE = 128`）：
  - **输出驱动 + 零填充补块**：块不满时按输出需要补零处理并完整发射，
    流停止/切设备时尾音不被压块丢弃（修复历史“切换嗡声”）；
  - **帧数自适应**：首次 process 的帧数与初始化预期不符时重建块缓冲
    （参考 EAPO `beforeFirstProcess` 处理）；
  - 分块 FFT 计划与缓冲全部在 initialize（非 RT）预分配，process 零分配。

## 5. 延迟与补偿

- `HybridPeqFilter::latency()` 返回 FIR 路径延迟（直接 FIR = 最小相位群延迟
  保守估计 N/4，1024 → 256；分块 FFT = 块大小 128）；
- **不上报、不补偿（v9.12 定稿）**：`latency_frames_atomic` / `latency_samples`
  恒为 0，`GetLatency` 无 child 返回 0——向引擎上报/补偿会导致帧协商错位、
  热重载与切歌播放卡住（2026-08-10 实证；v9.11 尝试激活后 v9.12 回退）；
  链延迟隐藏，音频整体滞后（直接 FIR ≤2047 采样 / 分块 128 采样）不可闻；
- `MAX_APO_LATENCY_SAMPLES = 8192` 保留为引擎按 `CalcInputFrames` 可能多给帧
  的缓冲安全余量。

## 6. 工厂与注册

- 新增 `pipeline/dsp/peq_hybrid.rs`（`HybridPeqFilter`），经
  `pipeline/dsp/factory.rs` 静态 `match EffectType::Peq` 分派构造
  （不再有 `PeqHybridFactory` 动态注册）；
- 移除 `GraphicEQ:` 命令：`config/commands/graphic.rs` 删除、parser 用例
  与指纹测试迁移为 TOML `[[effects]] type = "peq"`；
- `IIR,PK` / `Filter:` 等基础命令保留（PEQ 内部 IIR 复用其 biquad 实现，
  但作为独立滤波器不依赖其命令；命令层整体由 TOML 模型取代，见
  `config TOML 设计文档.md`）；
- `Convolution:` 不纳入本架构（保持现状，不做 IR 卷积适配）。

## 7. 测试计划

- 解析：单块段数 1–31 边界 + 全局合计 ≤ 31、缺/重/未知键、非法数值、单位可选、空参数
  passthrough、spec 指纹随参数变化；
- 频响拟合：单段与多段（含 fc = 100/150/200/250 跨频点组合）下，级联输出
  频响与目标曲线偏差 < ±0.5 dB（FFT/Goertzel 验证，200 Hz 两侧覆盖）；
- 引擎一致性：同一 IR 直接 FIR 与分块 FFT 输出逐样本一致；短流尾部完整；
  连续不同 frame_count 调用无错位；
- RT 安全：极端参数（31 段、±30 dB、Q=12、8192 抽头 @384k）有限、确定性、
  静音直通、多通道；
- 延迟：`chain.total_latency()` 与 `latency_frames_atomic` 写入一致；
- 回归：移除 `GraphicEQ:` 后 parser / factory / 文档 / config 测试用例更新。

## 8. 迁移

- config 测试用例：`GraphicEQ.txt` / `PEQ.txt` 文本文件 → `config.toml`
  样例（`[[effects]]` + `[[effects.bands]]`，原 31 点曲线迁移为 1–31 段
  peaking 近似）；
- 旧 txt 一次性转换由 `config convert` 承担（迁移期工具，见
  `config TOML 设计文档.md`）；
- 文档同步（定稿时）：`config 6.10`（GraphicEQ）废弃说明、
  `config 6.16` 效果器、`pipeline 4.18/4.22`、`Equalizer 行为文档 2.10`、
  `config 模块规范.md`、`CLI 引用规范.md`、`模块引用规范（无详细模块版）.md`、
  changelog、roadmap P1-6/P1-7。
