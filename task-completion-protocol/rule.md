# Task Completion Protocol - 状态流转与文件同步规则

> 本文档定义了 HiClaw 任务管理中的状态流转规则和文件同步协议。遵循“MinIO 单点真相”、“无状态 Worker”和“Matrix 房间透明通信”的核心理念。

---

## 1. 任务文件结构与读写权限

所有任务状态和产出均持久化在 MinIO 共享文件系统中，Worker **不得在本地维护持久化状态**。

```
shared/tasks/{task-id}/
├── meta.json          # 任务元数据（状态、分配者、时间戳） 【只读：Worker，可写：Manager】
├── spec.md            # 任务规格（由 Manager 创建） 【只读：Worker，可写：Manager】
├── plan.md            # 执行计划（由 Worker 创建） 【可写：Worker】
├── result.md          # 执行结果（由 Worker 创建） 【可写：Worker】
└── base/              # 参考资料（Manager 维护） 【只读：Worker，可写：Manager】
```

### 1.1 meta.json 结构

`meta.json` 是任务状态的**唯一权威来源**，**仅允许 Manager 修改**。

```json
{
  "task_id": "task-20260327-001",
  "title": "实现登录页面",
  "type": "finite",
  "status": "assigned",
  "created_at": "2026-03-27T10:00:00+08:00",
  "created_by": "admin",
  "assigned_to": "alice",
  "assigned_at": "2026-03-27T10:05:00+08:00",
  "completed_at": null,
  "result_summary": null
}
```

---

## 2. 任务状态流转 (单向数据流)

任务状态变更通过 Matrix 房间的自然语言沟通触发，由 Manager 统一更新底层文件。

### 2.1 状态定义

| 状态 | 含义 | 更新者 (写 meta.json) | 触发方式 |
|------|------|--------|----------|
| `created` | 任务已创建，未分配 | Manager | Admin 下发任务 |
| `assigned` | 任务已分配，待执行 | Manager | Manager 分配给 Worker |
| `in_progress` | 任务执行中 | Manager | Worker 在群里报告“我已开始执行” |
| `completed` | 任务已完成 | Manager | Worker 提交 `result.md` 后在群里报告完成 |
| `blocked` | 任务阻塞中 | Manager | Worker 在群里报告阻塞原因并求助 |
| `cancelled` | 任务已取消 | Manager | Admin 在群里要求取消任务 |

### 2.2 核心协作原则

1. **Worker 不写 meta.json**：Worker 改变状态的唯一方式是在对应的 Matrix 任务房间中 `@Manager` 报告状态（如“我已经完成了”或“我卡住了”）。
2. **Manager 统一更新**：Manager 收到通知后，负责调用状态更新工具修改 `meta.json`。
3. **全透明通信**：**严禁使用私聊通知**。所有的状态报告、阻塞求助和完成确认，必须在包含 `[Admin, Manager, Worker]` 的 Matrix 任务房间中进行。

---

## 3. 文件同步规则 (TaskSync 工具)

为了防止多并发导致的旧版本覆盖和脏写，**严禁 Agent 手写 `mc mirror` 命令**。必须使用内置的 `task-sync.sh` 工具进行同步。

### 3.1 Worker 同步操作

Worker 只需要关注自己负责的 `result.md` 和 `plan.md`。

- **拉取最新任务上下文（执行前）**：
  ```bash
  bash /opt/hiclaw/scripts/task-sync.sh pull --task-id {task-id}
  ```
  *(默认仅拉取 `meta.json` / `spec.md` / `base/`，不会覆盖 Worker 本地的 `result.md` / `plan.md` / `progress/` 草稿)*

- **拉取完整任务目录（恢复会话/换容器后）**：
  ```bash
  bash /opt/hiclaw/scripts/task-sync.sh pull-full --task-id {task-id}
  ```

- **推送执行结果（完成/阻塞时）**：
  ```bash
  bash /opt/hiclaw/scripts/task-sync.sh push --task-id {task-id}
  ```
  *(工具会自动推送 `result.md` / `plan.md` / `progress/`，并**严格保护** Manager 的 `meta.json` / `spec.md` / `base/` 不被 Worker 覆盖)*

### 3.2 Manager 同步操作

Manager 负责维护全局状态和任务规格。

- **更新任务状态**：
  Manager 不直接手工编辑 JSON，而是使用确定性脚本更新 `meta.json`：
  ```bash
  bash /opt/hiclaw/agent/skills/task-management/scripts/manage-task-meta.sh --action set-status --task-id {task-id} --status completed
  ```
  *(脚本会拉取远端 `meta.json`，写入状态字段，并推送回 MinIO)*

---

## 4. 典型场景流转示例

### 场景 A：Worker 正常完成任务

1. **Worker**：执行代码并生成 `result.md`。
2. **Worker**：运行 `bash /opt/hiclaw/scripts/task-sync.sh push --task-id {id}` 推送产出。
3. **Worker**：在 Matrix 任务房间发送消息：“@Manager 任务已完成，结果已提交。”
4. **Manager**：收到消息，检查 MinIO 中的 `result.md` 是否符合要求。
5. **Manager**：运行 `manage-task-meta.sh --action set-status --task-id {id} --status completed`。
6. **Manager**：在 Matrix 任务房间回复：“@Admin @Worker 结果确认无误，任务状态已标记为 completed。”

### 场景 B：Worker 遇到阻塞

1. **Worker**：将阻塞原因写入 `result.md` 或直接记录在工作区。
2. **Worker**：运行 `bash /opt/hiclaw/scripts/task-sync.sh push --task-id {id}`（如需要共享现场）。
3. **Worker**：在 Matrix 任务房间发送消息：“@Manager @Admin 我遇到了依赖冲突，无法继续执行，请看日志。”
4. **Manager**：运行 `manage-task-meta.sh --action set-status --task-id {id} --status blocked`。
5. **Admin / Manager**：在群内协助排查问题，提供解决方案。

---

## 5. 心跳与状态对齐机制 (Heartbeat)

为防止“Worker 已完成但 Manager 漏看消息”导致的状态不一致，Manager 的 Heartbeat 机制必须包含**状态对齐巡检**：

1. 定期扫描 MinIO 中的 `meta.json` 与 Manager 内存中的 `state.json`。
2. 发现 Worker 已经推送了包含成功结果的 `result.md` 但 `meta.json` 仍为 `assigned` 时，Manager 应主动在群内询问 Worker 或直接进入确认流程。
3. 发现 Worker 容器异常退出（长达 X 分钟无活动），将任务状态标记为 `blocked` 并向 Admin 告警。
