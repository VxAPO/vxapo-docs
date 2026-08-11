# config TOML 设计文档（模型驱动，v1 模型 Spec-Finalized）

> 状态：已实现（v9.11，roadmap P1-7 完成）。核心内容已并入 `config 模块规范.md` /
> `pipeline 模块规范.md` / `CLI 引用规范.md` 对应章节。

## 1. 目标与原则

彻底放弃 `config.txt`（EAPO 风格逐行命令），改用标准 TOML。原则：

- **模型驱动**：一份 `serde` 结构体三端共用——driver 反序列化并构造 Filter 链、
  CLI 读写、APP（Tauri，Rust 后端）经 JSON bridge 编辑展示；
- **现代语义**：类型化、显式键、数组表表达浮动段数，错误定位到键/段；
- **不做 EAPO 对齐**：命令解析/工厂注册不再兼容 EAPO 语法；
- **不做 IR 卷积**：`Convolution:` 不进入模型。

## 2. 文件与设备模型

- 每设备一个文件：`C:\ProgramData\VxAPO\{guid}\config.toml`（路径与现
  config.txt 一致，仅扩展名变更）；
- 不再需要 `Device:` / `If:` / `Include:` 等文件内条件/包含命令：
  设备隔离由文件系统天然承担，效果链按书写顺序执行；
- CLI 安装时自动导入 `vxapo-cli.exe` 同级 `config.toml`（原 config.txt
  约定同步改名）。

## 3. Schema（v1）

```toml
# VxAPO 配置 v1
version = 1

[meta]
app = "vxapo"          # APP 元数据，driver 忽略
schema = 1             # APP 元数据，driver 忽略

[[effects]]
type = "preamp"
gain_db = -3.0

[[effects]]
type = "peq"
enabled = true
crossover_hz = 200            # 默认 200；Fc≥crossover 归 FIR，Fc<crossover 归 IIR
name = "脚步声增强"            # 可选，APP 展示/映射用，driver 忽略
group = "FPS 预设"             # 可选，APP 卡片组归属，driver 忽略
[[effects.bands]]
fc = 1000                    # Hz，[20, 20000]
gain_db = 3.0                # dB，[-30, +30]
q = 1.0                      # [0.1, 12.0]
[[effects.bands]]
fc = 2500
gain_db = -2.0
q = 2.0

[[effects]]
type = "wide"
intensity = 0.354331
```

- `effects`：效果器数组，**顺序即处理顺序**（数组表保序）；
- 每个效果器 `type` 必填，参数键类型化（f32/整数/字符串/布尔）；
- 每效果器可用 `enabled = false` 显式旁路（缺省 true）；
- `name` / `group`：可选，**APP 元数据**——`name` 用于 APP 展示与预设映射，
  `group` 用于 APP 恢复卡片组归属；**driver 解析时完全忽略**，不参与 DSP
  构造与 spec 指纹；
- `[meta]`：可选，APP 元数据（schema 版本等），driver 忽略；
- 校验失败（未知 type、缺必填键、超范围、类型错误）→ 整文件解析失败，
  保留旧链（与现行热重载失败语义一致）；
- 空文件 / 无 `effects` = passthrough。

### 3.0 分层语义（定稿）

- **生成靠 APP，解析靠 driver**：`name` / `group` / `[meta]` 只由 APP 写入
  （预设库展开时生成），driver 只消费 DSP 字段（`type` / 参数 /
  `enabled` / `channels`）；
- **双模型分层（定稿）**：
  - `FileModel`（config 层，如 `config/model.rs`）：TOML 反序列化的**文件格式
    模型**，含 `version` / `[meta]` / `effects`（含 `name`/`group` 等 APP
    元数据），字段与文件一一对应；
  - `ChainModel`（dsp 层，如 `pipeline/dsp/model.rs`）：**DSP 语义模型**，
    只含效果器类型、参数、`enabled`、`channels`，无 `name`/`group`/`meta`；
  - **转换是 config 层的职责**：`FileModel → ChainModel`（丢弃 APP 元数据、
    校验范围/段数/声道名）在 config 层完成；依赖方向保持
    `config → pipeline/dsp`，dsp 层不引用 config；
- **spec 指纹只含 DSP 字段**：`name` / `group` / `meta` 变化不触发热重载；
- **外部导入（无元数据）**：预设语义无法恢复，导入规则 = 整条链包装成一个
  新预设（按顺序生成默认 `name`），用户可在 APP 手动重组；`name` 仅作展示
  与“预设库恰好同名”时的可选关联提示，不作为结构依据；
