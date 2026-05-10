# 🏛️ Kimi Agents — 全栈项目总统领 Skill

> 让 Kimi Code CLI 化身项目架构师，自动拆解任务、并行委派给专业子代理，并持续跟进进度。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Kimi Code CLI](https://img.shields.io/badge/Kimi%20Code%20CLI-1.41.0+-blue)](https://www.kimi.com/code)

## 这是什么？

**Kimi Agents** 是一个 [Kimi Code CLI](https://www.kimi.com/code) 的 **User Skill**，让 AI 在开发完整项目时扮演"总统领"角色：

```
你（用户） ←→ 总统领（主 Agent） ←→ 专业子代理（coder × N）
              拆解任务              并行执行
              跟进进度              输出产出
              验收交付              报告摘要
```

**解决的核心问题**：

- ❌ 单代理做全栈项目 → 步数爆掉、上下文溢出、效率低下
- ✅ 总统领 + 专业子代理并行 → 效率提升 3-4 倍，每个模块有专人负责

**基于真实源码设计**：本 Skill 的架构严格遵循 [kimi-cli](https://github.com/moonshot-ai/kimi-cli) 源码中的子代理/后台任务系统设计，不做超出产品能力边界的假设。

---

## 安装

### 方法一：直接复制（推荐）

```bash
# 克隆本仓库
git clone https://github.com/your-username/kimi-agents.git

# 复制到 Kimi Code CLI 的 skills 目录
cp -r kimi-agents ~/.kimi/skills/agents
```

### 方法二：手动创建

在 `~/.kimi/skills/` 目录下创建 `agents/` 文件夹，将本仓库的 `SKILL.md` 和 `references/` 目录复制进去：

```
~/.kimi/skills/agents/
├── SKILL.md
└── references/
    ├── delegation-prompts.md
    ├── workflow-patterns.md
    ├── context-protocol.md
    └── shell-orchestration.md
```

### 验证安装

在 Kimi Code CLI 中输入任意全栈开发需求，如：

> "帮我搭建一个用户管理的全栈应用，React + FastAPI + PostgreSQL"

如果 Kimi 自动进入"总统领模式"（开始拆解任务、启动子代理），说明 Skill 已生效。

---

## 快速开始

### 场景：搭建一个 Todo 应用

**你只需说一句话**：

> "帮我建一个 Todo 应用，React 前端 + FastAPI 后端 + PostgreSQL，能增删改查任务"

**Kimi 会自动执行**：

```
Phase 0: 需求对齐
  └── 跟你确认技术栈、功能细节

Phase 1: 并行启动 4 个后台子代理（达到系统上限）
  ├── [database]  设计 Schema
  ├── [backend]   搭建脚手架
  ├── [frontend]  搭建脚手架
  └── [deploy]    准备 Docker 配置

Phase 2: 等 database 完成后，并行启动 2 个
  ├── [backend]   实现 API（读取 database 产出的 schema）
  └── [frontend]  生成 TypeScript 类型

Phase 3: 等 backend 完成后，启动前端页面
  └── [frontend]  实现页面和组件（读取 API 契约）

Phase 4: 集成验收
  └── 检查前后端接口是否对齐，修复不一致
```

**整个过程你只需**：
1. 确认技术栈选择（一次）
2. 偶尔问"进度如何"（Kimi 自动汇报）
3. 最后验收成果

---

## 核心概念

### 1. 总统领（主 Agent）

- **职责**：拆解任务、委派子代理、跟进进度、验收交付
- **不做的**：写超过 10 行的代码（配置拼接除外）
- **必须做的**：维护上下文文件、检查接口契约

### 2. 子代理（subagent_type="coder"）

系统内置只有三种子代理类型：`coder`、`explore`、`plan`。本 Skill 统一使用 `coder`，通过 **prompt 赋予不同角色**：

| 角色 | 工作目录 | 典型任务 |
|---|---|---|
| 数据库架构师 | `database/` | Schema 设计、迁移、种子数据 |
| 后端工程师 | `backend/` | API 实现、路由、模型、测试 |
| 前端工程师 | `frontend/` | 页面、组件、API 封装 |
| DevOps 工程师 | `deploy/` | Docker、CI/CD、部署配置 |
| 集成工程师 | 全局 | 接口对齐修复、联调测试 |

### 3. 上下文文件协议（解决"手动复制粘贴"）

由于子代理间**完全隔离**（各自独立的 Context），所有信息共享必须通过文件系统：

```
.kimi/
├── project-brief.md          # 项目总览（技术栈、功能清单）
├── project-context.md        # 环境常量（端口、版本、环境变量）
├── project-contracts.md      # 接口契约（API 路径、参数、数据库字段）
├── agent-state/
│   ├── todo-board.md         # 统一待办看板
│   ├── phase-status.json     # 阶段状态
│   └── sessions.json         # 子代理会话记录（用于 resume）
└── agent-results/
    ├── database.md           # 数据库子代理产出摘要
    ├── backend.md            # 后端子代理产出摘要
    ├── frontend.md           # 前端子代理产出摘要
    └── integration.md        # 集成子代理产出摘要
```

**每个子代理的 prompt 中会自动包含**：
- **前置动作**：读取 `project-context.md` 和 `project-contracts.md`
- **后置动作**：将产出写入 `agent-results/{role}.md`，更新 `project-contracts.md`

### 4. Resume 机制（复用子代理记忆）

如果同一角色需要连续多轮工作（如"数据库设计"→"添加索引优化"），可以 **resume 同一子代理**：

```python
# 第一轮：创建新子代理
Agent(subagent_type="coder", prompt="设计数据库 Schema...")
# → 返回 agent_id: "a1b2c3d4"

# 第二轮：复用同一子代理（保留之前的 Context 历史）
Agent(subagent_type="coder", resume="a1b2c3d4", prompt="在 users 表上添加索引...")
```

**优势**：
- 保留之前的设计记忆，无需重新传递全部上下文
- prompt 只需传递增量信息

### 5. 渐进式并行策略

系统最多允许 **4 个后台子代理**同时运行。不要一上来就全并行：

```
Phase 0（试点）：启动 1 个最简单的任务
  └── 成功 → Phase 1 | 失败（401/429/步数爆掉）→ 解决问题

Phase 1（小规模）：启动 2 个无依赖任务
  └── 观察 2-3 分钟，确认无 rate limit

Phase 2（满负荷）：启动第 3、4 个任务
  └── 触及系统上限 4，任何时候出错立即停止追加
```

---

## 使用示例

### 示例 1：全栈 Web 应用（SaaS）

```markdown
用户：我要做一个在线笔记应用，支持 Markdown 编辑、标签、搜索

Kimi：
  1. 确认技术栈 → React + FastAPI + PostgreSQL + Docker
  2. Phase 1 并行启动：
     - database: 设计 notes、tags、note_tags 表
     - backend: 搭建 FastAPI 脚手架
     - frontend: 搭建 React + Vite 脚手架
     - deploy: 准备 docker-compose.yml
  3. Phase 2 等 database 完成后：
     - backend: 实现 notes CRUD API
     - frontend: 生成 API 类型定义
  4. Phase 3 等 backend 完成后：
     - frontend: 实现编辑器、列表、搜索页面
  5. Phase 4 集成验收：
     - 检查 API 契约对齐
     - 运行 docker compose up 验证
```

### 示例 2：Landing Page（纯前端）

```markdown
用户：帮我做一个公司官网，要好看、响应式

Kimi：
  1. 确认技术栈 → Next.js + Tailwind CSS
  2. Phase 1 并行：
     - frontend: 设计页面结构和配色
     - frontend: 准备图片和图标资源
  3. Phase 2：
     - frontend: 实现首页、关于页、联系页
  4. Phase 3：
     - deploy: Vercel 部署配置
```

### 示例 3：API 服务（无前端）

```markdown
用户：写一个爬虫服务，定时抓取新闻并存到数据库

Kimi：
  1. 确认技术栈 → FastAPI + PostgreSQL + Celery
  2. Phase 1 并行：
     - database: 设计 articles 表
     - backend: 搭建 FastAPI 脚手架
     - backend: 设计爬虫架构
  3. Phase 2：
     - backend: 实现爬虫核心逻辑
     - backend: 实现 API 端点
  4. Phase 3：
     - backend: 单元测试
     - deploy: Docker 部署
```

---

## 架构说明

### 基于 kimi-cli 源码的设计约束

本 Skill 严格遵循 [kimi-cli](https://github.com/moonshot-ai/kimi-cli) 源码中的子代理系统设计，不做超出能力边界的假设：

| 约束项 | 源码位置 | 说明 |
|---|---|---|
| 子代理类型只有 3 种 | `subagents/registry.py` | `coder`/`explore`/`plan`，无法自定义 |
| 子代理上下文完全隔离 | `subagents/core.py:prepare_soul()` | 每个子代理独立的 `Context.restore()` |
| 子代理不能嵌套 | `agents/default/coder.yaml` | toolset 已排除 `Agent` 工具 |
| 子代理没有 SetTodoList | `agents/default/coder.yaml` | 子代理内部 todo 主代理不可见 |
| 后台并发上限 4 | `background/manager.py` | `max_running_tasks = 4` |
| 后台超时 15 分钟 | `config.py` | `agent_task_timeout_s = 900` |
| 步数限制 | `config.py` | `max_steps_per_turn` 默认 1000，某些环境限制 100 |
| Resume 复用 Context | `subagents/core.py` | `context.restore()` 恢复历史对话 |

### 为什么用文件协议？

源码确认：**子代理间没有自动共享机制**。`prepare_soul()` 中每个子代理调用 `context.restore()` 只恢复自己的历史。因此本 Skill 设计了文件协议作为唯一的跨代理通信方式。

### 错误处理矩阵

基于源码中的 `SoulRunFailure` 类型：

| 错误 | 原因 | 处理方案 |
|---|---|---|
| `Max steps reached` | 任务太大，步数耗尽 | 拆分任务，或 resume 续接 |
| `API error (429)` | Rate limit | 停止新任务，等 1-2 分钟改串行 |
| `API error (401/403)` | 认证失败 | 停止所有任务，检查配置 |
| `LLM provider error` | 服务端错误 | 重试最多 3 次 |
| `Agent run error` | 意外错误 | 重新启动 |
| `Empty agent output` | 子代理无输出 | 重新启动 |
| `timed_out` | 超过 15 分钟 | 拆分任务 |

---

## 故障排查

### Q: 子代理说"任务过大，超过安全预算 60 步"

**原因**：当前环境 `max_steps_per_turn` 较低（如 100），子代理预估超过 60 步（留 40% 余量）。

**解决**：让总统领拆分成更小的任务。例如：
- ❌ "实现用户模块的 CRUD" → ✅ "实现 User model" + "实现 /users 路由" + "实现登录中间件"

### Q: 后台子代理状态显示 failed，但不知道原因

**排查步骤**：
1. 调用 `TaskList` 查看状态
2. 调用 `TaskOutput(block=false)` 读取 `output.log`
3. 检查 `failure_reason` 字段
4. 对照上面的错误处理矩阵处理

### Q: 前后端接口对不齐

**预防**：
- 后端子代理完成后必须更新 `project-contracts.md`
- 前端子代理开始前必须读取 `project-contracts.md`

**修复**：
- 启动集成修复子代理（前台模式，方便快速迭代）
- 读取 `.kimi/integration-gaps.md` 查看具体问题

### Q: 并行启动 4 个子代理后，第 5 个失败

**原因**：`max_running_tasks = 4`，系统硬性限制。

**解决**：等待已有任务完成（或部分完成）后再启动新任务。使用 `TaskList` 监控活跃任务数。

---

## 进阶用法

### 自定义子代理 Prompt

参考 `references/delegation-prompts.md` 中的模板，根据项目需求调整：

```python
Agent(
  subagent_type="coder",
  prompt="""
你是 [角色] 专家，负责 [具体任务]。

## 前置动作（开始工作前必须执行）
1. 读取 {{PROJECT_ROOT}}/.kimi/project-context.md
2. 读取 {{PROJECT_ROOT}}/.kimi/project-contracts.md

## 交付物
1. [文件路径]
2. [文件路径]

## 约束
- 只修改自己的工作目录
- 完成后写入 {{PROJECT_ROOT}}/.kimi/agent-results/[role].md
  """,
  run_in_background=true
)
```

### 使用 Shell 方案替代内置 Agent 工具

如果任务可能超过 15 分钟，或需要完全进程隔离，参考 `references/shell-orchestration.md`。

### 添加新的工作流模式

如果项目类型不在 `references/workflow-patterns.md` 的 5 种模式中，可以：
1. 复制现有模式作为模板
2. 调整任务拆解和依赖关系
3. 更新 `SetTodoList` 中的里程碑

---

## 项目结构

```
kimi-agents/
├── README.md                          # 本文档
├── SKILL.md                           # Skill 核心指令（Kimi 实际读取的文件）
├── references/
│   ├── delegation-prompts.md          # 6 类子代理的标准化 prompt 模板
│   ├── workflow-patterns.md           # 5 种典型项目的任务拆解策略
│   ├── context-protocol.md            # 上下文文件协议详细规范
│   └── shell-orchestration.md         # Shell 并行编排方案（备用）
├── examples/
│   └── crud-app/                      # CRUD 应用完整示例
├── LICENSE
└── .gitignore
```

---

## 贡献

欢迎贡献！如果你：

- 发现了更好的子代理 prompt 模板
- 有新的工作流模式想分享
- 发现了 bug 或有改进建议

请提交 Issue 或 PR。

---

## License

MIT License — 详见 [LICENSE](LICENSE) 文件。
