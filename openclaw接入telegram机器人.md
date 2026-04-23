# OpenClaw 接入 Telegram 机器人

本文记录一次把 Telegram Bot 接入当前这台 MacBook 上的 OpenClaw，并把私聊策略改成可直接使用的完整过程。

适合场景：

- 你已经在 Telegram 里用 BotFather 创建好了 bot
- 你已经拿到了 bot token
- 你希望当前这台机器上的 OpenClaw 直接通过 Telegram 收消息
- 你不想每次都先做 pairing 才能私聊 bot

## 目标

完成后，你应该能做到：

- OpenClaw 成功加载 Telegram channel
- Telegram bot 正常连接到当前机器上的 OpenClaw
- 你可以直接私聊机器人，不再先走 pairing code
- OpenClaw Gateway 维持正常运行

## 一、你需要准备什么

至少准备这两项：

- Telegram 机器人用户名，例如：`@chuan123_bot`
- Telegram bot token

bot token 通常来自 BotFather，格式类似：

```text
1234567890:AA......
```

## 二、把 Telegram bot 加到 OpenClaw

在终端执行：

```sh
openclaw channels add --channel telegram --name chuan123_bot --token '你的bot token'
```

说明：

- `--channel telegram`：表示新增的是 Telegram 消息入口
- `--name chuan123_bot`：只是本地配置里显示的名字
- `--token`：Telegram bot token

正常会看到类似：

```text
Added Telegram account "default".
```

这表示 OpenClaw 已经把 Telegram 账号配置写入本地配置文件。

## 三、把 Telegram 私聊策略改成直接可用

默认情况下，OpenClaw 的 Telegram 私聊策略可能是：

```text
dmPolicy: pairing
```

这意味着你第一次私聊 bot 时，会收到 pairing code，然后还要在电脑上再执行一次批准命令。

如果你希望当前这台机器作为唯一运行节点，并且你自己私聊 bot 时可以直接用，最简单的做法是把它改成：

```sh
openclaw config set channels.telegram.dmPolicy open
```

注意这里必须是：

```text
open
```

不是 `allow`。本次实际排查时，`allow` 会触发配置校验失败，因为合法值只有：

```text
pairing
allowlist
open
disabled
```

## 四、重启 Gateway 让配置生效

执行：

```sh
openclaw gateway restart
```

正常会看到类似：

```text
Restarted LaunchAgent: gui/501/ai.openclaw.gateway
```

如果你的 Gateway 还没有安装成服务，可以先参考安装文档中的 Gateway 小节。

## 五、检查配置是否已经生效

执行：

```sh
openclaw config get channels.telegram
```

正常应看到类似：

```json
{
  "name": "chuan123_bot",
  "enabled": true,
  "botToken": "__OPENCLAW_REDACTED__",
  "dmPolicy": "open",
  "groupPolicy": "allowlist"
}
```

重点看：

```json
"enabled": true
"dmPolicy": "open"
```

这表示：

- Telegram channel 已经启用
- 私聊消息可以直接进入 OpenClaw

## 六、检查 Gateway 是否正常

执行：

```sh
openclaw gateway health
openclaw gateway status
```

正常结果应包含类似：

```text
Gateway Health
OK
```

以及：

```text
Runtime: running
RPC probe: ok
Listening: 127.0.0.1:18789
```

## 七、检查 Telegram bot 本身是否可用

如果你想确认 bot token 本身没问题，可以直接调用 Telegram Bot API：

```sh
curl -sS 'https://api.telegram.org/bot你的bot token/getMe'
```

正常结果会返回 bot 身份信息，例如：

```json
{
  "ok": true,
  "result": {
    "id": 1234567890,
    "is_bot": true,
    "username": "chuan123_bot"
  }
}
```

这一步主要用于区分：

- 是 OpenClaw 配置问题
- 还是 Telegram token 本身就无效

## 八、最终验收

打开 Telegram，找到你的机器人，例如：

```text
@chuan123_bot
```

直接给它发一条普通消息，例如：

```text
你好
```

或者：

```text
你能收到我这条消息吗？
```

如果机器人已经正常回复，就说明接入成功。

本次实际结果是：

- Telegram bot 已成功接入当前这台 MacBook 上的 OpenClaw
- `dmPolicy` 已改为 `open`
- Gateway 健康检查返回 `OK`
- Telegram 私聊已能正常收到和回复消息

## 九、为什么之前会出现 pairing code

因为默认配置里：

```json
"dmPolicy": "pairing"
```

这会导致你第一次私聊 bot 时，OpenClaw 返回类似：

```text
OpenClaw: access not configured.
Pairing code: XXXXXXXX
```

并提示你执行：

```sh
openclaw pairing approve telegram <配对码>
```

这不是报错，而是默认安全策略。

如果当前 bot 只由你自己使用，并且只跑在当前这台机器上，把它改成 `open` 往往更省事。

## 十、常见问题

### 1. `dmPolicy` 设成 `allow` 失败

说明这个字段不接受 `allow`。

正确命令是：

```sh
openclaw config set channels.telegram.dmPolicy open
```

### 2. Gateway 重启后暂时显示 probe failed

刚重启完的几秒内，Gateway 可能还在 warm-up。

稍等几秒再执行：

```sh
openclaw gateway health
openclaw gateway status
```

### 3. Telegram 里发消息没有回复

依次检查：

```sh
openclaw config get channels.telegram
openclaw gateway health
openclaw gateway status
```

再检查 bot token 本身：

```sh
curl -sS 'https://api.telegram.org/bot你的bot token/getMe'
```

### 4. 机器人能启动，但群里不能直接用

因为当前配置通常还是：

```json
"groupPolicy": "allowlist"
```

这表示群组不是默认全开放，要额外做 allowlist 管理。

## 十一、本次机器上的实际状态

本次接入完成后，当前机器上的实际状态是：

```text
Telegram channel: enabled
Bot name: chuan123_bot
dmPolicy: open
groupPolicy: allowlist
Gateway Health: OK
Gateway: running
Telegram 私聊收发: 正常
```

到这里，OpenClaw 通过 Telegram 机器人接收消息的链路已经跑通。
