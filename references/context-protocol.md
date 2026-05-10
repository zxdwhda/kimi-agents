# 上下文文件协议（Context File Protocol）

基于 kimi-cli 源码分析，子代理间**完全隔离**，无法自动共享上下文。本协议定义了通过文件系统传递信息的标准格式。

## 设计原则

1. **主代理是唯一的协调者** — 子代理不直接通信，所有信息流转通过主代理 + 文件系统
2. **文件即契约** — 每个文件有明确的格式、写入方、读取方
3. **Resume 复用优先** — 同一角色的连续任务优先 resume，减少文件读写
4. **最小化传递** — prompt 中只包含当前任务必需的上下文摘要，不要全文复制

## 文件体系

### 项目级文件（由主代理维护）

#### `project-brief.md`
**用途**：项目总览，所有子代理必须读取  
**写入方**：主代理（Step 1 初始化）  
**读取方**：所有子代理  
**更新频率**：需求变更时更新  

```markdown
# Project Brief
## 技术栈
- Frontend: React 18 + TypeScript + Vite + Tailwind CSS
- Backend: FastAPI + Python 3.12 + SQLAlchemy
- Database: PostgreSQL 15
- Deploy: Docker Compose
## 核心功能
1. 用户注册登录（OAuth2 + JWT）
2. ...
## 目录约定
- /frontend/ — 前端代码
- /backend/ — 后端代码
- /database/ — 迁移和种子数据
- /deploy/ — 部署配置
- /.kimi/ — 代理状态与上下文
```

#### `project-context.md`
**用途**：环境常量和配置模板  
**写入方**：主代理（Step 1 初始化）  
**读取方**：所有子代理  
**更新频率**：环境变更时更新  

```markdown
# Project Context
## 环境常量
- Node.js 版本: 20.x
- Python 版本: 3.12
- 前端端口: 5173
- 后端端口: 8000
- 数据库端口: 5432
## 环境变量模板
DATABASE_URL=postgresql://user:pass@localhost:5432/dbname
JWT_SECRET=your-secret-key
FRONTEND_API_BASE=http://localhost:8000
```

#### `project-contracts.md`
**用途**：接口契约，前后端对齐的唯一真相源  
**写入方**：主代理 + 各子代理（修改接口时更新）  
**读取方**：前后端子代理  
**更新频率**：接口变更时立即更新  

```markdown
# Project Contracts

## API 契约
### POST /api/auth/register
- 请求: { "email": string, "password": string }
- 响应: { "success": true, "data": { "user_id": string, "token": string } }
- 错误: 400 (参数校验失败), 409 (邮箱已存在)

### GET /api/users/me
- 请求: Header Authorization: Bearer {token}
- 响应: { "success": true, "data": { "id": string, "email": string, "name": string } }

## 数据库契约
### users 表
- id: UUID PRIMARY KEY
- email: VARCHAR(255) UNIQUE NOT NULL
- password_hash: VARCHAR(255) NOT NULL
- name: VARCHAR(100)
- created_at: TIMESTAMP DEFAULT NOW()
- updated_at: TIMESTAMP DEFAULT NOW()
```

### 运行时状态文件（由主代理维护）

#### `agent-state/todo-board.md`
**用途**：统一待办看板（因为子代理没有 SetTodoList）  
**格式**：Markdown 任务列表  

```markdown
# Todo Board

## Phase 1: 基础设施（并行）
- [x] database:schema — 数据库 Schema 设计
- [x] backend:scaffold — 后端脚手架
- [x] frontend:scaffold — 前端脚手架
- [ ] deploy:config — 部署配置

## Phase 2: 核心功能（依赖 Phase 1）
- [ ] backend:api — 业务 API 实现
- [ ] frontend:pages — 前端页面

## Phase 3: 集成（依赖 Phase 2）
- [ ] integration:test — 联调测试
```

#### `agent-state/phase-status.json`
**用途**：机器可读的阶段状态  
**格式**：JSON  

```json
{
  "current_phase": 2,
  "total_phases": 3,
  "tasks": [
    { "id": "database:schema", "status": "completed", "agent_id": "a1b2c3d4" },
    { "id": "backend:api", "status": "running", "agent_id": "a5b6c7d8" },
    { "id": "frontend:pages", "status": "pending", "agent_id": null }
  ],
  "blocked_by": [],
  "last_updated": "2026-05-09T10:15:00Z"
}
```

#### `agent-state/sessions.json`
**用途**：子代理会话记录，用于 resume  
**格式**：JSON  

```json
{
  "agents": [
    {
      "agent_id": "a1b2c3d4",
      "role": "database",
      "task": "schema-design",
      "status": "completed",
      "created_at": "2026-05-09T10:00:00Z",
      "last_used": "2026-05-09T10:15:00Z",
      "can_resume": true
    }
  ]
}
```

