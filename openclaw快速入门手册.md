# OpenClaw 快速入门手册

本文是一份面向新手的 OpenClaw 上手指南，适合已经安装好 OpenClaw，想快速完成检查、配置、启动和第一次对话的用户。

如果你还没有安装 OpenClaw，请先阅读：[OpenClaw 在 MacBook 上的安装步骤](./openclaw在macbook安装步骤.md)。

## 你会完成什么

读完并执行本文命令后，你应该能做到：

- 确认 `openclaw` 命令已经可用
- 检查 OpenClaw 版本和配置文件
- 设置默认模型为 `openai-codex/gpt-5.4`
- 启动本地 Gateway
- 通过终端 UI 或命令行向 agent 发送第一条消息
- 使用常见诊断命令排查问题

## 一、确认命令可用

打开终端，执行：

```sh
command -v openclaw
openclaw --version
```

正常情况下会看到类似：

```text
/Users/你的用户名/.local/bin/openclaw
OpenClaw 2026.4.15 (041266a)
```

如果提示：

```text
command not found: openclaw
```

说明 `openclaw` 所在目录没有加入 `PATH`。可以先临时修复：

```sh
export PATH="$HOME/.local/bin:$PATH"
```

再重新检查：

```sh
openclaw --version
```

如果这样能成功，建议把 PATH 写入 shell 配置：

```sh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zprofile
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
```

## 二、查看帮助

OpenClaw 的主命令帮助：

```sh
openclaw --help
```

查看某个子命令帮助：

```sh
openclaw models --help
openclaw gateway --help
openclaw agent --help
openclaw tui --help
```

常用命令可以先记住这几个：

| 命令 | 作用 |
| --- | --- |
| `openclaw --version` | 查看 OpenClaw 版本 |
| `openclaw config file` | 查看当前配置文件路径 |
| `openclaw config validate` | 校验配置文件是否有效 |
| `openclaw models status --plain` | 查看模型配置和认证状态 |
| `openclaw gateway run` | 在前台启动本地 Gateway |
| `openclaw gateway status` | 查看 Gateway 状态 |
| `openclaw tui` | 打开终端聊天界面 |
| `openclaw agent --message "..."` | 从命令行发送一轮 agent 请求 |
| `openclaw status --all` | 输出完整诊断信息 |
| `openclaw doctor` | 运行健康检查和修复建议 |

## 三、初始化本地配置

如果你是第一次使用，可以先执行：

```sh
openclaw setup --wizard
```

它会通过交互式向导初始化本地配置和 agent 工作目录。

如果你希望使用默认值快速初始化，也可以执行：

```sh
openclaw setup --non-interactive
```

默认 agent 工作目录通常在：

```text
~/.openclaw/workspace
```

查看当前配置文件路径：

```sh
openclaw config file
```

校验配置：

```sh
openclaw config validate
```

正常会看到类似：

```text
Config valid: ~/.openclaw/openclaw.json
```

## 四、配置默认模型

设置默认模型为 ChatGPT Codex 5.4：

```sh
openclaw models set openai-codex/gpt-5.4
```

检查模型配置：

```sh
openclaw models status --plain
```

重点看这几项：

```text
Default       : openai-codex/gpt-5.4
Configured models (1): openai-codex/gpt-5.4
Providers w/ OAuth/tokens (1): openai-codex (1)
```

如果看到：

```text
openai-codex:default ok
```

说明 OpenAI Codex OAuth 凭据可用。

## 五、启动 Gateway

OpenClaw 的很多能力通过本地 Gateway 提供。第一次上手建议先以前台方式启动，方便观察日志。

执行：

```sh
openclaw gateway run
```

如果默认端口被占用，可以强制清理旧监听后启动：

```sh
openclaw gateway --force run
```

如果你想指定端口：

```sh
openclaw gateway --port 18789 run
```

保持这个终端窗口不要关闭。然后新开一个终端窗口，继续执行后面的命令。

检查 Gateway 状态：

```sh
openclaw gateway status
```

也可以直接探测健康状态：

```sh
openclaw gateway health
```

## 六、第一次对话

### 方式 1：打开终端 UI

在新终端中执行：

```sh
openclaw tui
```

进入界面后，直接输入你的问题即可。

也可以启动时顺便发送第一条消息：

```sh
openclaw tui --message "你好，帮我用一句话介绍 OpenClaw。"
```

### 方式 2：使用命令行发送一轮请求

执行：

```sh
openclaw agent --message "你好，帮我用一句话介绍 OpenClaw。"
```

如果想看结构化输出：

