# 示例：用户管理 CRUD 应用

这是一个完整的全栈 CRUD 应用开发示例，展示 Kimi Agents Skill 的实际工作流程。

## 需求

> "帮我建一个用户管理后台，能增删改查用户，React 前端 + FastAPI 后端 + PostgreSQL"

## 开发过程

### Phase 0: 需求对齐（主 Agent 完成）

主 Agent 跟用户确认后，初始化项目：

```bash
mkdir -p frontend/src/{pages,components,api,types,hooks}
mkdir -p backend/{routers,models,tests,utils}
mkdir -p database/migrations
mkdir -p deploy
mkdir -p .kimi/{agent-state,agent-results}
```

写入 `.kimi/project-brief.md`：

```markdown
# Project Brief
## 技术栈
- Frontend: React 18 + TypeScript + Vite + Tailwind CSS
- Backend: FastAPI + Python 3.12 + SQLAlchemy
- Database: PostgreSQL 15
- Deploy: Docker Compose
## 核心功能
1. 用户列表（分页、搜索）
2. 新增用户
3. 编辑用户
4. 删除用户
5. 用户详情页
## 目录约定
- /frontend/ — 前端代码
- /backend/ — 后端代码
- /database/ — 迁移和种子数据
- /deploy/ — 部署配置
```

写入 `.kimi/project-context.md`：

```markdown
# Project Context
## 环境常量
- Node.js 版本: 20.x
- Python 版本: 3.12
- 前端端口: 5173
- 后端端口: 8000
- 数据库端口: 5432
## 环境变量模板
DATABASE_URL=postgresql://user:pass@localhost:5432/userdb
JWT_SECRET=change-me-in-production
FRONTEND_API_BASE=http://localhost:8000
```

### Phase 1: 基础设施并行（4 个后台子代理）

#### 子代理 1: database:schema

```python
Agent(
  subagent_type="coder",
  description="设计用户管理数据库 Schema",
  prompt="""
你是数据库架构师。请为"用户管理后台"设计数据库 Schema。

## 前置动作（开始工作前必须执行）
1. 读取 {{PROJECT_ROOT}}/.kimi/project-context.md

## 上下文
- 项目根目录: /workspace/user-management
- 工作目录: /workspace/user-management/database
- 技术栈: PostgreSQL 15
- 功能: 用户增删改查，字段包括 id, name, email, avatar, role, status, created_at, updated_at

## 交付物
1. /workspace/user-management/database/schema.sql — CREATE TABLE 语句
2. /workspace/user-management/database/migrations/001_init.sql — 迁移文件
3. /workspace/user-management/database/seed.sql — 10 条测试数据
4. /workspace/user-management/database/README.md — 字段说明

## 设计规范
- 主键用 UUID
- 字段 snake_case
- 每张表 created_at + updated_at
- email 加唯一索引

## 后置动作（完成后必须执行）
1. 将产出摘要写入 /workspace/user-management/.kimi/agent-results/database.md
2. 将表结构更新到 /workspace/user-management/.kimi/project-contracts.md
  """,
  run_in_background=true
)
```

#### 子代理 2: backend:scaffold

```python
Agent(
  subagent_type="coder",
  description="搭建 FastAPI 后端脚手架",
  prompt="""
你是后端工程师。请搭建 FastAPI 项目脚手架。

## 前置动作（开始工作前必须执行）
1. 读取 {{PROJECT_ROOT}}/.kimi/project-context.md

## 上下文
- 项目根目录: /workspace/user-management
- 工作目录: /workspace/user-management/backend
- 技术栈: FastAPI + Python 3.12 + SQLAlchemy + asyncpg

## 交付物
1. /workspace/user-management/backend/main.py — 应用入口（含 CORS、异常处理）
2. /workspace/user-management/backend/config.py — 配置管理
3. /workspace/user-management/backend/database.py — 数据库连接
4. /workspace/user-management/backend/requirements.txt

## 后置动作
1. 将产出摘要写入 /workspace/user-management/.kimi/agent-results/backend.md
  """,
  run_in_background=true
)
```

#### 子代理 3: frontend:scaffold

