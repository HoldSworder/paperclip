# 通用规则（所有 Agent 共享）

本文件定义公司所有 Agent 的通用行为规范。每个 Agent 的角色 prompt 中通过引用本文件获取这些规则。

## 心跳流程

你在 Paperclip 心跳中运行。每次被唤醒时，必须先按 `$paperclip` skill 中的心跳流程（Heartbeat Procedure）执行：获取身份 → 获取 assignments → checkout Issue → 获取 Issue 详情和评论。这是你获得工作上下文的唯一方式。

## 多项目与多仓库

公司可能管理多个代码仓库，每个仓库对应一个独立的 **Project**，每个 Project 有自己的 **Workspace**（工作目录）。

### 核心概念

| 概念 | 说明 |
|------|------|
| Company | 公司，所有实体的顶层作用域 |
| Project | 项目，对应一个代码仓库 |
| Workspace | 工作空间，Project 的物理路径（cwd）、仓库地址、分支引用 |
| Issue | 任务，通过 `projectId` 绑定到具体 Project |

### 查询可用项目

```
GET /api/companies/{companyId}/projects
```

响应中每个 Project 包含 `workspaces` 数组和 `primaryWorkspace`，其中 `cwd` 为仓库路径。

### Issue 与 Project 的关系

- Issue 的 `projectId` 决定了它属于哪个仓库
- Agent 执行 Issue 时，Paperclip 根据 `projectId` 对应的 Workspace 自动解析工作目录
- 创建 Issue 时**必须指定 `projectId`**，确保 agent 被导向正确的仓库

## Issue 协调规则

### 单仓库需求

同一仓库内的需求在**同一 Issue 中流转**，通过 reassign（重新分配）和评论协调，不创建新 Issue。

reassign 方式：

```
PATCH /api/companies/{companyId}/issues/{issueId}
{
  "assigneeAgentId": "{目标agent-id}"
}
```

### 跨仓库需求

当需求涉及多个仓库时，使用**父子 Issue** 模式：

1. 原始 Issue 作为**父 Issue**（协调中心），记录完整需求
2. 每个需要修改的仓库拆为**子 Issue**，必须设置：
   - `parentId`: 父 Issue id
   - `projectId`: 该仓库对应的 Project id
3. 每个子 Issue 在其 Project 的仓库内独立流转
4. 父 Issue 跟踪整体进度，所有子 Issue 完成后父 Issue 标记 done

创建子 Issue：

```
POST /api/companies/{companyId}/issues
{
  "title": "子任务标题",
  "parentId": "{父issue-id}",
  "projectId": "{目标project-id}",
  "description": "任务描述",
  "priority": "medium"
}
```

### 判断是否跨仓库

接到需求时，按以下步骤判断：

1. 分析需求涉及哪些代码模块
2. 查询 `GET /api/companies/{companyId}/projects` 获取所有 Project 及其 Workspace
3. 确认涉及的模块分别属于哪些 Project
4. 如果涉及 **1 个 Project** → 单 Issue 流转
5. 如果涉及 **多个 Project** → 父子 Issue 模式

## Git 规范

### 分支命名

- 新功能：`feature/MMDD-<slug>`
- Bug 修复：`fix/MMDD-<slug>`

### 多仓库分支

跨仓库需求中，每个仓库独立创建分支，使用相同的 slug 以便关联：

- repo-a: `feature/0312-user-auth`
- repo-b: `feature/0312-user-auth`

### 操作前提

- 创建分支前，先切换到主分支并拉取最新代码
- 每个仓库的主分支名以 Project Workspace 配置为准，默认 `master`

## 代码提交规则

**所有代码提交（git commit / git push）必须经 Board 批准后才可执行。**任何 agent 未经批准严禁自行提交代码。

## 流转协议

所有领域 Agent（CTO、Reviewer、Engineer）完成工作后，统一将 Issue assign 给 **Dispatcher**。Dispatcher 根据评论中的 `[Next Action]` 标记决定下一步路由。

### [Next Action] 标记

每个 Agent 完成工作后，必须在评论末尾添加 `**[Next Action: XXX]**` 标记，供 Dispatcher 识别路由目标：

| 标记 | 含义 | 由谁标记 |
|------|------|---------|
| `SPEC_READY` | 方案已就绪，需审查 | CTO |
| `SPEC_APPROVED` | 方案审查通过 | Reviewer |
| `SPEC_REVISION_NEEDED` | 方案需修改 | Reviewer |
| `CODE_READY` | 编码完成，需审查 | Engineer |
| `CODE_APPROVED` | 代码审查通过 | Reviewer |
| `CODE_FIX_NEEDED` | 代码需修复 | Reviewer |

### 查询 Dispatcher

通过 `GET /api/companies/{companyId}/agents` 查询 `role` 为 `pm` 的 agent，获取其 id。

### 团队角色映射

系统 `role` 是枚举值，不能自定义。团队成员与系统 role 的对应关系：

| Agent | role |
|-------|------|
| CEO | `ceo` |
| Dispatcher | `pm` |
| CTO | `cto` |
| Reviewer | `qa` |
| Engineer | `engineer` |

## 通用评论规范

- 每步操作都要在 Issue 评论中留下进度记录
- 评论中引用其他 Issue 时使用 identifier（如 `PAP-123`）
- 跨仓库场景中，评论要注明当前操作针对哪个仓库/Project
