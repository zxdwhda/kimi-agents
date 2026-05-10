# Shell 并行编排方案

当用户明确偏好用终端命令（而非内置 `Agent` 工具）来并行启动多个 kimi 会话时，使用本方案。

**适用场景**：
- 用户说"用终端开多个窗口跑"
- 用户不信任后台子代理机制，想要看到独立的 log 文件
- 需要子代理之间完全进程隔离（避免共享上下文污染）
- 需要超长时任务（超过默认 15 分钟后台超时）

**不适用场景**：
- 需要主代理实时汇总子代理中间结果（Shell 方式的输出需要手动 cat）
- 需要统一审批流（Shell 方式启动的 kimi 实例有独立的审批通道）

---

## 方案 A: 独立会话并行（推荐）

每个模块启动一个独立的 kimi 会话，通过 `--session` 命名，输出重定向到独立日志。

```bash
# 1. 数据库设计（独立会话）
kimi --work-dir ./database --session db-schema "你是数据库架构师，请设计电商项目的 PostgreSQL Schema..." > ./.kimi/logs/db-schema.log 2>&1 &
DB_PID=$!

# 2. 后端脚手架（独立会话）
kimi --work-dir ./backend --session backend-scaffold "你是后端工程师，请用 FastAPI 搭建项目脚手架..." > ./.kimi/logs/backend-scaffold.log 2>&1 &
BE_PID=$!

# 3. 前端脚手架（独立会话）
kimi --work-dir ./frontend --session frontend-scaffold "你是前端工程师，请用 React + Vite 搭建项目..." > ./.kimi/logs/frontend-scaffold.log 2>&1 &
FE_PID=$!

# 4. 保存 PID 供后续查询
echo "DB_PID=$DB_PID" > ./.kimi/pids
echo "BE_PID=$BE_PID" >> ./.kimi/pids
echo "FE_PID=$FE_PID" >> ./.kimi/pids
```

**查询进度**：

```bash
# 查看各模块最新输出
tail -n 30 ./.kimi/logs/db-schema.log
tail -n 30 ./.kimi/logs/backend-scaffold.log

# 检查进程是否存活
ps -p $DB_PID > /dev/null && echo "db-schema 运行中" || echo "db-schema 已结束"
ps -p $BE_PID > /dev/null && echo "backend-scaffold 运行中" || echo "backend-scaffold 已结束"
```

**优点**：
- 完全进程隔离，一个崩溃不影响其他
- 日志持久化，可随时回看
- 不受 15 分钟后台超时限制

**缺点**：
- 主代理（当前会话）无法直接读取子会话的上下文，只能通过 log 文件间接了解
- 没有内置的 `TaskList` 统一管理

---

## 方案 B: 单会话内后台任务（折中）

在同一个 kimi 会话内，用 `Shell` 工具的 `run_in_background=true` 启动多个后台命令。

```bash
Shell(
  command="cd ./database && kimi --session db-task '设计数据库 Schema...' > ../.kimi/logs/db.log 2>&1",
  run_in_background=true,
  description="启动数据库设计任务"
)

Shell(
  command="cd ./backend && kimi --session backend-task '搭建后端脚手架...' > ../.kimi/logs/backend.log 2>&1",
  run_in_background=true,
  description="启动后端脚手架任务"
)
```

**查询进度**：

```bash
# 主代理通过 Shell 查询日志
Shell(command="tail -n 20 ./.kimi/logs/db.log")
Shell(command="tail -n 20 ./.kimi/logs/backend.log")
```

**优点**：
- 主代理可以用 `TaskList` / `TaskOutput` 管理这些后台 Shell 任务
- 日志集中管理

**缺点**：
- 子 kimi 实例仍是独立进程，主代理无法直接读取它们的内部状态

---

## 方案 C: 顺序脚本编排（最简单）

如果用户项目较小，不需要真正的并行，可以用一个 bash 脚本顺序执行：

```bash
#!/bin/bash
set -e

echo "=== Phase 1: 数据库设计 ==="
kimi --work-dir ./database --session db "设计 Schema..."

echo "=== Phase 2: 后端开发 ==="
kimi --work-dir ./backend --session backend "基于 schema 开发 API..."

echo "=== Phase 3: 前端开发 ==="
kimi --work-dir ./frontend --session frontend "基于 API 文档开发页面..."

echo "=== Phase 4: 集成测试 ==="
kimi --work-dir . --session integration "联调前后端..."
```

**适用场景**：
- 用户明确说"一步步来，不要并行"
- 模块之间强依赖，必须等前一个完全完成

---

## 与内置 Agent 工具的对比

| 维度 | 内置 `Agent` 工具 | Shell 独立会话 |
|---|---|---|
| 并行能力 | ✅ `run_in_background=true` | ✅ `&` 后台进程 |
| 主代理汇总 | ✅ 直接读取子代理输出 | ❌ 需读 log 文件 |
| 上下文隔离 | ✅ 子代理独立上下文 | ✅ 完全进程隔离 |
| 超时控制 | ⚠️ 默认 15 分钟（可配） | ✅ 无限制 |
| 审批流 | ✅ 统一审批通道 | ❌ 各实例独立审批 |
| 进度查询 | ✅ `TaskList` / `TaskOutput` | ⚠️ `ps` + `tail` |
| 恢复/续接 | ✅ `agent_id` resume | ❌ 需重新启动 |

**建议**：
- 默认优先使用内置 `Agent` 工具（体验更好，主代理能直接管理）
- 当用户明确要求"用终端跑"、"要独立窗口"、"任务可能超过 15 分钟"时，切换到 Shell 方案
