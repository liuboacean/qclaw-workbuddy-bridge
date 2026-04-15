# QClaw ↔ WorkBuddy 任务分发桥

通过共享 JSON 队列 + macOS launchd 事件驱动，实现 **QClaw（微信入口）** 和 **WorkBuddy（执行引擎）** 的双向打通。**零轮询、零 Token 浪费**。

> 用户在微信/QClaw 提交复杂任务 → WorkBuddy 自动执行 → 结果推送回微信

---

## 架构（v1.2.0 · 零轮询模式）

```
微信/QClaw 对话
    ├── 提交任务 → qclaw-task-submitter Skill
    │       → qclaw_queue.py add "任务描述"
    │       → 写入 ~/.workbuddy/queue/qclaw_tasks.json
    │       → 写入 ~/.workbuddy/queue/.trigger 触发文件
    │
    ├── [launchd 事件触发，零 Token 消耗]
    │       → 检测到 .trigger 文件出现
    │       → 触发 qclaw_queue.py watch --once
    │       → 处理所有 pending 任务
    │       → 执行完成 → 删除 .trigger → 退出
    │
    └── 查询结果 → qclaw-result-checker Skill
            → qclaw_queue.py list --status done
            → qclaw_queue.py result <task_id>
            → 展示执行结果
```

### 为什么是零 Token

| 方案 | 触发方式 | 无任务时 |
|------|---------|---------|
| 旧版轮询（已废弃） | 定时唤醒 | 每 30 分钟浪费一次 Token |
| **当前版本（v1.2.0）** | macOS launchd 事件驱动 | **完全不触发，零消耗** |

---

## 包含的 Skill

| 目录 | 功能 | 触发场景 |
|------|------|---------|
| `skills/qclaw-task-submitter/` | 提交任务到 WorkBuddy | "帮我生成/分析/制作..."、"交给 WorkBuddy" |
| `skills/qclaw-result-checker/` | 查询任务执行结果 | "结果出来了吗"、"任务完成了没" |
| `skills/qclaw-workbuddy-bridge/` | 队列核心脚本（qclaw_queue.py） | 被上述两个 Skill 调用 |

---

## 快速安装

```bash
# ClawHub 安装（推荐）
clawhub install qclaw-task-submitter
clawhub install qclaw-result-checker
clawhub install qclaw-workbuddy-bridge

# 或直接从 GitHub 克隆
git clone https://github.com/liuboacean/qclaw-workbuddy-bridge.git
```

---

## 启动 launchd 触发机制（核心）

```bash
# 确认队列目录存在（首次自动创建）
python3 skills/qclaw-workbuddy-bridge/scripts/qclaw_queue.py list

# 加载 launchd Agent（macOS 系统服务）
launchctl load ~/Library/LaunchAgents/com.liubo.qclaw-bridge.plist

# 确认运行状态
launchctl list | grep qclaw-bridge
```

> **无需配置 WorkBuddy 自动化**。launchd 监听 `.trigger` 文件，文件出现即触发 WorkBuddy 处理任务。

### 手动兜底（launchd 未触发时）

```bash
python3 skills/qclaw-workbuddy-bridge/scripts/qclaw_queue.py watch --once
```

### 卸载

```bash
launchctl unload ~/Library/LaunchAgents/com.liubo.qclaw-bridge.plist
```

---

## 队列操作命令

```bash
# 提交任务（同时自动写.trigger，触发 launchd）
python3 .../qclaw_queue.py add "任务描述"

# 监听触发（launchd 调用，或手动兜底）
python3 .../qclaw_queue.py watch --once

# 查看队列
python3 .../qclaw_queue.py list

# 获取单个任务结果
python3 .../qclaw_queue.py result <task_id>

# 更新任务状态
python3 .../qclaw_queue.py status <task_id> processing
python3 .../qclaw_queue.py done <task_id> <result.json>
```

---

## 目录结构

```
qclaw-workbuddy-bridge/
├── skills/
│   ├── qclaw-task-submitter/      # QClaw 任务提交 Skill
│   ├── qclaw-result-checker/      # QClaw 结果查询 Skill
│   └── qclaw-workbuddy-bridge/    # 核心队列脚本
│       └── scripts/
│           └── qclaw_queue.py     # 队列管理 CLI（全命令见上方）
├── queue/
│   └── qclaw_tasks.json           # 任务队列文件（示例）
└── README.md
```

---

## 发布信息

| 平台 | 链接 |
|------|------|
| GitHub | github.com/liuboacean/qclaw-workbuddy-bridge |
| ClawHub | clawhub.com/liuboacean/skills/qclaw-workbuddy-bridge |
| SkillHub | skillhub.cloud.tencent.com/skills/qclaw-workbuddy-bridge |

- **版本**: v1.2.0
- **作者**: liuboacean
- **许可**: MIT-0
