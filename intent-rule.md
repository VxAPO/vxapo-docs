# intent-rule：产品意图对路线清单的约束规则

> **本文件定义 `intent.md`（产品意图）如何约束 `roadmap.md`（路线清单）的决策。**
> `intent.md` 只承载产品定义本身（是什么、为谁、边界、约束从哪来）；
> 本文件承载**治理规则**——两者分离，互不混淆。

---

## 一、层级关系

```
intent.md（产品意图——为谁/为何做，单一权威）
    ↓ 本文件（intent-rule：意图 → 路线的约束规则）
roadmap.md（路线清单——要做什么）
    ↓
规范文档（模块引用规范 + 子规范——怎么做）
    ↓
实现（执行端 agent）
```

- `intent.md` 是**产品意图的单一权威来源**。
- `intent-rule.md` 只描述「意图如何约束路线」，不定义产品内容本身。
- `roadmap.md` 是意图的**执行规划**，必须可追溯、不与意图冲突。

---

## 二、硬性规则

### R1. 单一权威

`intent.md` 是产品意图的唯一定义。roadmap / 规范 / 实现中出现的任何「产品意图」表述，
若与 `intent.md` 冲突，**一律以 `intent.md` 为准**，并回退修订该表述来源。

### R2. 可追溯性

**roadmap 每一个条目必须能追溯至 `intent.md` 的至少一个具体章节**（如「用户行为模型」
「概念模型」「Filter 类型边界」）。登记新条目（`roadmap-add-item` skill）时须填写
「意图落点」字段：`intent.md 第 X 节`。

- 无法映射到用户可感知价值、或无法对应 intent 章节的功能**不登记**。
- 反向：intent 中已有的产品承诺（预设系统、多预设叠加、设备继承、单设备禁用等）
  必须在 roadmap 中**有对应条目**——意图承诺了但路线没规划 = 意图悬空，属缺口。

### R3. 冲突红线

**roadmap 不得登记与 `intent.md` 冲突的功能。** 冲突判定：

| 冲突类型 | 示例 | 处理 |
|----------|------|------|
| 直接冲突 | roadmap 加「DLL 主动写预设」→ 违反五节「驱动层不主动写 config」 | 登记被拒绝 |
| 边界冲突 | roadmap 加「App 直接建 Chain」→ 违反五节「应用层不触碰 pipeline/RT」 | 登记被拒绝 |
| 范围冲突 | roadmap 加「Capture 全量支持（v1）」→ 与六节「Capture 保留 AEC 决策」不符 | 需先修订 intent 六节 |
| 语义冲突 | roadmap 加「预设直接引用 Filter」→ 违反四节「Preset 不直接引用 Filter」 | 登记被拒绝 |

> 以上冲突一旦触及产品语义（而非纯技术机制），**必须修订 `intent.md` 本身**，
> 而非在 roadmap 层"绕开"。意图先行，规划随后。

### R4. intent 变更治理（解耦）

**`intent.md` 的变更：**

- ❌ **不需要 changelog**——intent 是产品意愿的反馈，与规范实现解耦，不属于「规范版本变更」
  （与 changelog-rule 的触发条件无关）。
- ❌ **不需要主规范版本递增**——intent 不直接改变规范语义。
- ✅ **需要用户确认**——intent 编辑本身属重大决策（用户意愿的固化）。
- ✅ **需要评估 roadmap 影响**——修订后检查 roadmap 中**未进入 Implementing** 的条目：
  - 与新意图冲突 → 回退 `Spec-Drafting` / `Backlog` 并标注原因；
  - 与新意图一致但缺条目 → 登记新条目或扩展现有条目；
  - 已进入 `Implementing` 的条目若被意图推翻 → 回退 `Spec-Drafting`，修订规范后重新放行。

### R5. 实现冲突回退

执行端实现中发现「规范与 intent 冲突」时（同 `主规范 十七` 反馈闭环的零容忍精神）：

```
实现中发现 spec 行为违背 intent.md
    → 停止实现（不改状态）
    → 反馈：标注冲突的 intent 章节 + spec 章节
    → 规范侧核对：
        ├─ intent 对 → 修订 spec（常规版本递增流程）
        └─ spec 对（intent 表述有误）→ 修订 intent（用户确认，不递增版本）+ 澄清 spec 注释
```

## 三、与现有治理机制的关系

| 机制 | 关系 |
|------|------|
| `roadmap.md + roadmap-rule` | intent-rule 是 roadmap-rule 的**上位输入**——roadmap-rule 管理状态机执行，intent-rule 管理"什么能进路线" |
| `主规范 十六（路线治理）/ 十七（实现反馈闭环）` | intent-rule 补充其**准入判据**（可追溯性 + 冲突红线）与**意图冲突回退**路径 |
| `changelog-rule` | **不受 intent 变更触发**——changelog 只记录规范实现变化 |
| `spec-finalize` skill | 定稿审查新增一项：核对规范落点是否符合 intent.md（若涉及产品语义） |

## 四、自检清单

- [ ] 新 roadmap 条目：已填「意图落点」（`intent.md 第 X 节`）？
- [ ] 新 roadmap 条目：与 intent 无冲突（R3）？
- [ ] intent 变更：已获用户确认？
- [ ] intent 变更：未触碰 changelog / 主规范版本？（除非同时有规范语义变更）
- [ ] intent 变更后：roadmap 未进入 Implementing 的条目已重新评估？
- [ ] 实现冲突：已走 R5 回退路径并记录？

## 五、反例（禁止）

- roadmap 登记无「意图落点」的条目（无法追溯 → 不登记）
- intent 已明确排除的功能（如 v1 不预留 Filter 扩展点）仍被 roadmap 规划
- intent.md 变更却写了 changelog / 递增了主规范版本（除非该变更同时伴随规范语义变化）
- 实现违背 intent 但"绕过去"继续编码（零容忍）