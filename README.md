# QClaw ↔ WorkBuddy 任务分发桥

通过共享 JSON 队列，实现 **QClaw（微信入口）** 和 **WorkBuddy（执行引擎）** 的双向打通。

> 用户在微信/QClaw 提交复杂任务 → WorkBuddy 自动执行 → 结果推送回微信

## 架构

```
微信/QClaw 对话
    ├── 提交任务 → qclaw-task-submitter
    │       → qclaw_queue.py add "任务描述"
    │       → ~/.workbuddy/queue/qclaw_tasks.json
    │
    ├── [每10分钟自动化轮询]
    │       → qclaw-bridge-poll Automation
    │       → WorkBuddy 读取 pending → 执行 → done
    │
    └── 查询结果 → qclaw-result-checker
            → qclaw_queue.py result <id>
            → 展示执行结果
```

## 包含的 Skill

| 目录 | 功能 | 触发场景 |
|------|------|---------|
| `skills/qclaw-task-submitter/` | 提交任务到 WorkBuddy | "帮我生成/分析/制作..."、"交给 WorkBuddy" |
| `skills/qclaw-result-checker/` | 查询任务执行结果 | "结果出来了吗"、"任务完成了没" |
| `skills/qclaw-workbuddy-bridge/` | 队列核心脚本（qclaw_queue.py） | 被上述两个 Skill 调用 |

## 快速安装

```bash
# WorkBuddy 内置市场（推荐）
clawhub install qclaw-task-submitter
clawhub install qclaw-result-checker
clawhub install qclaw-workbuddy-bridge

# 或直接从 GitHub 克隆
git clone https://github.com/liuboacean/qclaw-workbuddy-bridge.git
```

## 初始化队列

```bash
python3 skills/qclaw-workbuddy-bridge/scripts/qclaw_queue.py list
```

## 配置自动化

在 WorkBuddy 中创建定时自动化：

- **名称**: `qclaw-bridge-poll`
- **频率**: 每 10 分钟一次
- **prompt**: 见 `skills/qclaw-workbuddy-bridge/SKILL.md` 第三步

## 队列操作命令

```bash
# 添加任务
python3 .../qclaw_queue.py add "任务描述"

# 查看队列
python3 .../qclaw_queue.py list --status pending

# 获取结果
python3 .../qclaw_queue.py result <task_id>

# 标记完成
python3 .../qclaw_queue.py done <task_id> <result.json>
```

## 目录结构

```
qclaw-workbuddy-bridge/
├── skills/
│   ├── qclaw-task-submitter/     # QClaw 任务提交 Skill
│   ├── qclaw-result-checker/     # QClaw 结果查询 Skill
│   └── qclaw-workbuddy-bridge/   # 核心队列脚本
│       └── scripts/
│           └── qclaw_queue.py    # 队列管理 CLI
├── queue/
│   └── qclaw_tasks.json          # 任务队列文件（示例）
└── README.md
```

## 发布信息

- **GitHub**: github.com/liuboacean/qclaw-workbuddy-bridge
- **ClawHub**: clawhub.com/liuboacean/skills/qclaw-workbuddy-bridge
- **SkillHub**: skillhub.cloud.tencent.com/skills/qclaw-workbuddy-bridge
- **版本**: v1.0.0
- **作者**: liuboacean

## 完全无轮询模式（launchd，推荐）

通过 macOS launchd 监听触发文件，**零 Token 消耗，文件出现即执行**。

### 安装 launchd Agent

```bash
# 安装 agent（自动安装）
launchctl load ~/Library/LaunchAgents/com.liubo.qclaw-bridge.plist

# 确认运行状态
launchctl list | grep qclaw-bridge
```

### 工作原理

```
QClaw 提交任务 → add 命令写 .trigger 文件
                           ↓
              launchd 检测到文件 → 触发 watch --once
                           ↓
              qclaw_queue.py 处理 pending 任务
                           ↓
              任务 done 后删除 .trigger → 等待下一轮
```

**零轮询，完全事件驱动**。

### 卸载

```bash
launchctl unload ~/Library/LaunchAgents/com.liubo.qclaw-bridge.plist
```