```python
Agent(
  subagent_type="coder",
  description="搭建 React 前端脚手架",
  prompt="""
你是前端工程师。请搭建 React + Vite 项目脚手架。

## 上下文
- 项目根目录: /workspace/user-management
- 工作目录: /workspace/user-management/frontend
- 技术栈: React 18 + TypeScript + Vite + Tailwind CSS

## 交付物
1. /workspace/user-management/frontend/package.json
2. /workspace/user-management/frontend/vite.config.ts
3. /workspace/user-management/frontend/src/main.tsx
4. /workspace/user-management/frontend/src/App.tsx
5. /workspace/user-management/frontend/tailwind.config.js
6. /workspace/user-management/frontend/index.html

## 后置动作
1. 将产出摘要写入 /workspace/user-management/.kimi/agent-results/frontend.md
  """,
  run_in_background=true
)
```

#### 子代理 4: deploy:config

```python
Agent(
  subagent_type="coder",
  description="准备 Docker 部署配置",
  prompt="""
你是 DevOps 工程师。请准备 Docker Compose 部署配置。

## 上下文
- 项目根目录: /workspace/user-management
- 技术栈: React + FastAPI + PostgreSQL
- 前端端口: 5173, 后端端口: 8000, 数据库端口: 5432

## 交付物
1. /workspace/user-management/docker-compose.yml
2. /workspace/user-management/Dockerfile.backend
3. /workspace/user-management/Dockerfile.frontend
4. /workspace/user-management/.env.example

## 后置动作
1. 将产出摘要写入 /workspace/user-management/.kimi/agent-results/integration.md
  """,
  run_in_background=true
)
```

### 进度跟进（Phase 1 进行中）

主 Agent 调用 `TaskList` 监控：

```
📊 Phase 1 进度: 2/4 完成

✅ 已完成
- database:schema — Schema 已输出到 /database/schema.sql
- backend:scaffold — FastAPI 脚手架已搭建

⏳ 进行中
- frontend:scaffold — 子代理进程存活，正在配置 Tailwind
- deploy:config — 正在写 docker-compose.yml
```

### Phase 2: 核心功能（依赖 Phase 1）

等 database:schema 完成后，启动：

#### 子代理 5: backend:api

```python
Agent(
  subagent_type="coder",
  description="实现用户管理 REST API",
  prompt="""
你是后端工程师。请实现用户管理的完整 REST API。

## 前置动作（开始工作前必须执行）
1. 读取 /workspace/user-management/.kimi/project-context.md
2. 读取 /workspace/user-management/.kimi/project-contracts.md
3. 读取 /workspace/user-management/database/schema.sql
4. 读取 /workspace/user-management/.kimi/agent-results/database.md

## 上下文
- 已有脚手架: /workspace/user-management/backend/main.py
- 数据库 Schema: users 表（id UUID, name VARCHAR, email VARCHAR, avatar VARCHAR, role VARCHAR, status VARCHAR, created_at TIMESTAMP, updated_at TIMESTAMP）

## 交付物
1. /workspace/user-management/backend/routers/users.py — 用户 CRUD 路由
   - GET /api/users — 列表（支持分页、搜索）
   - GET /api/users/{id} — 详情
   - POST /api/users — 创建
   - PUT /api/users/{id} — 更新
   - DELETE /api/users/{id} — 删除
2. /workspace/user-management/backend/models.py — SQLAlchemy 模型
3. /workspace/user-management/backend/schemas.py — Pydantic 模型
4. /workspace/user-management/backend/API.md — 完整 API 文档

## 接口规范
- 响应统一包装: { "success": true, "data": ..., "error": null }
- 输入校验用 Pydantic

## 后置动作
1. 将产出摘要写入 /workspace/user-management/.kimi/agent-results/backend.md
2. 将 API 文档更新到 /workspace/user-management/.kimi/project-contracts.md
  """,
  run_in_background=true
)
```

#### 子代理 6: frontend:types

