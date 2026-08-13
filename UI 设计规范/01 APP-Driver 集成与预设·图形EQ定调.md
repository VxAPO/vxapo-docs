# VxAPO UI 设计规范 — 01 APP-Driver 集成与预设·图形EQ定调

> 状态：设计定稿（v1，2026-08-11）｜ 对应仓库：docs（模块引用规范）
> 本文档定调 APP 与 driver 的结合方式，以及「预设视图 / 高级视图」的底层语义；
> 交互细节由 `02 预设视图交互细则`、`03 高级视图交互细则` 承接。

---

## 一、总纲与数据流

- **职责边界**：APP 是唯一编辑方，driver 是唯一生效方。APP 生成 `config.toml`，
  driver 解析并在音频链上生效；driver 不向 APP 回写调音数据。
- **单文件往返**：所有调音数据（含 APP 语义层）保存在
  `C:\ProgramData\VxAPO\{GUID}\config.toml`；保存后经现有 watcher 热重载生效。
- **废弃**：APP 旧 `lib/config.ts` 的 config.txt 生成逻辑废弃，改为按本文档契约
  生成 TOML。

---

## 二、视图定调：预设视图 / 高级视图

- **预设与高级不是模式，是同一份数据的两种视图**。切换视图不改变数据、不丢分组、
  不拆卡。
- 页面为单页布局：
  - **底部固定区**：合成频响曲线（始终可见）+ 运行参数（VxAPO 安装状态、调音开关、
    配置保存状态、响度校正、当前设备声道数 / 采样率 / 位深）。
  - **上部视图区**：按标签在「预设视图」「高级视图」之间切换。

---

## 三、数据契约（APP 内存模型 ↔ TOML）

### 3.1 块模型

每个「块」= 一个 `[[effects]] type="peq"`：

| APP 模型 | TOML | 说明 |
|---|---|---|
| `group`（可选） | `group` | 大组 / 预设语义；无值 = 无组裸 band |
| `name`（可选） | `name` | 组内卡片语义；仅 group 存在时有意义 |
| `enabled` | `enabled` | 仅卡片级开关（无 band 级开关） |
| `crossover_hz` | `crossover_hz` | 默认 200 |
| `bands[]` | `[[effects.bands]]` | `fc` / `gain_db` / `q` |

- 无组裸 band = 无 group / name 的单 band 块（1 段）。
- 块按 APP 列表顺序**保序**写入；driver 按顺序级联处理。
- `name` / `group` 仅供 APP 语义层，driver 解析后忽略；**不新增任何元数据字段**
  （语义即 name / group）。
- **顶层总开关（v9.18）**：`enabled = false` = 整链 passthrough（driver 直接
  空链，文件内容保留、不参与校验）；APP 的调音开关（标签页圆点 / 全局调音）
  写入该字段，**不替换/清空效果内容**。

### 3.2 段数约束

- 单块：1..31 段（v9.16 已落地，`MIN_PEQ_BANDS = 1`）。
- 全局：所有 peq 块 band 总数 ≤ 31（v9.16 已落地，config 层跨块校验）。

---

## 四、预设视图

- 按 `group` 聚合为大圆角块；组内按 `name` 分小卡，小卡宽度 = 滑块数。
- 每个 `name` 对应**单个 band**（不做一 name 多段）；语义卡自上而下：
  `name`（强）→ 语义副标（弱）→ **横排滑块** → 数值。
- 每个卡片头带**顺序标号**（组 `01`，name `01.1` …），拖拽可重排
  （顺序 = 链顺序）。
- 删除 = **整组删除**（连同组内所有 name 卡）。
- 点击组标签：呼出菜单（重命名 / 保存为预设 / 删除）。
- APP 解析只认 **TOML 顺序 + 有无 group / name**，不比对已知预设名；TOML 中出现
  未知 group / name 自动登记进预设库。

---

## 五、高级视图

- 同一份数据；有组块仍以块形式平铺（**编辑不拆卡**）；单个 peak 卡自上而下：
  Fc → Q → Gain 滑块 → Gain 数值；卡片头带顺序标号。
- 无组 band 以单滑块形式独立增删（全局 0~31）。
- 曲线始终可见，band 可直接在曲线上拖拽。

---

## 六、自定义预设闭环（安卓桌面文件夹式）

- 高级视图中把 peaking A 拖到 peaking B 上 → 合并为一个 `group`。
- 组内再拖 → 生成 `name`（小组）；把 band 拖出组 → 解除分组。
- 点组标签菜单「保存为自定义预设」→ 命名弹窗以图形方式展示当前组块，用户设定
  name / group。
- 也可切到预设视图直接点击文本重命名。
- 预设可删除；删除预设不删除已应用的块（块是数据，预设是库条目）。

---

## 七、TOML 导入双选项

- ① 完整导入：保留全部参数（含 fc）。
- ② 仅导入预设结构：只解析 group / name / q / gain 与其他效果器参数，**丢弃 fc**。
- 无 fc 的库内预设 band 应用时落到默认锚点（20–20k 对数均匀分布）并立即在曲线上
  可拖。

---

## 八、段数预算

- 每个预设声明自身段数。
- 应用前校验：当前总数 + 预设段数 ≤ 31；超限阻止应用并提示
  「预设段数 / 当前总数 / 剩余」。

---

## 九、Driver 契约调整（v9.16 已落地）

1. `MIN_PEQ_BANDS`：6 → 1（允许单段卡 / 无组单 band）。
2. `config/model.rs`：新增跨效果块的全局 peq band 总数 ≤ 31 校验
   （错误信息：`total 'peq' bands count N exceeds max 31`）。
3. 其余现状已满足：卡片级 `enabled`、`name` / `group` 忽略、保序级联。

---

## 十、完整 TOML 示例

```toml
version = 1

[meta]
app = "vxapo"
schema = 1

# —— FPS 预设（group 大组）——
[[effects]]
type = "peq"
group = "FPS 预设"
name = "脚步声增强"
enabled = true
crossover_hz = 200

[[effects.bands]]
fc = 250
gain_db = 4.0
q = 1.2

[[effects.bands]]
fc = 500
gain_db = 2.0
q = 1.0

[[effects.bands]]
fc = 900
gain_db = 1.5
q = 1.5

# —— FPS 预设内第二张卡（name 小组，单段）——
[[effects]]
type = "peq"
group = "FPS 预设"
name = "枪声增强"
enabled = true
crossover_hz = 200

[[effects.bands]]
fc = 3200
gain_db = 3.0
q = 2.0

# —— 高级视图下的无组裸 band（单段）——
[[effects]]
type = "peq"
enabled = true
crossover_hz = 200

[[effects.bands]]
fc = 10000
gain_db = -2.0
q = 1.0
```

> 说明：本示例全局段数 = 3 + 1 + 1 = 5，符合全局 ≤ 31；「枪声增强（1 段）」与
> 「无组裸 band（1 段）」自 v9.16 起可直接被 driver 解析生效。

---

## 十一、曲线预览

- 曲线 = 全部启用 band 的合成响应，APP 侧以 RBJ 公式求和近似；与 driver
  IIR / FIR 混合实现存在微小差异，以 driver 实际听感为准。

---

## 十二、后续文档规划

- `02 预设视图交互细则`：大圆角 / 小卡 / 滑块布局、重命名、菜单、删除确认。
- `03 高级视图交互细则`：拖拽成组 / 拆组、命名弹窗、单 band 增删、曲线上拖拽。
- `04 设计令牌与组件`：继承 v1 三 TAB 草案（`_归档/`）中的令牌与组件基础。