- **跨设备迁移**：本 APP 导出的 config.toml 自带 `group`，拷贝文件即可恢复
  卡片组结构，无需同时迁移 APP 预设库。

### 3.1 效果器类型与参数（v1）

| type | 参数 | 说明 |
|------|------|------|
| `peq` | `crossover_hz`（默认 200）、`bands`（数组表，6–31 段：`fc/gain_db/q`） | 混合式 PEQ，见 `PEQ 设计文档.md` |
| `preamp` | `gain_db` [-120, +48] | 全局增益 |
| `delay` | `ms`（上限 1000） | 延迟 |
| `copy` | `source`/`dest` 或映射表 | 通道复制 |
| `aural` | `tune_hz`/`drive`/`odd`/`even`/`wet`/`dry` | 与现命令参数一致 |
| `reverb` | `room_size`/`decay`/`damping`/`bandwidth`/`density`/`lat5`/`lat6`/`pre_delay_ms`/`motion_rate`/`motion_depth_ms`/`wet`/`dry` | 与现命令参数一致 |
| `maximizer` | `gain_boost_db`/`max_output_db`/`release_ms`/`target`/`lookahead_ms`/`dither`/`wet`/`dry` | 与现命令参数一致 |
| `wide` | `intensity` | 与现命令参数一致 |
| `loudness` | `phon`/`reference_phon` | 等响校正 |
| `vst` | — | v9.11 移除（不做 IR/外部插件加载） |
| `biquad` | `type`（PK/LP/HP/LS/HS/AP/NO）、`fc_hz`/`gain_db`/`q` | 基础滤波 |

### 3.2 移除与保留

- **移除**：`GraphicEQ:`（由 `peq` 取代）、`Device:`/`If:`/`Include:`/`Stage:`、
  `REW:` 运行时解析、`Convolution:`（不做 IR 卷积）；
- **保留语义**：`IIR,PK`/`Filter:` 合并进 `biquad`/`peq` 类型；
  `Channel:` 用效果器级 `channels = ["L", "R"]` 字段表达（缺省全部通道）；
- **REW 导入**：保留为“导入时转换”——CLI/APP 读 REW 导出文本，转为
  `peq.bands` 写入 config.toml（运行时不再解析 REW 格式）。

## 4. 指纹与热重载

- spec 指纹改为「模型序列化哈希」：`serde` 序列化后取 hash
  （`effects` 顺序、全部 DSP 参数参与；`name`/`group`/`meta` 不参与），
  不再基于文本行；
- watcher 监控 `config.toml`（mtime/size 预检语义沿用 v9.5）；
- 空参数/空链语义：文件存在但无 `effects` = 显式 passthrough，
  与「文件不存在」区分（指纹不同，热重载可切换）。

## 5. CLI / APP 交互

- driver 暴露 `parse_config_toml(bytes) -> ChainModel`（控制线程调用）；
- CLI：`config get`/`config set`/`config import-rew` 基于模型读写；
- APP（Tauri）：Rust 后端 `toml::to_string`/`from_str`，前端只消费 JSON
  模型（`tauri::State` 桥接），不解析 TOML；
- 依赖：driver 新增 `serde`（derive）+ `toml`（仅控制线程路径，
  RT 线程零解析）。

## 6. 迁移步骤

1. 定义 `ChainModel`/`EffectConfig` 结构体（serde，默认值即现有效果器默认）；
2. 重写解析层：`config/parser.rs` → TOML 反序列化 + 校验 → `Filter` 链
   （registry 按 `type` 构造，工厂接口不变）；
3. watcher/指纹/诊断日志同步；
4. CLI 安装导入改名 `config.toml`；提供 `config convert`（旧 txt → toml
   一次性转换，仅迁移期）；
5. APP 接入 JSON 模型；
6. 文档与测试全量更新（config 6.x、pipeline 4.x、CLI 引用规范、
   模块引用规范、changelog、roadmap P1-7）。

## 9. 源码树重构方案（v9.11，定稿）

### 9.1 目标树（driver/src）

