# OpenClaw Health Monitor

> Self-healing infrastructure for OpenClaw deployments

## Why We Built This

OpenClaw is a powerful AI gateway that runs as a persistent background process — handling messages, executing tasks, and coordinating agents around the clock. But like any long-running process, things go wrong:

- The gateway crashes at 3am and nobody notices until morning
- A session gets stuck "thinking" forever, silently blocking the queue
- A stale lock file from a previous crash prevents new sessions from starting
- Two gateway processes start fighting each other
- An API provider goes down and the agent just... stops responding

The standard answer is "just restart it manually." But that's not good enough when you're building an AI assistant that's supposed to be always-on.

**We needed a system that could watch itself and fix itself.**

## What It Does

OpenClaw Health Monitor is a lightweight shell script that runs every 5 minutes via macOS `launchd`. It performs six automated checks:

### 1. Multi-Process Detection
Detects when multiple gateway processes are running simultaneously (a common cause of message duplication and state corruption). Kills all but the newest process.

### 2. Stale Lock File Cleanup
Session lock files can get orphaned after a crash. The monitor detects locks whose owning process is no longer running and removes them, unblocking new sessions.

### 3. Gateway Auto-Restart
If the gateway process is not running, the monitor starts it automatically and sends a wake event to notify agents to resume any in-progress tasks.

### 4. Stuck Session Detection
Detects two types of stuck AI sessions:
- **Thinking-only sessions**: The assistant generated only a `<thinking>` block and stopped — a sign of interrupted generation
- **Synthetic error toolResult sessions**: A tool call failed with a synthetic error and the session never recovered

Both types are cleaned up automatically after a 5-minute grace period.

### 5. Provider Error Auto-Retry
When an API provider fails (e.g., "All models failed"), the monitor detects the error from logs and sends a wake event to trigger a retry. Includes exponential backoff and a maximum retry limit (5 retries, 2-minute intervals).

### 6. Queue Stuck Detection
If a session has been in the queue for more than 3 minutes without completing, the monitor restarts the gateway to clear the backlog and notifies agents to resume.

## Repair Artifact Archiving

Every action taken by the monitor is logged to `~/.openclaw/logs/health-check.log` with timestamps. This gives you a full audit trail of what broke, when, and how it was fixed — without needing to dig through raw gateway logs.

## What's NOT Included

This system does **not** include AI-powered repair. There is no LLM analyzing your logs or generating fix suggestions. Every check is deterministic: a condition is either true or false, and the response is predefined.

This is intentional. Deterministic checks are:
- Fast (no API calls)
- Reliable (no hallucinations)
- Auditable (you can read exactly what it does)

## How It Works

```
launchd (every 5 min)
    └── gateway-health-check.sh
            ├── check_gateway_running()
            ├── check_multiple_gateways()
            ├── check_stale_locks()
            ├── check_stuck_thinking_sessions()
            ├── check_queue_stuck()
            └── check_provider_errors()
```

When a fix is applied, the script sends a `cron/wake` event to the OpenClaw gateway, which triggers the main agent to check task state and resume any interrupted work.

## Requirements

- macOS (uses `launchd` and `stat -f %m`)
- OpenClaw gateway installed and configured
- `curl` (pre-installed on macOS)

## Installation

```bash
# Copy the script
cp gateway-health-check.sh ~/.openclaw/workspace/scripts/

# Install the launchd plist
cp ai.openclaw.health-check.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/ai.openclaw.health-check.plist

# Verify it's running
launchctl list | grep openclaw
```

## License

MIT
