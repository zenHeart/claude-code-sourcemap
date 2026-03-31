# Claude Code Cheatsheet

> Complete reference for Claude Code CLI
> Based on official docs and source code analysis

---

## 1. 特性功能总览

| 特性 | 概述 |
|------|------|
| `cli` | 命令行入口，支持多种启动模式和全局参数 |
| `slash-command` | 斜杠命令，80+ 内置命令（/help, /commit, /review 等） |
| `hooks` | 生命周期钩子，30+ 事件类型，支持 command/http/prompt/agent |
| `subagent` | 子 Agent 编排，支持自定义 agent 类型和工具白名单 |
| `tools` | 40+ 内置工具（Read/Edit/Bash/Grep 等） |
| `skills` | 可复用技能包，支持 YAML 定义和 hook 嵌入 |
| `permissions` | 细粒度权限系统，支持规则匹配和交互确认 |
| `mcp` | Model Context Protocol，支持 MCP 服务器连接 |
| `worktree` | Git worktree 隔离，支持并行会话 |
| `compact` | 上下文压缩，自动管理 token 预算 |
| `session` | 会话管理，支持 resume/clear/export |
| `team` | 多 Agent 协作，支持 teammate 角色定义 |
| `config` | 多层配置（user/project/local/policy） |

---

## 2. CLI 命令

### 2.1 全局参数

| 参数 | 功能 | 示例 |
|------|------|------|
| `--version`, `-v` | 显示版本号 | `claude --version` |
| `--model <model>` | 指定模型 | `claude --model sonnet` |
| `--no-input` | 非交互模式 | `claude --no-input "..."` |
| `--output-format` | 输出格式 (json/text) | `claude --output-format json` |
| `--add-dir <path>` | 添加额外目录 | `claude --add-dir /path` |
| `--resume [<session-id>]` | 恢复会话 | `claude --resume` |
| `--worktree <name>` | 创建 git worktree | `claude --worktree feature` |
| `--agent <type>` | 指定 agent 类型 | `claude --agent Explore` |
| `--max-turns <n>` | 最大轮数限制 | `claude --max-turns 10` |
| `--budget <amount>` | token 预算上限 | `claude --budget 10000` |
| `--permission-mode <mode>` | 权限模式 | `claude --permission-mode auto` |
| `--no-env` | 禁用环境变量读取 | `claude --no-env` |

### 2.2 子命令

| 命令 | 功能 | 示例 |
|------|------|------|
| `claude remote-control` | 启动远程控制模式 | `claude remote-control` |
| `claude daemon` | 启动守护进程 | `claude daemon` |
| `claude init` | 初始化项目配置 | `claude init` |
| `claude login` | 登录 Claude | `claude login` |
| `claude logout` | 登出 | `claude logout` |
| `claude config` | 配置管理 | `claude config edit` |
| `claude mcp` | MCP 服务器管理 | `claude mcp list` |
| `claude plugin` | 插件管理 | `claude plugin install` |
| `claude skills` | 技能管理 | `claude skills list` |

---

## 3. Slash Commands

### 3.1 核心命令

| 命令 | 功能 | 示例 |
|------|------|------|
| `/help` | 显示帮助 | `/help` |
| `/exit` | 退出 | `/exit` |
| `/clear` | 清除会话 | `/clear` |
| `/resume` | 恢复会话 | `/resume` |
| `/compact` | 手动压缩上下文 | `/compact` |
| `/rewind` | 回退消息 | `/rewind 5` |

### 3.2 Git 操作

| 命令 | 功能 | 示例 |
|------|------|------|
| `/commit` | 创建 commit | `/commit` |
| `/branch` | 列出/创建分支 | `/branch` |
| `/diff` | 显示更改 | `/diff` |
| `/review` | 代码审查 | `/review` |
| `/pr_comments` | 查看 PR 评论 | `/pr_comments` |

### 3.3 开发辅助

| 命令 | 功能 | 示例 |
|------|------|------|
| `/plan` | 进入计划模式 | `/plan` |
| `/tasks` | 任务管理 | `/tasks` |
| `/model` | 切换模型 | `/model sonnet` |
| `/permissions` | 权限管理 | `/permissions` |
| `/cost` | 显示 token 消耗 | `/cost` |
| `/stats` | 显示统计 | `/stats` |

### 3.4 配置管理

| 命令 | 功能 | 示例 |
|------|------|------|
| `/config` | 编辑配置 | `/config edit` |
| `/hooks` | 查看 hooks | `/hooks` |
| `/skills` | 管理技能 | `/skills list` |
| `/mcp` | MCP 服务器 | `/mcp list` |
| `/env` | 环境变量 | `/env` |
| `/theme` | 主题设置 | `/theme` |

