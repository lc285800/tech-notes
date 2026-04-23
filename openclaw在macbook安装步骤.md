# OpenClaw 在 MacBook 上的安装步骤

本文档记录一次在 MacBook 上安装 OpenClaw，并把默认 GPT 模型配置为 `openai-codex/gpt-5.4` 的完整流程。

适合对象：不熟悉命令行的 MacBook 用户。你可以按顺序复制命令执行。

## 目标

安装完成后，你应该能做到：

- 在终端直接运行 `openclaw`
- OpenClaw 版本正常显示
- 默认模型配置为 `openai-codex/gpt-5.4`
- OpenClaw 能识别 OpenAI Codex OAuth 登录凭据
- OpenClaw Gateway 可以作为 macOS 后台服务运行
- 可以在浏览器打开 OpenClaw 网页控制面板

## 一、打开终端

1. 打开 Launchpad。
2. 搜索“终端”或 “Terminal”。
3. 打开终端窗口。

后面的命令都在终端里执行。

## 二、创建安装目录

先准备一个工作目录：

```sh
mkdir -p ~/Documents/openclaw
cd ~/Documents/openclaw
```

确认当前目录：

```sh
pwd
```

正常会看到类似：

```text
/Users/你的用户名/Documents/openclaw
```

## 三、先尝试官方安装器

OpenClaw 官方安装命令如下：

```sh
curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method npm --no-onboard
```

说明：

- `curl`：下载官方安装脚本。
- `--install-method npm`：使用 npm 方式安装 OpenClaw。
- `--no-onboard`：先跳过交互式引导，后面我们手动配置模型。

如果这一步直接成功，可以跳到“八、配置模型”。

## 四、如果提示 npm 不存在

本次安装时，Mac 上能找到 Node.js，但没有 `npm`。表现为类似：

```text
env: npm: No such file or directory
```

这种情况下，可以在用户目录安装一份独立 Node.js。这样不需要管理员权限，也不会改系统目录。

执行：

```sh
mkdir -p ~/.local/openclaw-node ~/.local/bin

NODE_VERSION=v24.11.1
ARCH=arm64
TMP=$(mktemp -d)

curl -fsSL --retry 3 "https://nodejs.org/dist/${NODE_VERSION}/node-${NODE_VERSION}-darwin-${ARCH}.tar.gz" -o "$TMP/node.tgz"
tar -xzf "$TMP/node.tgz" -C "$TMP"

rm -rf ~/.local/openclaw-node/node
mv "$TMP/node-${NODE_VERSION}-darwin-${ARCH}" ~/.local/openclaw-node/node

ln -sf ~/.local/openclaw-node/node/bin/node ~/.local/bin/node
ln -sf ~/.local/openclaw-node/node/bin/npm ~/.local/bin/npm
ln -sf ~/.local/openclaw-node/node/bin/npx ~/.local/bin/npx
```

验证 Node.js 和 npm：

```sh
~/.local/bin/node --version
~/.local/bin/npm --version
```

正常会看到类似：

```text
v24.11.1
11.6.2
```

## 五、把用户目录命令加入 PATH

为了以后可以直接输入 `openclaw`，需要让终端知道 `~/.local/bin` 这个目录。

执行：

```sh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zprofile
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
export PATH="$HOME/.local/bin:$PATH"
```

说明：

- `~/.zprofile`：Mac 终端登录时会读它。
- `~/.zshrc`：交互式 zsh 终端会读它。
- 最后一行 `export PATH=...` 是让当前窗口立刻生效。

## 六、如果提示 git 或 Xcode Command Line Tools 问题

本次安装时，npm 需要从 GitHub 拉一个依赖，但系统 `git` 不可用，报错类似：

```text
xcode-select: note: No developer tools were found, requesting install.
```

如果你愿意用系统方式，可以按弹窗安装 Apple Command Line Tools。

如果你想继续使用“不改系统目录”的方式，可以安装一份用户态 git。

先下载 micromamba：

```sh
mkdir -p ~/.local/micromamba-bin

TMP=$(mktemp -d)
curl -fsSL --retry 3 https://micro.mamba.pm/api/micromamba/osx-arm64/latest -o "$TMP/micromamba.tar.bz2"
tar -xjf "$TMP/micromamba.tar.bz2" -C "$TMP" bin/micromamba

mv "$TMP/bin/micromamba" ~/.local/micromamba-bin/micromamba
chmod +x ~/.local/micromamba-bin/micromamba
```

再用 micromamba 安装 git：

