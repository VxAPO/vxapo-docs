# Roadmap 维护规则（强制）

## 规则

**任何新功能、新需求必须先登记路线清单（`roadmap.md`），禁止"规范外直接实现"。**
**状态流转必须遵守本规则的状态机与硬门禁。**

## 状态机

```
Backlog（待办）→ Spec-Drafting（规范起草）→ Spec-Finalized（规范定稿）
    → Implementing（实现中）→ Done（完成）
```

| 状态 | 含义 | 进入条件 |
|------|------|----------|
| `Backlog` | 已登记待办，尚未选中 | 通过 `roadmap-add-item` skill 登记 |
| `Spec-Drafting` | 规范起草中：核对/补写该功能涉及的规范章节 | 任务被选中开始规划；**若有依赖，前置任务须至少 Spec-Finalized** |
| `Spec-Finalized` | 规范定稿：落点回填、门禁放行 | 通过 `spec-finalize` skill 审查定稿 |
| `Implementing` | 执行端 agent 按规范落地 | **仅 Spec-Finalized 可进入**（硬门禁） |
| `Done` | 实现完成并通过合规核对 | 规范侧核对通过后由规范侧标记并归档 |

## 触发条件与 Skill 引用

| 用户意图 | 动作 |
|----------|------|
| "登记路线清单" / "add to roadmap" / "new roadmap item" / "plan this feature" | 激活 **`roadmap-add-item`** skill，仅登记，不涉及状态流转 |
| "定稿" / "规范审查" / "finalize spec" / "spec review" | 激活 **`spec-finalize`** skill，执行定稿审查 + 三文件联动 |
| 其余状态流转（进入 Spec-Drafting、进入 Implementing、标记 Done） | 由本规则直接管理，无需 skill |

## 硬门禁

1. **仅 `Spec-Finalized` 可进入 `Implementing`**——实现之前，规范已对其引用约束、模块边界、RT 安全做过审查。
2. **禁止"绕过规范先实现"**：实现中发现规范不足 → 任务状态回退 `Spec-Drafting`，修订规范后再继续。
3. **Backlog 不写 changelog**：登记仅更新 roadmap.md；当条目促成规范变更、版本递增时，才由 `spec-finalize` 按 changelog-rule 记录。

## 反馈外置（v8.0）

**执行端实现中发现的规范问题反馈，统一记录于根目录 `feedback.md`，禁止写入 `roadmap.md`**（维护规则见 `.clinerules/feedback-rule.md`）。
roadmap 条目命中的反馈**只留引用**：`> 反馈记录：feedback.md #PX-X`；状态机/DoD 不受反馈外置影响。

## 三文件联动

一次版本变更必须同时满足：

| 文件 | 动作 | 规则 |
|------|------|------|
| `roadmap.md` | 任务状态更新（如 → `Spec-Finalized`）、规范落点回填 | 本规则 |
| 规范文档（主规范/子规范） | 对应章节落地、主规范版本号递增（如 v6.9 → v6.10） | 本规范 + changelog-rule |
| `changelog.md` | 新增记录（**严格遵循 changelog-rule 的`记录格式`**） | changelog-rule |

**执行顺序（重要，v8.2 补强）**：本次变更暴露「先改 roadmap 状态、后补子规范」的顺序错误（P0-5 教训）——
**定稿必须先落地规范正文，再改 roadmap 状态**。正确顺序：

1. **先改规范正文**（含对应子规范条目——如 object 7.1.11/7.1.12；若有子规范必须同步修订）
2. **再递增主规范版本号**（如 v8.1 → v8.2）
3. **写 changelog**（严格 changelog-rule 格式，标注精确章节号）
4. **最后改 roadmap 状态**（`Spec-Finalized` + 规范落点回填）

> **禁止**：仅改主规范版本号 + changelog + roadmap 状态，却**遗漏子规范对应条目**的落地
> （如 P0-5 定稿漏改 object 7.1.11/7.1.12 的 catch_unwind 语义——被用户指正后补）。
> 子规范条目缺失 = 定稿不完整 = 执行端无法按规范落地。

**一致性校验**：changelog 版本号 == 主规范版本号 == roadmap 条目规范落点引用版本。三者缺一不可。

## 提交前自检

- [ ] 若存在 `Spec-Finalized` 状态变更：**先改规范正文（含对应子规范条目）**、主规范版本号已递增、changelog 已按 changelog-rule 记录、roadmap 规范落点已回填
- [ ] 若存在 `Done` 状态变更：规范侧合规核对记录已附
- [ ] 无 `Implementing` 条目违反"仅 Spec-Finalized 可进入"门禁
- [ ] 无"规范外直接实现"的功能（未登记 roadmap 即实现）
- [ ] 子规范对应条目已落地（roadmap 落点中引用的每个子规范章节号在对应子规范文件中确实存在）

## 范围约定

- roadmap.md 从本规则生效日起维护，此前已完成的工作（规范 v6.0-v6.9 已落地的决策）不强制补录 Backlog 登记；
  但**后续新增功能**一律必须登记。
- 路线清单中 P0/P1 条目的顺序即优先级顺序（P0 在前）；P2 为方向预留，登记时需补全字段。

## 反例（禁止）

- 未登记 roadmap 直接开始实现
- 任务在 `Spec-Drafting` / `Backlog` 状态即被实现（跳过硬门禁）
- 版本号递增但 roadmap 条目状态未同步为 Spec-Finalized
- changelog 记录了未在 roadmap 中登记的功能