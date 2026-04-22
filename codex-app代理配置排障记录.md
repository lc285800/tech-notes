# Codex App 代理配置排障记录

## 背景

在 macOS 上使用 Codex App 时，每次开始对话都会反复提示 `reconnect`，通常需要重连 4 到 5 次后才能正常回复。Clash Verge 已开启系统代理，但 Codex App 仍然表现为连接不稳定。

本次排障目标：

- 让 Codex App 默认自动走 Clash 代理。
- 确保退出并重新打开 Codex App 后配置仍然生效。
- 通过命令行验证代理链路可用。

## 环境信息

- 系统：macOS
- 代理客户端：Clash Verge
- Clash 代理端口：`127.0.0.1:7897`
- Codex App 进程：`/Applications/Codex.app`
- 持久化配置文件：

```text
/Users/evanliu/Library/LaunchAgents/com.openai.codex.proxy-env.plist
```

## 问题现象

系统代理已经在 Clash Verge 中开启，`scutil --proxy` 可以看到 macOS 系统代理配置：

```text
HTTPEnable : 1
HTTPProxy : 127.0.0.1
HTTPPort : 7897
HTTPSEnable : 1
HTTPSProxy : 127.0.0.1
HTTPSPort : 7897
SOCKSEnable : 1
SOCKSProxy : 127.0.0.1
SOCKSPort : 7897
```

但是 Codex App 进程启动时没有继承这些代理环境变量：

```text
HTTP_PROXY
HTTPS_PROXY
ALL_PROXY
NO_PROXY
```

因此 Codex App 可能仍然尝试直连 OpenAI 服务，导致反复 `reconnect`。

## 原因分析

macOS GUI App 通常由 LaunchServices / launchd 启动，不一定会读取 shell 配置文件，例如：

- `~/.zshrc`
- `~/.zprofile`
- `~/.bash_profile`

所以即使终端里配置了代理环境变量，Codex App 从 Dock、Finder 或应用启动器打开时，也未必能继承。

本次修复选择在用户级 `launchd` 环境中设置代理变量，并通过 `LaunchAgent` 做持久化。这样 Codex App 重启后可以从 GUI 启动环境中继承代理配置。

## 修复方案

创建用户级 LaunchAgent：

```text
/Users/evanliu/Library/LaunchAgents/com.openai.codex.proxy-env.plist
```

内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.openai.codex.proxy-env</string>

  <key>ProgramArguments</key>
  <array>
    <string>/bin/sh</string>
    <string>-c</string>
    <string>launchctl setenv HTTP_PROXY http://127.0.0.1:7897; launchctl setenv HTTPS_PROXY http://127.0.0.1:7897; launchctl setenv ALL_PROXY socks5://127.0.0.1:7897; launchctl setenv NO_PROXY 127.0.0.1,localhost,::1,*.local</string>
  </array>

  <key>RunAtLoad</key>
  <true/>
</dict>
</plist>
```

立即写入当前用户的 launchd 环境：

```bash
launchctl setenv HTTP_PROXY http://127.0.0.1:7897
launchctl setenv HTTPS_PROXY http://127.0.0.1:7897
launchctl setenv ALL_PROXY socks5://127.0.0.1:7897
launchctl setenv NO_PROXY '127.0.0.1,localhost,::1,*.local'
```

加载 LaunchAgent：

```bash
launchctl bootout gui/$(id -u) "$HOME/Library/LaunchAgents/com.openai.codex.proxy-env.plist" 2>/dev/null
launchctl bootstrap gui/$(id -u) "$HOME/Library/LaunchAgents/com.openai.codex.proxy-env.plist"
launchctl kickstart -k gui/$(id -u)/com.openai.codex.proxy-env
```

## 验证记录

检查 plist 格式：

```bash
plutil -lint "$HOME/Library/LaunchAgents/com.openai.codex.proxy-env.plist"
```

结果：

```text
/Users/evanliu/Library/LaunchAgents/com.openai.codex.proxy-env.plist: OK
```

检查 launchd 中的环境变量：

```bash
launchctl getenv HTTP_PROXY
launchctl getenv HTTPS_PROXY
launchctl getenv ALL_PROXY
launchctl getenv NO_PROXY
```

结果：

```text
http://127.0.0.1:7897
http://127.0.0.1:7897
socks5://127.0.0.1:7897
127.0.0.1,localhost,::1,*.local
```

检查 LaunchAgent 状态：

```bash
launchctl print gui/$(id -u)/com.openai.codex.proxy-env
```

关键结果：

```text
path = /Users/evanliu/Library/LaunchAgents/com.openai.codex.proxy-env.plist
state = not running
runs = 2
last exit code = 0
properties = runatload | inferred program
```

这里 `state = not running` 是正常的，因为这个 LaunchAgent 是一次性任务：登录时运行 `launchctl setenv`，执行完就退出。`last exit code = 0` 表示执行成功。

显式通过代理访问 OpenAI API：

```bash
curl -I -x http://127.0.0.1:7897 --connect-timeout 8 --max-time 15 https://api.openai.com/v1/models
```

结果包含：

```text
HTTP/1.1 200 Connection established
HTTP/2 401
www-authenticate: Bearer realm="OpenAI API"
```

`401` 是正常结果，表示请求已经到达 OpenAI API，只是没有携带 API key。关键是出现了 `HTTP/1.1 200 Connection established`，说明代理链路正常。

还额外验证了由 launchd 启动的测试进程可以继承代理变量，并且不显式指定 `-x` 参数时也能通过代理访问 OpenAI API。

## 最终验收

退出 Codex App 后重新打开，测试对话可以快速回复，不再反复出现 `reconnect`。说明 Codex App 已经从新的 GUI 启动环境中继承代理配置，本次修复生效。

## 后续注意事项

如果 Clash Verge 的代理端口发生变化，例如从 `7897` 改成 `7890`，需要同步修改：

```text
/Users/evanliu/Library/LaunchAgents/com.openai.codex.proxy-env.plist
```

并重新加载：

```bash
launchctl bootout gui/$(id -u) "$HOME/Library/LaunchAgents/com.openai.codex.proxy-env.plist" 2>/dev/null
launchctl bootstrap gui/$(id -u) "$HOME/Library/LaunchAgents/com.openai.codex.proxy-env.plist"
launchctl kickstart -k gui/$(id -u)/com.openai.codex.proxy-env
```

然后完全退出并重新打开 Codex App。