```text
config.rs                     保留（模块入口）
config/parser.rs              重写：TOML → FileModel → ChainModel → 链
config/model.rs               ★ 新增：FileModel（serde，含 name/group/meta）
config/watcher.rs             保留（监控 config.toml，指纹 = ChainModel 哈希）
config/error.rs               保留（扩展 TOML 错误）
config/commands.rs            ✗ 删除（不再逐行命令）
config/commands/*（14 个）     ✗ 删除

pipeline/dsp.rs               模块声明同步
pipeline/dsp/model.rs         ★ 新增：ChainModel / EffectType / 各效果器参数
                              （serde 仅用于 config 反序列化；dsp 不依赖 config）
pipeline/dsp/factory.rs       重构：静态 match EffectType → Box<dyn Filter>
                              删除 FilterFactory/FilterRegistry/index/OutcomeKind
pipeline/dsp/fir.rs           ★ 新增：SIMD dot/init_fir_simd（自 convolution.rs
                              迁出）+ 直接 FIR 延迟线 + 分块 FFT 引擎（PEQ 内部）
pipeline/dsp/peq_hybrid.rs    ★ 新增：混合 PEQ
pipeline/dsp/filter.rs        保留（Filter trait）
pipeline/dsp/transition.rs    保留
pipeline/dsp/math.rs          保留
pipeline/dsp/aural.rs         保留
pipeline/dsp/reverb.rs        保留
pipeline/dsp/maximizer.rs     保留
pipeline/dsp/wide.rs          保留（dot 引用切到 fir.rs）
pipeline/dsp/loudness.rs      保留
pipeline/dsp/gain.rs          保留（preamp）
pipeline/dsp/graphic_eq.rs    ✗ 删除（PEQ 取代）
pipeline/dsp/biquad.rs        ✗ 删除（IIR biquad 实现并入 peq_hybrid）
pipeline/dsp/peq.rs           ✗ 删除
pipeline/dsp/convolution.rs   ✗ 删除（dot 迁出后）
pipeline/dsp/copy.rs          ✗ 删除
pipeline/dsp/delay.rs         ✗ 删除
pipeline/dsp/vst.rs           ✗ 删除
pipeline/dsp/hp_lp.rs         ✗ 删除
```

### 9.2 工厂分派（定稿）

- `factory.rs` 提供 `create_from_model(effect: &EffectConfig, ctx) -> Box<dyn Filter>`，
  静态 `match EffectType` 穷尽 7 个类型（peq/preamp/aural/reverb/maximizer/
  wide/loudness），编译器保证完整性；
- 参数已由 serde + config 层校验（范围/段数/声道名）完成，构造不失败；
- 删除 `FilterFactory` trait、`FilterRegistry`、`FilterCreateResult`、
  `OutcomeKind`、`index` 常量与注册顺序测试。

### 9.3 依赖先行项

- `wide.rs` 当前 `use convolution::dot/init_fir_simd`——先迁出到 `fir.rs`
  再删 convolution.rs；
- `rustfft` 保留（分块 FFT + 最小相位 cepstrum）；
- `serde`（derive）+ `toml` 新增依赖，仅控制线程路径使用。

### 9.4 实施阶段（每阶段可编译可测试）

1. 基础设施：serde/toml；`config/model.rs`（FileModel）+ `pipeline/dsp/model.rs`
   （ChainModel）；新建 `dsp/fir.rs`，`wide.rs` 切换引用；
2. PEQ 引擎：`peq_hybrid.rs`（IIR 级联 + 频响合成 + 最小相位 FIR，先直接 FIR）；
3. 解析层切换：`parser.rs` 重写（TOML → FileModel → ChainModel → 链），
   watcher 指纹改模型哈希，删 config/commands/* 与已移除 dsp 文件，
   测试迁移（TOML fixtures）；
4. 分块 + 延迟：`fir.rs` 分块 FFT（块跨调用累积，对齐 EAPO libHybridConv）
   接入 PEQ；延迟策略 v9.12 定稿为**不上报**（激活引擎帧数补偿实测卡住，回退）；
5. 收尾：CLI/APP、`config convert`、文档定稿合并、v9.11 归档。

## 7. 测试计划

- 反序列化：合法/缺键/未知 type/超范围/类型错误/空文件/顺序保持；
- 指纹：参数变化、顺序变化、空链 vs 无文件均产生不同指纹；
- 热重载：`config.toml` 修改触发重建、失败保留旧链；
- 迁移：`config convert` 对现有效果器参数逐键等价（默认值一致）；
- RT：解析只在控制线程，RT 无分配/无解析；
- 回归：现有效果器全量测试迁移到 TOML 构造路径后不变。