```python
Agent(
  subagent_type="coder",
  description="生成前端 TypeScript 类型定义",
  prompt="""
你是前端工程师。请生成 TypeScript 类型定义。

## 前置动作
1. 读取 /workspace/user-management/.kimi/project-contracts.md

## 上下文
- 用户字段: id, name, email, avatar, role, status, created_at, updated_at
- API: GET/POST/PUT/DELETE /api/users

## 交付物
1. /workspace/user-management/frontend/src/types/user.ts
2. /workspace/user-management/frontend/src/types/api.ts
  """,
  run_in_background=true
)
```

### Phase 3: 前端页面（依赖 Phase 2）

等 backend:api 完成后：

#### 子代理 7: frontend:pages

```python
Agent(
  subagent_type="coder",
  description="实现用户管理前端页面",
  prompt="""
你是前端工程师。请实现用户管理的前端页面。

## 前置动作
1. 读取 /workspace/user-management/.kimi/project-context.md
2. 读取 /workspace/user-management/.kimi/project-contracts.md
3. 读取 /workspace/user-management/.kimi/agent-results/backend.md
4. 读取 /workspace/user-management/frontend/src/types/user.ts

## 上下文
- API 基础地址: http://localhost:8000
- 已有类型定义: src/types/user.ts
- 技术栈: React + TypeScript + Tailwind CSS

## 交付物
1. /workspace/user-management/frontend/src/api/client.ts — API 封装
2. /workspace/user-management/frontend/src/pages/UserList.tsx — 用户列表（分页、搜索）
3. /workspace/user-management/frontend/src/pages/UserForm.tsx — 新增/编辑表单
4. /workspace/user-management/frontend/src/pages/UserDetail.tsx — 用户详情
5. /workspace/user-management/frontend/src/components/ConfirmDialog.tsx — 删除确认弹窗
6. /workspace/user-management/frontend/src/App.tsx — 路由配置

## 后置动作
1. 将产出摘要写入 /workspace/user-management/.kimi/agent-results/frontend.md
  """,
  run_in_background=true
)
```

### Phase 4: 集成验收

所有模块完成后，主 Agent 执行：

1. **接口对齐检查**：
   - 读取 `backend/API.md` 提取所有路由
   - 读取 `frontend/src/api/client.ts` 提取所有调用
   - 对比：URL、方法、参数名、返回字段

2. **如发现问题**，启动集成修复子代理（前台模式）：

```python
Agent(
  subagent_type="coder",
  description="修复前后端接口不对齐",
  prompt="""
你是集成工程师。请修复以下接口不对齐问题：

1. 前端调用 POST /api/user（单数），但后端路由是 POST /api/users（复数）
2. 前端传递 { name, email }，但后端要求 { username, email }

## 前置动作
1. 读取 /workspace/user-management/.kimi/project-contracts.md
2. 读取 /workspace/user-management/backend/routers/users.py
3. 读取 /workspace/user-management/frontend/src/api/client.ts

## 交付物
1. 修复后的代码
2. /workspace/user-management/.kimi/integration-gaps.md — 修复记录
  """,
  run_in_background=false  # 前台模式，方便快速迭代
)
```

3. **运行验证**：
   ```bash
   docker compose up -d
   ```

4. **最终交付摘要**：

```markdown
🎉 项目交付完成

## 项目结构
user-management/
├── frontend/          # React + Vite + Tailwind
├── backend/           # FastAPI + SQLAlchemy
├── database/          # PostgreSQL Schema + 迁移
├── deploy/            # Docker Compose
└── .kimi/             # 代理状态与上下文

## 启动命令
cd user-management && docker compose up -d

## 访问地址
- 前端: http://localhost:5173
- 后端 API: http://localhost:8000
- API 文档: http://localhost:8000/docs

## 已知问题
- 无
```

## 学习要点

1. **文件协议**：每个子代理都遵循"读取 → 工作 → 写入 → 报告"的标准流程
2. **Resume 机会**：如果 backend:api 完成后需要补充接口，可以 resume 同一 agent_id
3. **错误处理**：如果某个子代理因步数耗尽失败，主 Agent 会检测并拆分重试
4. **并行控制**：Phase 1 的 4 个任务刚好触及系统上限，Phase 2 的 2 个任务安全并行
