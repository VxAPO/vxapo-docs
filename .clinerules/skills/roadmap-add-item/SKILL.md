---
name: roadmap-add-item
description: Add a new task entry to roadmap.md. Use when planning a new feature, starting a new Phase task, or when a requirement needs to be tracked in the roadmap. Trigger on phrases like "add to roadmap", "new roadmap item", "plan this feature", "登记路线清单".
---

# 新增路线清单条目

仅负责**登记**（Backlog 状态），不涉及状态流转。

## 前置检查

1. 读取 roadmap.md，确认该功能尚未存在（避免重复）
2. 登记阶段**不检查依赖状态**——Backlog 是待办池，允许登记依赖未定稿的任务。
   依赖检查的时机是：**本任务要进入 Spec-Drafting 之前**，才要求其依赖任务至少 Spec-Finalized
   （见 roadmap-rule 状态机）

## 步骤

### Step 1：确定条目信息

| 字段 | 填写规则 |
|------|----------|
| 编号 | P{n}-{序号}，如 P0-1、P1-3 |
| 优先级 | P0（阻塞后续）/ P1（核心功能）/ P2（增强） |
| 关联 Phase | 对应《VxAPO 完整方案》的 Phase N |
| 目标 | 一句话 |
| 影响模块 | 规范模块树中的路径（如 `object/apo.rs`、`config/commands/*.rs`）；若涉及规范仓库之外的 crate（如 vxapo-cli），需在"规范落点"预先声明其边界 |
| 依赖 | 前置任务编号，无则填"无" |

### Step 2：写入 roadmap.md

状态设为 `Backlog`。按优先级排序插入（P0 区域在前，P1 次之，P2 预留区）。规范落点留空（Spec-Drafting 时回填）。

格式：

```
### P{n}-{序号}  {标题}
- 状态：Backlog
- 优先级：P{n} ｜ 关联 Phase：Phase {N}
- 目标：{一句话}
- 影响模块：{模块列表}
- 规范落点：（待定稿时回填）
- 依赖：{前置编号 或 无}
- DoD：☐ 规范定稿 ☐ 实现 ☐ 测试
```

### Step 3：changelog 说明

**登记 Backlog 不写 changelog**（roadmap-rule 硬门禁 3）。仅当本条目促成规范变更、版本递增时，才由 `spec-finalize` skill 按 changelog-rule 记录。

### Step 4：自检

- [ ] 条目格式与 roadmap.md 现有条目一致
- [ ] 优先级排序正确（P0 在前）
- [ ] 无重复条目
- [ ] 未写 changelog（Backlog 阶段）
- [ ] 登记阶段未误改任务状态（保持 Backlog）

## 后续路径（不属于本 skill）

- 任务被选中开始规划 → 状态 `Spec-Drafting`（由 roadmap-rule 管理）
- 规范草稿审查定稿 → 状态 `Spec-Finalized`（激活 **`spec-finalize`** skill）
- 实现落地 → 状态 `Implementing` → `Done`（由 roadmap-rule 管理）