### 子代理产出文件（由子代理写入，主代理读取）

#### `agent-results/{role}.md`
**用途**：子代理的结构化产出摘要  
**写入方**：子代理（完成后）  
**读取方**：主代理 + 后续子代理  

命名约定：
- `database.md` — 数据库子代理
- `backend.md` — 后端子代理
- `frontend.md` — 前端子代理
- `integration.md` — 集成子代理

```markdown
# Database Agent Results

## 任务: schema-design
## 状态: 完成
## 交付物
- /database/schema.sql — 完整 CREATE TABLE 语句
- /database/migrations/001_init.sql — 初始迁移
- /database/seed.sql — 种子数据
- /database/README.md — 表关系说明

## 关键决策
- 使用 UUID 作为主键
- 密码使用 bcrypt 哈希
- 枚举类型使用 CHECK 约束

## 接口变更
- 新增 users 表（见 project-contracts.md）
- 无变更

## 步数消耗
约 35 步

## 阻塞项
无
```

## 协议流程

### 启动子代理的标准流程

```
主代理
  │
  ├─ 1. 检查 project-brief.md / project-context.md 是否存在
  ├─ 2. 读取前置子代理的 agent-results/{role}.md
  ├─ 3. 将必要上下文摘要写入 prompt
  ├─ 4. 在 prompt 中明确指令子代理读取哪些文件
  ├─ 5. 启动 Agent(subagent_type="coder", prompt="...", run_in_background=true)
  │
  ▼
子代理
  │
  ├─ 1. 按 prompt 指令读取 project-context.md / project-contracts.md
  ├─ 2. 执行任务（读/写文件、执行 Shell）
  ├─ 3. 将产出写入 agent-results/{role}.md
  ├─ 4. 如修改接口，更新 project-contracts.md
  ├─ 5. 输出 ---SUMMARY--- 格式的摘要
  │
  ▼
主代理
  │
  ├─ 1. 读取子代理的 SUMMARY（前台）或 TaskOutput（后台）
  ├─ 2. 读取 agent-results/{role}.md 确认产出
  ├─ 3. 更新 todo-board.md 和 phase-status.json
  ├─ 4. 如需要，启动下一批子代理
```

### Resume 流程（同一角色连续任务）

```
主代理
  │
  ├─ 1. 从 sessions.json 找到之前的 agent_id
  ├─ 2. 检查该 agent 状态是否为 completed / idle（可 resume）
  ├─ 3. 启动 Agent(subagent_type="coder", resume="{agent_id}", prompt="新任务...")
  │
  ▼
子代理（复用之前的 Context 历史）
  │
  ├─ 1. 自动恢复之前的对话历史
  ├─ 2. 只需在 prompt 中传递新增/变更的上下文
  ├─ 3. 执行任务
  │
  ▼
主代理
  ├─ 读取产出，更新状态
```

**Resume 的优势**：
- 子代理保留之前的工作记忆（Context 历史）
- prompt 中只需传递增量信息，不用重复全部上下文
- 特别适合"设计→修正→完善"这类连续迭代任务

**Resume 的限制**：
- 只能 resume 同一 subagent_type（coder resume coder）
- 被 resume 的子代理必须处于非运行状态
- 后台任务完成后子代理状态变为 idle，可以 resume

## 文件读写工具指令

子代理在 prompt 中被要求执行文件读写时，使用以下工具：

```
# 读取上下文文件
ReadFile(path="{{PROJECT_ROOT}}/.kimi/project-context.md")
ReadFile(path="{{PROJECT_ROOT}}/.kimi/project-contracts.md")

# 写入产出
WriteFile(path="{{PROJECT_ROOT}}/.kimi/agent-results/database.md", content="...")

# 更新契约
ReadFile(path="{{PROJECT_ROOT}}/.kimi/project-contracts.md")
# 修改后
WriteFile(path="{{PROJECT_ROOT}}/.kimi/project-contracts.md", content="...")
```

## 最小化上下文传递原则

**不要**在 prompt 中全文复制大文件内容。应该：

1. **摘要传递**：只提取当前任务需要的信息
   - ❌ 不好："Schema 如下：[粘贴 200 行 SQL]"
   - ✅ 好："已设计 users 表（id UUID, email VARCHAR, password_hash VARCHAR），详见 /database/schema.sql"

2. **引用传递**：告诉子代理文件位置，让它自己读取
   - ❌ 不好："API 文档如下：[粘贴完整 API.md]"
   - ✅ 好："API 契约已更新到 /.kimi/project-contracts.md，请读取并遵循"

3. **差异传递**：只传递变更部分
   - ❌ 不好：每次重新发送全部上下文
   - ✅ 好："相比上一轮，主要变更是：users 表新增了 avatar 字段"
