# OpenClaw 健康监控

> OpenClaw 部署的自愈基础设施

## 为什么要做这套东西

OpenClaw 是一个强大的 AI 网关，作为持久化后台进程运行——全天候处理消息、执行任务、协调 Agent。但和所有长期运行的进程一样，问题总会出现：

- 凌晨 3 点 Gateway 崩溃了，没人知道，直到早上才发现
- 某个 Session 永远卡在"思考中"，悄悄地堵塞整个队列
- 上次崩溃留下的 lock 文件，导致新 Session 无法启动
- 两个 Gateway 进程同时运行，互相打架
- API Provider 挂了，Agent 就这么停止响应了

标准答案是"手动重启一下"。但如果你在构建一个全天候运行的 AI 助手，这个答案远远不够。

**我们需要一套能自我监控、自我修复的系统。**

## 它能做什么

OpenClaw 健康监控是一个轻量级 Shell 脚本，通过 macOS `launchd` 每 5 分钟运行一次。它执行六项自动化检查：

### 1. 多进程检测
检测多个 Gateway 进程同时运行的情况（消息重复和状态损坏的常见原因）。自动保留最新进程，终止其余进程。

### 2. 僵尸 Lock 文件清理
崩溃后 Session lock 文件可能变成孤儿。监控程序检测到所属进程已不存在的 lock 文件后，自动删除，解除对新 Session 的阻塞。

### 3. Gateway 自动重启
如果 Gateway 进程未在运行，监控程序自动启动它，并发送 wake 事件通知 Agent 恢复进行中的任务。

### 4. 卡死 Session 检测
检测两种类型的卡死 AI Session：
- **仅有 thinking 的 Session**：助手只生成了 `<thinking>` 块就停止了——生成中断的标志
- **Synthetic error toolResult Session**：工具调用以 synthetic error 失败，Session 再也没有恢复

两种类型在 5 分钟宽限期后自动清理。

### 5. Provider 错误自动重试
当 API Provider 失败时（如"All models failed"），监控程序从日志中检测到错误，发送 wake 事件触发重试。包含退避机制和最大重试次数限制（5 次重试，2 分钟间隔）。

### 6. 队列卡住检测
如果某个 Session 在队列中超过 3 分钟未完成，监控程序重启 Gateway 清除积压，并通知 Agent 恢复任务。

## 修复产物存档

监控程序的每一个操作都会带时间戳记录到 `~/.openclaw/logs/health-check.log`。这给你提供了完整的审计追踪——什么出了问题、什么时候、怎么修复的——无需翻阅原始 Gateway 日志。

## 不包含的功能

本系统**不包含** AI 驱动的修复功能。没有 LLM 分析你的日志或生成修复建议。每项检查都是确定性的：条件要么为真要么为假，响应是预定义的。

这是有意为之的。确定性检查的优势：
- 快速（无 API 调用）
- 可靠（无幻觉）
- 可审计（你可以直接读懂它做了什么）

## 工作原理

```
launchd（每 5 分钟）
    └── gateway-health-check.sh
            ├── check_gateway_running()        # Gateway 是否在运行
            ├── check_multiple_gateways()      # 多进程检测
            ├── check_stale_locks()            # 僵尸 lock 文件
            ├── check_stuck_thinking_sessions() # 卡死 Session
            ├── check_queue_stuck()            # 队列卡住
            └── check_provider_errors()        # Provider 错误重试
```

修复操作执行后，脚本向 OpenClaw Gateway 发送 `cron/wake` 事件，触发主 Agent 检查任务状态并恢复被中断的工作。

## 环境要求

- macOS（使用 `launchd` 和 `stat -f %m`）
- 已安装并配置 OpenClaw Gateway
- `curl`（macOS 预装）

## 安装

```bash
# 复制脚本
cp gateway-health-check.sh ~/.openclaw/workspace/scripts/

# 安装 launchd plist
cp ai.openclaw.health-check.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/ai.openclaw.health-check.plist

# 验证运行状态
launchctl list | grep openclaw
```

## 许可证

MIT
