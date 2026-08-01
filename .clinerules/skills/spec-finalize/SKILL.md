---
name: spec-finalize
description: Finalize a spec draft into a released version. Use when a spec section is ready for review and release, or when transitioning a roadmap item from Spec-Drafting to Spec-Finalized. Trigger on phrases like "定稿", "finalize spec", "规范审查", "spec review".
---

# 规范定稿流程

将 Spec-Drafting 状态的规范草稿审查为正式版本，更新版本号、changelog、路线清单状态。
**本 skill 是 "Spec-Drafting → Spec-Finalized" 的唯一入口。**

## 前置条件

1. 路线清单（roadmap.md）中存在对应条目，状态为 `Spec-Drafting`
2. 规范草稿已写入对应章节（主规范或子规范）
3. 若该条目有依赖，其依赖任务须已 `Spec-Finalized`（roadmap-rule 状态机）

## 步骤

### Step 1：引用约束审查

基于主规范的结构化章节做通用检查：

- 新依赖：对照主规范「十一、引用约束总表」——新增引用是否在允许范围内
- 新模块：对照主规范「二、完整模块树」——是否遵循现有模块结构
- 触禁依赖：对照主规范「十二、禁止依赖链检查」——是否存在禁止引入的依赖
- 公开接口：是否最小化，避免越层调用

> 参考：既有专项决策（如 v6.9 双链过渡的"无分配、无锁间接"、热重载的"锁外解析、锁内交换、RT 零析构"）作为**示例**，不作为所有草稿的硬性检查项——新功能按自身特性对号入座。

### Step 2：RT 安全审查

检查规范草稿中涉及实时线程的部分：

- RT 路径内是否有堆分配（Box、Vec push、String 创建）
- RT 路径内是否有 I/O（文件、网络、日志宏）
- RT 路径内是否有 Mutex / RwLock（lock 操作）
- 锁持有期间是否有耗时操作（文件 I/O、网络、大量计算）
- 如有违反，修改规范草稿

### Step 3：模块边界审查

检查规范草稿中声明的影响模块：

- 新增模块是否遵循现有 crate 结构
- 跨模块依赖是否合理（无循环依赖、无越层调用）
- 公开接口是否最小化
- 涉及规范仓库之外的 crate（如 vxapo-cli / vxapo-app）时，是否在条目中声明了其边界

### Step 4：递增版本号

按**次版本**递增（本项目惯例）：

- 主规范版本号递增（如 v6.9 → v6.10），首行 `# 模块引用规范文档 vX.Y`
- 如涉及子规范，同步更新对应子规范相关内容

> 仅当出现**破坏性 / 结构重构级变更**时，才考虑 major 递增（如 v6.x → v7.0），并需先在 roadmap 中说明理由。

### Step 5：写 changelog

**严格按 `.clinerules/changelog-rule.md` 的「记录格式」区块**写入，不得自创格式：

```
## vX.Y — YYYY-MM-DD

变更类型：`结构重构` / `缺陷修复` / `实现对齐` / `外部借鉴` / `文档同步`

- **条目**：一句话描述——**对应章节**：`模块 节号`
```

- 每条变更必须标注关键章节号（精确到 `文件名 + 节号`）
- 变更类型与描述必须与「关键设计决策摘要补记」表一致

### Step 6：更新路线清单状态 + 一致性校验

1. 更新 roadmap.md 对应条目：
   - 状态：`Spec-Drafting` → `Spec-Finalized`
   - 规范落点：回填章节号
   - DoD：勾选"规范定稿"
2. 执行**三文件一致性校验**（roadmap-rule 联动要求）：
   - `changelog.md` 版本号 == 主规范版本号 == roadmap 条目规范落点引用版本
   - 三者缺一不可

### Step 7：无法合规的处理

若 Step 1-3 中发现规范草稿无法完全合规：

- **非核心违反**（如公开接口可再收敛、RT 路径可优化）：修改草稿使其合规
- **核心依赖链违反 / RT 安全无法保证**：**停留在 `Spec-Drafting`**，不进入 Spec-Finalized；
  在条目中记录"已知限制"并交由 roadmap 决策（延期 / 调整范围 / 拆分子任务）
- 已记录的已知限制须在 DoD 中显式声明，不得静默放行

### Step 8：自检

- [ ] 引用约束总表无新增违反
- [ ] RT 路径无分配、无 I/O、无锁间接
- [ ] 模块边界合理
- [ ] 版本号已递增（次版本，首行同步）
- [ ] changelog 已按 changelog-rule 记录格式更新
- [ ] 三文件一致性校验通过（changelog 版本 == 主规范版本 == roadmap 落点版本）
- [ ] 路线清单状态已更新为 `Spec-Finalized`
- [ ] DoD 第一项已勾选
- [ ] 未合规项已显式记录（如有）

## 后续路径（不属于本 skill）

- 执行端按规范落地 → 状态 `Implementing`（由 roadmap-rule 硬门禁放行，仅 Spec-Finalized 可进入）
- 实现完成 → 规范侧合规核对 → 状态 `Done` 并归档至 roadmap.md「已完成」区（由 roadmap-rule 管理）