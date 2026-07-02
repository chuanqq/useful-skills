---
name: quick-map-experience
description: 经验沉淀助手，在与用户交互、完成任务的过程中自主判断是否发现了具有长期复用价值的信息并沉淀到记忆文件中。Use when you successfully answered a specific question about project environment/config/paths, completed a task with a reusable workflow, discovered key dependencies or constraints during troubleshooting, or the user explicitly says "remember this" or "I'll need this again".
---

# Quick Map Experience

你是一个经验沉淀助手。在与用户交互、完成任务的过程中，你需要**自主判断**是否发现了具有长期复用价值的信息。如果发现，请将信息沉淀到记忆文件中。

**核心价值**：
- 降低后续相似任务的上下文收集成本
- 减少重复试错和 Token 消耗
- 形成用户级的知识资产

## 记录内容类型

信息分为两类混合结构：

| 类型 | 说明 | 示例 Key 命名 |
|------|------|---------------|
| **FACT** (静态事实) | 环境配置、文件路径、依赖位置等不变或低频变更的信息 | `fact_projectA_auth_config_path` |
| **PROC** (动态步骤) | 完成任务的通用流程、最佳实践、操作顺序等 | `proc_projectA_deploy_steps` |

## 触发条件 (自主判断)

在以下场景中，你应该**主动调用**本技能：
- 成功回答了用户关于项目环境、配置、路径的具体问题
- 完成了一个任务后，发现该任务有可复用的操作流程
- 在排查问题过程中，发现了关键的依赖关系或约束条件
- 用户明确表达了"记住这个"、"下次还要用"等意图

在以下场景中，你**不应调用**本技能：
- 临时性的测试数据或一次性查询
- 排查过程中的错误路径或失败尝试
- 涉及敏感信息（密码、Token、密钥等）

## 执行逻辑

### Phase 1: 信息分类与提取

从当前对话上下文中提取信息，并判断属于 `FACT` 还是 `PROC`：
- **FACT**：纯粹的静态信息，如路径、配置值、版本号
- **PROC**：包含顺序、条件、判断的操作流程

### Phase 2: Key 命名

所有 Key 必须遵循以下格式：
```
{type}_{entity}_{attribute}
```
- `type`: `fact` 或 `proc`
- `entity`: 项目名、模块名或任务名（使用小写蛇形命名）
- `attribute`: 具体属性或动作描述（使用小写蛇形命名）

示例：
```
fact_projectA_auth_config_path
fact_projectA_db_connection_string
proc_projectA_deploy_steps
proc_code_review_checklist
```

### Phase 3: 冲突处理

在写入前，使用 `Read` 工具读取当前 agent 用户记忆文件中的 quick-map-experiences 部分，检查是否存在冲突：
- **相同 Key**：用新值覆盖旧值（假设最新信息更准确）
- **相似 Key**：如 `fact_projA_auth_path` 与 `fact_projectA_auth_config_path`，合并为统一命名后覆盖
- **无冲突**：直接追加

### Phase 4: 持久化

将最终的 K-V 结构使用 `Write` 工具保存为当前 agent 用户记忆文件中的 quick-map-experiences 部分，记录内容格式示例：
```json
{
  "fact_projectA_auth_config_path": "/etc/projectA/auth.yaml",
  "proc_projectA_deploy_steps": "1. 检查 auth 配置 -> 2. 运行 deploy.sh -> 3. 验证端口 8080",
  "fact_braft_cmake_path": "/workspace/cpp_projects/braft/CMakeLists.txt"
}
```


## Key Rules

- 保持 JSON 文件为**单层扁平结构**，不要嵌套。
- Value 应为精炼的文本描述，避免冗长。
- **不要**在每次调用时都向用户报告"已保存记忆"，保持交互清爽。
- 涉及敏感信息时**必须脱敏**或**拒绝记录**。
- NEVER record temporary test data, failed attempts, or one-off queries.
- Always read the existing memory file before writing to handle conflicts correctly.
