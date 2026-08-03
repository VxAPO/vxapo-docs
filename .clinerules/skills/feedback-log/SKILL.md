---
name: feedback-log
description: Log a feedback entry to feedback.md. Use when the implementation (executor) discovers a specification problem (infeasibility, omission, contradiction, defect) during implementation, or when a completed implementation report includes potential issues, or when the user/spec side proposes revision opinions on landed specifications. Trigger on phrases like "反馈记录", "发现规范问题", "add feedback", "log feedback", "潜在问题反馈".
---

# 记录实现反馈（外置 feedback.md）

仅负责**登记反馈记录**（写入 feedback.md），不涉及规范修订、状态流转或 changelog。

## 前置检查

1. 读取 feedback.md，确认该问题尚未存在（避免重复记录）；若同条目已存在且为后续补充，则**追加**到对应 `[PX-X]` 的「修订记录」而非新建。
2. 确认反馈**不属产品意图（intent.md）意愿变更**——意图变更走 `.clinerules/intent-rule.md` 解耦治理，不落 feedback.md。
3. 确认反馈**不写入 roadmap.md**——roadmap 只留引用（`> 反馈记录：feedback.md #PX-X`）；反馈正文一律落 feedback.md。

## 步骤

### Step 1：确定条目标识与影响版本

| 字段 | 填写规则 |
|------|----------|
| 标识 | `[PX-N-M]`——PX-N 为 roadmap 条目编号；M 为该条目下反馈序号（P0-4 第 3 条 → `[P0-4-3]`） |
| 影响版本 | 规范 vX.Y（首次涉及的定稿版本）；若属后期修订则标「vX.Y 起」 |
| 来源 | 执行端反馈 / 用户决策 / 规范侧 |

### Step 2：写入 feedback.md（逆序插入）

**最新反馈在最上**——新记录插入文件顶部（标题与分隔线之后），而非追加到尾部。

格式：

```markdown
### [PX-N-M] <一句话问题标题>（vX.Y + 来源）

**影响版本**：规范 vX.Y 起

**问题**：
- 分类：规范文本矛盾 / 原型不可实现 / 规范遗漏 / 实现缺陷发现 / 实现完成报告随附潜在问题
- 具体描述（触发场景 / 现象 / 根因）

**规范侧判定**：
- 已确认缺陷（对应章节）/ 需用户决策 / 维持现状 + 理由

**修订记录**（规范侧处理，可多轮）：
- vX.Y：<修订动作摘要>——**对应章节**：`文件 节号`

**状态**：待修订 / 已修订（vX.Y） / 维持现状
```

### Step 3：roadmap 引用

在对应 roadmap 条目（如 P0-4）追加一行引用（若尚未有）：

```
> 反馈记录：`feedback.md #PX-N-M`（问题标题）
```

仅一行引用，**不复制反馈正文**。

### Step 4：无 changelog

**登记反馈 ≠ 规范修订**——仅写入 feedback.md + roadmap 引用，**不递增版本、不写 changelog**。反馈驱动的修订才由 `spec-finalize` 按 changelog-rule 记录。

### Step 5：自检

- [ ] 条目含 `PX-N-M` + 影响版本 vX.Y + 来源
- [ ] 已逆序插入（最新在最上）
- [ ] roadmap.md 仅一行引用（未复制正文）
- [ ] 未写 changelog / 未递增版本（登记阶段）
- [ ] 非 intent 意愿变更（意图走 intent-rule）
- [ ] 非重复条目（同问题合并到既有 `[PX-X]` 修订记录）

## 后续路径（不属于本 skill）

- 规范侧核对反馈 → 修订规范（版本递增 + changelog）→ feedback.md 追加「修订记录」+ 状态「已修订」——由规范侧执行，参照 `.clinerules/feedback-rule.md`
- 修订完成 → roadmap 条目标记 Done（roadmap-rule 管理）