```sh
export MAMBA_ROOT_PREFIX="$HOME/.local/micromamba"

~/.local/micromamba-bin/micromamba create -y -n openclaw-tools -c conda-forge git ca-certificates openssh
```

验证 git：

```sh
~/.local/micromamba/envs/openclaw-tools/bin/git --version
```

正常会看到类似：

```text
git version 2.53.0
```

把用户态 git 加入当前终端 PATH：

```sh
export PATH="$HOME/.local/micromamba/envs/openclaw-tools/bin:$HOME/.local/bin:$PATH"
```

为了避免 npm 使用 SSH 方式访问 GitHub，建议把 GitHub SSH 地址重写成 HTTPS：

```sh
git config --global url."https://github.com/".insteadOf "ssh://git@github.com/"
git config --global --add url."https://github.com/".insteadOf "git@github.com:"
```

## 七、安装 OpenClaw

设置 npm 全局安装目录为用户目录：

```sh
npm config set prefix "$HOME/.local"
```

安装 OpenClaw：

```sh
npm --loglevel error --no-fund --no-audit \
  --fetch-retries=5 \
  --fetch-retry-mintimeout=20000 \
  --fetch-retry-maxtimeout=120000 \
  --fetch-timeout=600000 \
  install -g openclaw@latest
```

如果网络不好，可能会出现：

```text
npm error network read ETIMEDOUT
```

这种情况通常不是配置错了，是下载超时。再次执行上面的安装命令即可，npm 会复用已经下载过的缓存。

如果安装卡在 OpenClaw 的 postinstall 阶段很久，可以跳过 bundled plugin postinstall 后再安装一次：

```sh
OPENCLAW_DISABLE_BUNDLED_PLUGIN_POSTINSTALL=1 npm --loglevel error --no-fund --no-audit \
  --fetch-retries=5 \
  --fetch-timeout=600000 \
  install -g openclaw@latest
```

本次安装最终成功输出类似：

```text
added 795 packages in 2m
/Users/evanliu/.local/bin/openclaw
OpenClaw 2026.4.15 (041266a)
```

验证 OpenClaw：

```sh
openclaw --version
```

正常会看到类似：

```text
OpenClaw 2026.4.15 (041266a)
```

## 八、配置 GPT 模型为 ChatGPT Codex 5.4

执行：

```sh
openclaw models set openai-codex/gpt-5.4
```

成功后会看到类似：

```text
Updated ~/.openclaw/openclaw.json
Default model: openai-codex/gpt-5.4
```

## 九、验证模型配置

执行：

```sh
openclaw models status --plain
```

重点看这几行：

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

也可以验证配置文件：

```sh
openclaw config validate
```

正常输出：

```text
Config valid: ~/.openclaw/openclaw.json
```

## 十、安装并启动 Gateway 服务

如果你希望 OpenClaw 在本机持续提供网页控制面板和本地 Gateway，建议把 Gateway 安装成 macOS 用户级 LaunchAgent。

### 1. 先补上 `gateway.mode`

本次机器上第一次启动 Gateway 时，OpenClaw 提示：

```text
Gateway start blocked: existing config is missing gateway.mode.
```

因此需要先写入：

```sh
openclaw config set gateway.mode local
```

正常会看到类似：

```text
Updated gateway.mode. Restart the gateway to apply.
```

### 2. 安装 Gateway 为 LaunchAgent

执行：

```sh
openclaw gateway install --force
```

正常输出类似：

```text
Installed LaunchAgent: /Users/你的用户名/Library/LaunchAgents/ai.openclaw.gateway.plist
Logs: /Users/你的用户名/.openclaw/logs/gateway.log
```

说明：

- `LaunchAgent` 是 macOS 用户级后台服务
- 这样以后登录系统后，OpenClaw Gateway 更容易被统一管理
- `--force` 表示如果之前装过，就覆盖更新

### 3. 启动 Gateway 服务

执行：

```sh
openclaw gateway start
```

如果你刚修改过配置，也可以直接执行：

```sh
openclaw gateway restart
```

### 4. 检查 Gateway 是否真的运行起来了

执行：

```sh
openclaw gateway status
```

正常结果应重点包含：

```text
Service: LaunchAgent (loaded)
Runtime: running
RPC probe: ok
Listening: 127.0.0.1:18789
Dashboard: http://127.0.0.1:18789/
```

也可以进一步检查健康状态：

```sh
openclaw gateway health
```

正常输出类似：

```text
Gateway Health
OK
```

## 十一、打开网页控制面板