### 3.5 高级功能

| 命令 | 功能 | 示例 |
|------|------|------|
| `/agents` | Agent 管理 | `/agents` |
| `/memory` | 记忆管理 | `/memory` |
| `/context` | 上下文查看 | `/context` |
| `/btw` | 记笔记 | `/btw ...` |
| `/feedback` | 发送反馈 | `/feedback` |

### 3.6 全部命令列表

```
add-dir, agents, ant-trace, autofix-pr, backfill-sessions, branch,
break-cache, bridge, btw, bughunter, chrome, clear, color, compact,
config, context, copy, cost, ctx_viz, debug-tool-call, desktop,
diff, doctor, effort, env, exit, export, extra-usage, fast, feedback,
files, good-claude, heapdump, help, hooks, ide, install-github-app,
install-slack-app, issue, keybindings, login, logout, mcp, memory,
mobile, mock-limits, model, oauth-refresh, onboarding, output-style,
passes, perf-issue, permissions, plan, plugin, privacy-settings,
pr_comments, rate-limit-options, release-notes, reload-plugins,
remote-env, remote-setup, rename, reset-limits, resume, review, rewind,
sandbox-toggle, session, share, skills, stats, status, stickers,
summary, tag, tasks, teleport, terminalSetup, theme, thinkback,
thinkback-play, upgrade, usage, vim, voice
```

---

## 4. Hooks 系统

### 4.1 Hook 事件类型

| 事件 | 触发时机 | 可阻塞 |
|------|----------|--------|
| `SessionStart` | 会话开始或恢复 | 否 |
| `UserPromptSubmit` | 用户提交提示词 | 是 |
| `PreToolUse` | 工具执行前 | 是 |
| `PermissionRequest` | 权限请求时 | 是 |
| `PostToolUse` | 工具执行后 | 否 |
| `PostToolUseFailure` | 工具执行失败后 | 否 |
| `Stop` | 响应完成时 | 是 |
| `StopFailure` | API 错误时 | 否 |
| `SubagentStart` | 子 Agent 启动 | 否 |
| `SubagentStop` | 子 Agent 停止 | 是 |
| `TaskCreated` | 任务创建时 | 是 |
| `TaskCompleted` | 任务完成时 | 是 |
| `Notification` | 通知发送时 | 否 |
| `PreCompact` | 上下文压缩前 | 否 |
| `PostCompact` | 上下文压缩后 | 否 |
| `InstructionsLoaded` | 指令加载时 | 否 |
| `ConfigChange` | 配置变更时 | 是 |
| `CwdChanged` | 工作目录变更时 | 否 |
| `FileChanged` | 文件变更时 | 否 |
| `WorktreeCreate` | worktree 创建时 | 是 |
| `WorktreeRemove` | worktree 删除时 | 否 |
| `Elicitation` | MCP 请求用户输入 | 是 |
| `ElicitationResult` | 用户响应后 | 是 |
| `TeammateIdle` | teammate 空闲时 | 是 |
| `SessionEnd` | 会话结束时 | 否 |

### 4.2 Matcher 模式

| 事件 | Matcher 字段 | 示例 |
|------|-------------|------|
| `PreToolUse`, `PostToolUse` | tool name | `Bash`, `Edit`, `mcp__.*` |
| `SessionStart` | how started | `startup`, `resume`, `clear` |
| `SessionEnd` | why ended | `clear`, `logout`, `bypass_permissions_disabled` |
| `Notification` | notification type | `idle_prompt`, `permission_prompt` |
| `SubagentStart` | agent type | `Bash`, `Explore`, `Plan` |

### 4.3 Hook 类型

| 类型 | 字段 | 说明 |
|------|------|------|
| `command` | `command` | 执行 shell 命令 |
| `http` | `url`, `headers` | 发送 HTTP POST 请求 |
| `prompt` | `prompt`, `model` | LLM 评估 |
| `agent` | `prompt`, `model` | Agent 执行 |

### 4.4 Hook 配置位置

| 位置 | 作用域 | 可共享 |
|------|--------|--------|
| `~/.claude/settings.json` | 用户级别 | 否 |
| `.claude/settings.json` | 项目级别 | 是 |
| `.claude/settings.local.json` | 本地级别 | 否 |
| 插件 `hooks/hooks.json` | 插件级别 | 是 |
| Skill/Agent frontmatter | 组件级别 | 是 |

