# 典型工作流模式

本文件提供常见项目类型的标准任务拆解和并行策略。总统领应根据项目类型选择对应模式，复制到 `SetTodoList` 中并调整。

## Resume 策略（跨模式通用）

以下场景优先使用 `resume` 而非创建新子代理：

1. **同一角色的连续迭代**：如"数据库设计"→"数据库设计修正"→"添加索引优化"
   - 使用 `resume="{agent_id}"` 复用同一子代理的 Context
   - 优点：保留之前的设计记忆，prompt 只需传递增量信息
   - 限制：只能 resume 同一 `subagent_type`（coder resume coder）

2. **步数耗尽后的续接**：子代理因 `Max steps reached` 中断
   - 先读取该子代理已产出的文件
   - 用 `resume="{agent_id}"` 让同一子代理继续完成剩余工作
   - 在 prompt 中明确"继续上一轮未完成的任务"

3. **修复任务**：发现之前子代理的产出有 bug
   - 如果能找到原 agent_id，优先 resume
   - 如果原 agent_id 不可用或跨会话丢失，创建新子代理

**Resume 不可用的情况**：
- 不同角色之间（数据库子代理不能 resume 后端子代理）
- 子代理仍处于 running 状态
- 跨会话（sessions.json 中没有记录）

---

## 模式 A: 全栈 Web 应用（最常用）

适用场景：SaaS、后台管理系统、内容平台、工具型网站。

### 推荐技术栈（默认）
- Frontend: React 18 + TypeScript + Vite + Tailwind CSS + shadcn/ui
- Backend: FastAPI (Python) / Express (Node) / Gin (Go)
- Database: PostgreSQL
- Deploy: Docker Compose

### 任务拆解与依赖图

```
Phase 1（最多 4 个并行，全部后台）
├── database:schema     → 产出: schema.sql, README.md
├── backend:scaffold    → 产出: main.py, 路由框架
├── frontend:scaffold   → 产出: App.tsx, 路由配置
└── deploy:config       → 产出: docker-compose.yml, Dockerfile

Phase 2（依赖 Phase 1，最多 2 个并行）
├── backend:api         → 依赖 database:schema（读取 schema 和契约）
│   → 产出: routers/, models.py, API.md
└── frontend:types      → 依赖 backend:api 的接口草稿（轻量，可与 backend:api 并行启动）
│   → 产出: src/types/api.ts

Phase 3（依赖 Phase 2，最多 2 个并行）
├── frontend:pages      → 依赖 backend:api 完整文档 + frontend:types
│   → 产出: pages/, components/, api/client.ts
└── backend:tests       → 依赖 backend:api
    → 产出: tests/

Phase 4（依赖 Phase 3，串行）
└── integration:test    → 依赖 backend:api + frontend:pages
    → 产出: 修复记录, 验证通过
```

### 启动顺序与并发控制

```
# Step 1: Phase 1 全部后台并行（4 个任务 = 系统上限）
Agent(coder, "database:schema", background=true)
Agent(coder, "backend:scaffold", background=true)
Agent(coder, "frontend:scaffold", background=true)
Agent(coder, "deploy:config", background=true)

# Step 2: 等 database:schema 完成后
# 检查: TaskList 确认 database:schema 状态为 completed
# 读取: .kimi/agent-results/database.md 和 database/schema.sql

# Step 3: Phase 2 启动（2 个并行）
Agent(coder, "backend:api", background=true)
Agent(coder, "frontend:types", background=true)

# Step 4: 等 backend:api 完成后
# 检查: TaskList 确认 backend:api 状态
# 读取: .kimi/agent-results/backend.md 和 backend/API.md
# 更新: .kimi/project-contracts.md 中的 API 契约

# Step 5: Phase 3 启动（2 个并行）
Agent(coder, "frontend:pages", background=true)
Agent(coder, "backend:tests", background=true)

# Step 6: 等全部完成后，Phase 4 串行（前台，方便快速修 bug）
Agent(coder, "integration:test", background=false)
```

### Resume 机会

- `database:schema` 完成后如需调整 → resume 同一 agent_id
- `backend:api` 完成后如需补充接口 → resume 同一 agent_id
- `frontend:pages` 完成后如需调整页面 → resume 同一 agent_id

---

## 模式 B: 纯前端展示站 / Landing Page

适用场景：企业官网、活动页、个人作品集、营销落地页。

### 推荐技术栈
- Astro / Next.js / Vite + React
- Tailwind CSS
- 如需 CMS：Strapi / Sanity（可选后端子代理）

### 任务拆解

```
Phase 1（最多 2 个并行）
├── frontend:design     → 产出: 设计文档（配色、布局、文案）
└── frontend:assets     → 产出: 图片、图标、字体

Phase 2（依赖 Phase 1）
├── frontend:components → 产出: 复用组件库
└── frontend:pages      → 产出: 页面实现

Phase 3（串行）
└── deploy:config       → 产出: Vercel/Netlify 配置
```

### 策略

- 通常不需要后端和数据库子代理
- 如果用户说"要 CMS"，增加一个 backend:headless 子代理（Phase 1 并行）
- 部署优先选 Vercel（最简单）
- 由于任务少，不容易达到 4 个并发上限