Gateway 正常运行后，执行：

```sh
openclaw dashboard
```

OpenClaw 会：

- 自动生成或读取本地登录 token
- 把带 token 的 Dashboard URL 复制到剪贴板
- 自动在默认浏览器中打开控制面板

本次机器上的实际输出类似：

```text
Dashboard URL: http://127.0.0.1:18789/#token=...
Copied to clipboard.
Opened in your browser. Keep that tab to control OpenClaw.
```

如果你只想打印地址、不自动打开浏览器，可以执行：

```sh
openclaw dashboard --no-open
```

## 十二、最终配置文件长什么样

本次生成的配置文件是：

```text
~/.openclaw/openclaw.json
```

内容类似：

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
  },
  "gateway": {
    "mode": "local"
  },
  "meta": {
    "lastTouchedVersion": "2026.4.15",
    "lastTouchedAt": "2026-04-21T16:53:42.257Z"
  }
}
```

其中最关键的是：

```json
"primary": "openai-codex/gpt-5.4"
```

以及：

```json
"mode": "local"
```

## 十三、新开终端后再检查一次

关闭当前终端，重新打开一个新终端。

执行：

```sh
command -v openclaw
openclaw --version
openclaw models status --plain
openclaw gateway status
```

正常结果应该包含：

```text
/Users/你的用户名/.local/bin/openclaw
OpenClaw 2026.4.15
Default       : openai-codex/gpt-5.4
Runtime: running
```

## 十四、常见问题

### 1. command not found: openclaw

说明 `~/.local/bin` 没有进入 PATH。

执行：

```sh
export PATH="$HOME/.local/bin:$PATH"
```

然后再试：

```sh
openclaw --version
```

如果能用了，再确认配置文件里有 PATH：

```sh
grep '.local/bin' ~/.zprofile ~/.zshrc
```

如果没有，补上：

```sh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zprofile
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
```

### 2. npm: No such file or directory

说明没有 npm。

按本文“四、如果提示 npm 不存在”安装用户目录 Node.js。

### 3. xcode-select: No developer tools were found

说明系统 git 需要 Apple Command Line Tools。

你可以：

- 按 macOS 弹窗安装 Command Line Tools
- 或按本文“六、如果提示 git 或 Xcode Command Line Tools 问题”安装用户态 git

### 4. npm network ETIMEDOUT

说明网络超时。

重新执行安装命令即可：

```sh
npm --loglevel error --no-fund --no-audit \
  --fetch-retries=5 \
  --fetch-retry-mintimeout=20000 \
  --fetch-retry-maxtimeout=120000 \
  --fetch-timeout=600000 \
  install -g openclaw@latest
```

### 5. 默认模型不是 openai-codex/gpt-5.4

重新设置：

```sh
openclaw models set openai-codex/gpt-5.4
```

再验证：

```sh
openclaw models status --plain
```

### 6. Gateway status 提示 missing gateway.mode

说明配置文件里还没有：

```json
"gateway": {
  "mode": "local"
}
```

执行：

```sh
openclaw config set gateway.mode local
openclaw gateway restart
```

然后再检查：

```sh
openclaw gateway status
```

### 7. Gateway 服务显示 running，但端口没监听

这通常意味着服务进程起来了，但启动过程中又被配置拦下。

先看状态：

```sh
openclaw gateway status
```

如果看到类似：

```text
Gateway port 18789 is not listening
Last gateway error: ... missing gateway.mode
```

按上一条补上：

```sh
openclaw config set gateway.mode local
openclaw gateway restart
```

### 8. 网页控制面板没有自动打开

先手动打印地址：

```sh
openclaw dashboard --no-open
```

再把输出中的 URL 复制到浏览器打开。

如果仍然打不开，先检查：

```sh
openclaw gateway health
openclaw gateway status
```

## 十五、本次机器上的实际结果

本次 MacBook 上安装后的实际结果：

```text
openclaw 路径: /Users/evanliu/.local/bin/openclaw
OpenClaw 版本: OpenClaw 2026.4.15 (041266a)
配置文件: /Users/evanliu/.openclaw/openclaw.json
默认模型: openai-codex/gpt-5.4
OAuth 提供方: openai-codex
配置校验: Config valid
Gateway 服务: LaunchAgent loaded
Gateway 地址: http://127.0.0.1:18789/
Gateway 健康检查: OK
网页控制面板: 已可在浏览器打开
```

到这里，OpenClaw 已经安装完成，GPT 模型、Gateway 服务和网页控制面板也已经配置完成.