### 4.5 Hook 配置示例

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(rm *)",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/block-rm.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/lint.sh"
          }
        ]
      }
    ]
  }
}
```

---

## 5. Tools 系统

### 5.1 文件操作工具

| 工具 | 描述 | 权限 |
|------|------|------|
| `Read` | 读取文件内容 | 否 |
| `Write` | 创建/覆盖文件 | 是 |
| `Edit` | 精准编辑文件 | 是 |
| `Glob` | 模式匹配文件 | 否 |
| `Grep` | 搜索文件内容 | 否 |
| `NotebookEdit` | Jupyter notebook 编辑 | 是 |

### 5.2 执行工具

| 工具 | 描述 | 权限 |
|------|------|------|
| `Bash` | 执行 shell 命令 | 是 |
| `PowerShell` | 执行 PowerShell (Windows) | 是 |

### 5.3 代码智能工具

| 工具 | 描述 | 权限 |
|------|------|------|
| `LSP` | 语言服务器协议（跳转定义、引用等） | 否 |
| `Grep` | 搜索代码模式 | 否 |

### 5.4 Agent 相关工具

| 工具 | 描述 | 权限 |
|------|------|------|
| `Agent` | 派生子 Agent | 否 |
| `EnterPlanMode` | 进入计划模式 | 否 |
| `ExitPlanMode` | 退出计划模式 | 是 |
| `EnterWorktree` | 创建 worktree 并切换 | 否 |
| `ExitWorktree` | 退出 worktree | 否 |

### 5.5 任务管理工具

| 工具 | 描述 | 权限 |
|------|------|------|
| `TaskCreate` | 创建任务 | 否 |
| `TaskGet` | 获取任务详情 | 否 |
| `TaskList` | 列出任务 | 否 |
| `TaskUpdate` | 更新任务 | 否 |
| `TaskStop` | 停止任务 | 否 |
| `TodoWrite` | 写入 todo | 否 |

### 5.6 定时任务工具

| 工具 | 描述 | 权限 |
|------|------|------|
| `CronCreate` | 创建定时任务 | 否 |
| `CronDelete` | 删除定时任务 | 否 |
| `CronList` | 列出定时任务 | 否 |

### 5.7 Web 工具

| 工具 | 描述 | 权限 |
|------|------|------|
| `WebFetch` | 获取网页内容 | 是 |
| `WebSearch` | 网络搜索 | 是 |

### 5.8 团队协作工具

| 工具 | 描述 | 权限 |
|------|------|------|
| `TeamCreate` | 创建团队 | 否 |
| `TeamDelete` | 删除团队 | 否 |
| `SendMessage` | 发送消息 | 否 |

### 5.9 其他工具

| 工具 | 描述 | 权限 |
|------|------|------|
| `AskUserQuestion` | 询问用户问题 | 否 |
| `Skill` | 执行技能 | 是 |
| `MCP` | MCP 工具调用 | 否 |
| `ListMcpResourcesTool` | 列出 MCP 资源 | 否 |
| `ReadMcpResourceTool` | 读取 MCP 资源 | 否 |
| `ToolSearch` | 搜索 MCP 工具 | 否 |
| `Brief` | 生成简要总结 | 否 |
| `Config` | 配置操作 | 否 |

---

## 6. Subagent 系统

### 6.1 内置 Agent 类型

| 类型 | 用途 |
|------|------|
| `Bash` | 执行命令 |
| `Explore` | 代码探索 |
| `Plan` | 制定计划 |
| 自定义 | 用户定义的 agent |

### 6.2 Agent 配置

```yaml
---
name: code-reviewer
description: Expert code reviewer
agent:
  prompt: "You are a code reviewer..."
  tools:
    - Read
    - Grep
    - Glob
    - Bash
  maxTurns: 10
---
```

### 6.3 Coordinator 模式

```yaml
---
name: coordinator
agent:
  prompt: "Coordinate a team of agents..."
  mode: coordinator
  workers:
    - type: Bash
    - type: Explore
    - type: Plan
---
```

---

## 7. Skills 系统

### 7.1 Skill 定义

```yaml
---
name: my-skill
description: A useful skill
commands:
  - name: greet
    description: Greet someone
    action:
      type: command
      command: echo "Hello!"
hooks:
  PreToolUse:
    - matcher: Bash
      hooks:
        - type: command
          command: ./hooks/pre-bash.sh