---

## 模式 C: API + 爬虫 / 数据处理服务

适用场景：数据采集平台、定时任务服务、ETL 管道、内部 API 网关。

### 推荐技术栈
- Backend: FastAPI / Flask / Go
- Database: PostgreSQL / SQLite（轻量）
- Scheduler: Celery / cron / GitHub Actions
- Deploy: Docker / 云函数

### 任务拆解

```
Phase 1（最多 3 个并行）
├── database:schema
├── backend:scaffold
└── backend:design      → 产出: 爬虫/处理器模块设计文档

Phase 2（依赖 Phase 1，最多 2 个并行）
├── backend:api         → 核心处理逻辑 + API
└── backend:scheduler   → 定时任务配置

Phase 3（串行或并行）
├── backend:tests       → 重点测试边界条件和异常处理
└── deploy:config
```

### 策略

- 无前端，减少一个子代理
- 重点检查：异常处理、重试机制、数据去重、限速
- 如有大量数据，提醒用户是否需要分页或流式处理
- 爬虫任务可能超过 15 分钟超时，考虑拆分或 Shell 方案

---

## 模式 D: 小程序 / H5 活动页

适用场景：微信小程序、支付宝小程序、抖音小程序、限时 H5 活动。

### 推荐技术栈
- 微信小程序原生 / Taro / uni-app
- 后端同模式 A（如需登录、支付）

### 任务拆解

```
Phase 1（最多 3 个并行）
├── frontend:scaffold   → 小程序框架搭建
├── backend:scaffold    → 如需后端
└── database:schema     → 如需后端

Phase 2（依赖 Phase 1，最多 3 个并行）
├── frontend:pages      → 页面 + 组件
├── backend:api         → 登录、支付、数据接口
└── frontend:api        → 封装微信 API：登录、扫码、支付

Phase 3（串行）
└── integration:test + deploy:config
```

### 策略

- 小程序有严格的包体积限制（2MB），提醒前端子代理注意
- 支付和登录流程必须前后端对齐，集成阶段多花时间
- 微信 API 调用需要特殊处理，建议前端子代理专门封装

---

## 模式 E: AI 应用（Chatbot、Agent、Copilot）

适用场景：基于 LLM 的聊天应用、AI 助手、文档分析工具、代码生成器。

### 推荐技术栈
- Frontend: React + TypeScript（流式输出 UI：SSE / WebSocket）
- Backend: FastAPI + LangChain / 纯 API 封装
- AI: OpenAI API / Claude API / 自部署模型
- Deploy: Docker / Vercel + 云服务器

### 任务拆解

```
Phase 1（最多 3 个并行）
├── backend:llm-client  → 封装流式 API 调用
├── frontend:scaffold
└── frontend:chat-ui    → 消息列表、输入框、流式渲染

Phase 2（依赖 Phase 1，最多 2 个并行）
├── backend:api         → 对话历史、上下文管理、Prompt 模板
└── frontend:integration → 对接后端 SSE

Phase 3（串行或并行）
├── backend:advanced    → RAG、工具调用、多轮记忆
└── integration:test
```

### 策略

- 流式输出是核心体验，必须前后端对齐 SSE / WebSocket 协议
- 上下文管理（历史消息截断策略）是后端重点
- 如有文件上传，增加文件解析子代理（PDF/图片 OCR）
- LLM API 调用可能触发 rate limit，注意控制并发

---

## 并发控制速查表

| Phase | 典型任务数 | 是否触及 4 并发上限 | 建议 |
|---|---|---|---|
| 模式 A Phase 1 | 4 | ✅ 触及上限 | 全部后台，同时启动 |
| 模式 A Phase 2 | 2 | ❌ | 后台并行 |
| 模式 A Phase 3 | 2 | ❌ | 后台并行 |
| 模式 B Phase 1 | 2 | ❌ | 后台并行 |
| 模式 C Phase 1 | 3 | ❌ | 后台并行 |
| 模式 D Phase 1 | 3 | ❌ | 后台并行 |
| 模式 E Phase 1 | 3 | ❌ | 后台并行 |

**通用规则**：
- 触及 4 并发上限时，全部后台运行，主代理立即进入跟进状态
- 未触及时，前台/后台均可，集成阶段建议前台
- 任何时候出现 429/401，立即停止新任务启动

---

## 通用风险 checklist

无论哪种模式，总统领在验收前必须确认：

- [ ] 各子代理是否遵守了"只修改自己的工作目录"约束？
- [ ] 前后端接口契约是否对齐（URL、方法、参数名、返回字段）？
- [ ] 环境变量是否提取到了 `.env.example`？
- [ ] 是否有 README 告诉用户怎么启动？
- [ ] 数据库密码/密钥是否硬编码在代码里？
- [ ] 错误处理是否统一（前端不会裸暴露后端 500 堆栈）？
- [ ] 各子代理是否按协议写入了 `.kimi/agent-results/{role}.md`？
- [ ] `project-contracts.md` 是否反映了最新的接口状态？