```sh
openclaw agent --message "列出三个常用命令。" --json
```

如果想指定思考强度：

```sh
openclaw agent --message "帮我分析这个项目的 README。" --thinking medium
```

可选思考强度包括：

```text
off | minimal | low | medium | high | xhigh
```

## 七、常用工作流

### 1. 启动本地服务并进入聊天

终端 A：

```sh
openclaw gateway run
```

终端 B：

```sh
openclaw tui
```

这是最适合新手的方式：一个窗口看服务日志，一个窗口聊天。

### 2. 快速检查当前状态

```sh
openclaw status
openclaw models status --plain
openclaw config validate
```

如果需要完整诊断：

```sh
openclaw status --all
```

### 3. 查看和搜索文档

```sh
openclaw docs
```

也可以直接访问官方 CLI 文档：

```text
https://docs.openclaw.ai/cli
```

### 4. 管理模型

查看模型帮助：

```sh
openclaw models --help
```

查看当前状态：

```sh
openclaw models status --plain
```

设置默认文本模型：

```sh
openclaw models set openai-codex/gpt-5.4
```

设置默认图像模型：

```sh
openclaw models set-image <模型名>
```

## 八、配置文件在哪里

默认配置文件：

```text
~/.openclaw/openclaw.json
```

可以用命令确认：

```sh
openclaw config file
```

本文涉及的模型配置大致长这样：

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "openai-codex/gpt-5.4"
      },
      "models": {
        "openai-codex/gpt-5.4": {}
      }
    }
  }
}
```

最关键的是：

```json
"primary": "openai-codex/gpt-5.4"
```

## 九、进阶入口

OpenClaw 的功能比较多，熟悉基本用法后，可以继续探索这些命令：

| 命令 | 适合场景 |
| --- | --- |
| `openclaw channels` | 配置 Telegram、Discord、Slack、WhatsApp 等聊天渠道 |
| `openclaw message` | 发送、读取和管理消息 |
| `openclaw agents` | 管理多个隔离 agent |
| `openclaw skills` | 查看和管理可用技能 |
| `openclaw plugins` | 管理插件和扩展 |
| `openclaw cron` | 管理定时任务 |
| `openclaw mcp` | 管理 MCP 配置和桥接 |
| `openclaw sessions` | 查看已保存的会话 |
| `openclaw logs` | 查看 Gateway 日志 |
| `openclaw backup` | 备份本地 OpenClaw 状态 |
| `openclaw update` | 更新 OpenClaw |

查看某个命令的详细说明：

```sh
openclaw <命令> --help
```

例如：

```sh
openclaw channels --help
openclaw plugins --help
openclaw update --help
```

## 十、常见问题

### 1. `openclaw` 命令找不到

先执行：

```sh
export PATH="$HOME/.local/bin:$PATH"
```

再执行：

```sh
openclaw --version
```

如果恢复正常，把 PATH 写入配置文件：

```sh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zprofile
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
```

### 2. 配置文件无效

执行：

```sh
openclaw config validate
```

如果提示配置错误，可以先运行：

```sh
openclaw doctor
```

需要自动应用推荐修复时：

```sh
openclaw doctor --repair
```

### 3. Gateway 启动失败

先查看状态：

```sh
openclaw gateway status
```

如果是端口占用，可以尝试：

```sh
openclaw gateway --force run
```

或者换一个端口：

```sh
openclaw gateway --port 19001 run
```

### 4. 模型不是预期的 `openai-codex/gpt-5.4`

重新设置：

```sh
openclaw models set openai-codex/gpt-5.4
```

再检查：

```sh
openclaw models status --plain
```

### 5. OAuth 或 token 不可用

先查看模型认证状态：

```sh
openclaw models status --plain
```

再运行健康检查：

```sh
openclaw doctor
```

如果你刚刚登录或修改过凭据，建议重启 Gateway：

```sh
openclaw gateway restart
```

如果你是以前台方式运行 Gateway，可以在原来的 Gateway 窗口按 `Ctrl+C` 停止，再重新执行：

```sh
openclaw gateway run
```

## 十一、建议的新手路线

第一次使用时，建议按这个顺序走：

```sh
openclaw --version
openclaw setup --wizard
openclaw models set openai-codex/gpt-5.4
openclaw models status --plain
openclaw config validate
openclaw gateway run
```

然后新开一个终端：

```sh
openclaw tui --message "你好，帮我检查一下 OpenClaw 是否已经可以正常使用。"
```

如果这一步能正常收到回复，就说明 OpenClaw 的基础使用链路已经跑通。