---
```

### 7.2 触发方式

| 方式 | 说明 |
|------|------|
| `/skill <name>` | 斜杠命令触发 |
| `@<name>` | @ 提及触发 |
| 自动 | 匹配关键词自动触发 |

---

## 8. Permissions 系统

### 8.1 权限模式

| 模式 | 说明 |
|------|------|
| `default` | 交互确认 |
| `plan` | 仅计划模式 |
| `auto` | 自动允许安全操作 |
| `bypassPermissions` | 跳过所有确认 |
| `dontAsk` | 不询问，直接拒绝 |

### 8.2 权限规则

```json
{
  "permissions": {
    "allow": [
      "Read(/src/**/*.ts)",
      "Glob(*.md)",
      "Bash(git *)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Write(/etc/**)"
    ]
  }
}
```

---

## 9. Session 会话

### 9.1 会话命令

| 命令 | 功能 |
|------|------|
| `/session` | 查看当前会话 |
| `/resume` | 恢复会话 |
| `/clear` | 清除会话 |
| `/export` | 导出会话 |
| `/share` | 分享会话 |

### 9.2 会话元数据

| 字段 | 说明 |
|------|------|
| `session_id` | 会话唯一 ID |
| `transcript_path` | 记录文件路径 |
| `cwd` | 当前工作目录 |

---

## 10. MCP (Model Context Protocol)

### 10.1 MCP 命令

| 命令 | 功能 |
|------|------|
| `/mcp list` | 列出已连接的 MCP 服务器 |
| `/mcp add` | 添加 MCP 服务器 |
| `/mcp remove` | 移除 MCP 服务器 |

### 10.2 MCP 工具命名

```
mcp__<server>__<tool>
例如: mcp__memory__create_entities
```

---

## 11. 配置系统

### 11.1 配置层级

| 优先级 | 位置 | 说明 |
|--------|------|------|
| 1 (最高) | `settings.local.json` | 本地配置，gitignore |
| 2 | `.claude/settings.json` | 项目配置，可提交 |
| 3 | `~/.claude/settings.json` | 用户配置 |
| 4 | Managed Policy | 组织管理员配置 |

### 11.2 常用配置项

```json
{
  "model": "claude-3-5-sonnet-20241022",
  "permissionMode": "auto",
  "maxTurns": 100,
  "outputFormat": "text",
  "env": {
    "CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR": "1"
  }
}
```

---

## 12. 环境变量

| 变量 | 说明 |
|------|------|
| `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR` | 保持工作目录 |
| `CLAUDE_CODE_USE_POWERSHELL_TOOL` | 启用 PowerShell |
| `CLAUDE_PROJECT_DIR` | 项目根目录 |
| `CLAUDE_CODE_REMOTE` | 远程环境标识 |
| `CLAUDE_ENV_FILE` | 环境变量文件 |

---

## 13. Keyboard Shortcuts

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+C` | 中断当前操作 |
| `Ctrl+O` | 查看详细输出 |
| `Ctrl+L` | 清屏 |
| `Ctrl+D` | 退出 |
| `Tab` | 自动补全 |

---

## 14. Exit Codes

| Code | 说明 |
|------|------|
| 0 | 成功 |
| 1 | 一般错误 |
| 2 | 阻塞错误（Hook 拒绝） |
| 130 | Ctrl+C 中断 |

---

## 15. Feature Flags

| Flag | 说明 |
|------|------|
| `COORDINATOR_MODE` | 多 Agent 协调模式 |
| `KAIROS` | 助手模式 |
| `VOICE_MODE` | 语音模式 |
| `BG_SESSIONS` | 后台会话 |
| `BRIDGE_MODE` | 远程控制模式 |
| `PROACTIVE` | 主动模式 |

---

## Quick Reference

### 启动 Claude

```bash
# 交互模式
claude

# 指定模型
claude --model sonnet

# 恢复会话
claude --resume

# 非交互
claude --no-input "Your question here"

# 使用 agent
claude --agent Explore "Explore the codebase"
```

### 常用命令

```bash
# Git 操作
/commit          # 创建 commit
/branch           # 分支管理
/diff             # 查看更改
/review           # 代码审查

# 会话管理
/resume           # 恢复会话
/clear            # 清除会话
/compact          # 压缩上下文

# 配置
/config edit      # 编辑配置
/hooks            # 查看 hooks
/skills           # 管理技能
/permissions      # 权限管理

# 开发
/plan             # 进入计划模式
/tasks            # 任务管理
/mcp              # MCP 服务器
```

### Hook 快速上手

```bash
# 创建 hook 目录
mkdir -p .claude/hooks

# 禁止危险命令
echo '#!/bin/bash
command=$(jq -r ".tool_input.command")
[[ "$command" == "rm -rf"* ]] && exit 2
exit 0' > .claude/hooks/block-rm.sh
chmod +x .claude/hooks/block-rm.sh
```

然后在 `.claude/settings.json` 中配置：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "if": "Bash(rm *)",
        "command": ".claude/hooks/block-rm.sh"
      }]
    }]
  }
}
```
