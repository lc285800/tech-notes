# Mac 配置 GitHub 推送权限

本文记录一次在 Mac 上为本地 Git 仓库配置 GitHub 推送权限，并最终把文档成功推送到远程仓库的完整过程。

适合场景：

- 你已经有一个 GitHub 仓库
- 本地仓库已经配置了 `origin`
- 执行 `git push` 时失败
- 你希望这台 Mac 以后可以直接向 GitHub 推送

## 目标

完成后，你应该能做到：

- 在本机安装 GitHub CLI `gh`
- 用浏览器完成 GitHub 授权
- 让本地 `git push origin main` 正常工作
- 遇到 `fetch first` 时知道怎么处理

## 一、先确认当前问题是什么

本次机器上的远程仓库是：

```sh
git remote -v
```

结果类似：

```text
origin  https://github.com/lc285800/tech-notes.git (fetch)
origin  https://github.com/lc285800/tech-notes.git (push)
```

本次第一次推送时报错：

```text
fatal: could not read Username for 'https://github.com': Device not configured
```

这表示：

- 本地仓库使用的是 GitHub HTTPS 地址
- 这台 Mac 上的 `git` 没有可用的 GitHub 登录态
- 所以 `git push` 无法完成身份认证

这不是单纯“浏览器里登录过 GitHub 网页就行”的问题。  
终端里的 `git push` 需要单独配置认证。

## 二、检查是否已安装 GitHub CLI

先检查：

```sh
gh --version
```

如果提示：

```text
command not found: gh
```

说明这台机器还没装 GitHub CLI。

本次机器上就是这种情况。

## 三、用用户目录方式安装 gh

为了不依赖 Homebrew，也不改系统目录，可以把 `gh` 安装到：

```text
~/.local/bin
```

执行：

```sh
mkdir -p ~/.local/bin ~/.local/gh-cli
TMP=$(mktemp -d)

curl -fsSL https://github.com/cli/cli/releases/download/v2.91.0/gh_2.91.0_macOS_arm64.zip -o "$TMP/gh.zip"
unzip -q "$TMP/gh.zip" -d "$TMP"

rm -rf ~/.local/gh-cli/gh_2.91.0_macOS_arm64
mv "$TMP/gh_2.91.0_macOS_arm64" ~/.local/gh-cli/

ln -sf ~/.local/gh-cli/gh_2.91.0_macOS_arm64/bin/gh ~/.local/bin/gh
~/.local/bin/gh --version
```

本次安装成功后输出类似：

```text
gh version 2.91.0 (2026-04-22)
```

如果你的 `PATH` 已经包含 `~/.local/bin`，后面就可以直接用：

```sh
gh
```

## 四、用浏览器完成 GitHub 登录

执行：

```sh
gh auth login --hostname github.com --git-protocol https --web --clipboard
```

过程中通常会先询问：

```text
Authenticate Git with your GitHub credentials? (Y/n)
```

输入：

```text
Y
```

然后会给出一次性验证码，类似：

```text
1985-9C14
```

并提示打开：

```text
https://github.com/login/device
```

此时操作步骤是：

1. 打开网页
2. 输入验证码
3. 登录 GitHub
4. 授权 `GitHub CLI`

授权成功后，终端会输出类似：

```text
✓ Authentication complete.
- gh config set -h github.com git_protocol https
✓ Configured git protocol
✓ Logged in as lc285800
```

## 五、验证登录状态

执行：

```sh
gh auth status
```

正常结果类似：

```text
github.com
  ✓ Logged in to github.com account lc285800 (keyring)
  - Active account: true
  - Git operations protocol: https
```

重点看这几项：

- `Logged in`
- `Active account: true`
- `Git operations protocol: https`

这说明：

- `gh` 已经登录成功
- 认证信息已写入本机 keyring
- 后续 Git 走 HTTPS 协议时可以复用这套授权

## 六、再次执行 git push

登录成功后，再执行：

```sh
git push origin main
```

如果一切顺利，这一步就能直接成功。

但本次机器上，第二次推送遇到的是另一个常见问题：

```text
! [rejected] main -> main (fetch first)
```

这表示：

- 身份认证已经没问题了
- 推送失败的原因不再是“没权限”
- 而是远程分支已经有了新的提交，本地分支落后或发生分叉

这其实是一个正常的 Git 同步问题。

## 七、处理 `fetch first`

先拉取远端引用：

```sh
git fetch origin
```

然后查看本地和远端的提交关系：

```sh
git log --oneline --decorate --graph --max-count=12 main origin/main
```

本次机器上看到的是：

- 本地 `main` 有自己的新提交
- 远端 `origin/main` 也有新提交

因此选择：

```sh
git rebase origin/main
```

## 八、处理 rebase 冲突

本次 rebase 过程中出现了两类情况：

### 1. 初始提交与远端已有历史重叠

这类冲突通常说明：

- 本地最早的一次初始提交
- 与远端仓库已有内容高度重叠

如果确认远端已有那部分内容，可以直接跳过那次旧提交：

```sh
git rebase --skip
```

### 2. 文档内容小冲突

本次还在：

```text
openclaw在macbook安装步骤.md
```

里出现了轻微冲突，实际上只是最后一句的标点不同。

解决方法就是：

1. 手动保留最终版本
2. `git add`
3. 继续 rebase

如果终端里没有默认编辑器，`git rebase --continue` 可能报：

```text
error: Terminal is dumb, but EDITOR unset
```

可以这样继续：

```sh
GIT_EDITOR=true git rebase --continue
```

## 九、rebase 完成后再次推送

当 rebase 完成后，再执行：

```sh
git push origin main
```

本次机器上的最终结果是：

```text
To https://github.com/lc285800/tech-notes.git
   ffa9bec..0d6003c  main -> main
```

说明这台 Mac 已经可以正常向 GitHub 推送。

## 十、最终验收命令

以后你可以用这两条快速确认：

```sh
gh auth status
git push origin main
```

如果第二条返回：

```text
Everything up-to-date
```

就说明：

- 登录态正常
- Git 推送权限正常
- 本地和远端同步正常

## 十一、常见问题

### 1. `gh: command not found`

说明 GitHub CLI 没装，或者 `~/.local/bin` 没进 `PATH`。

先检查：

```sh
echo $PATH
```

如果没有 `~/.local/bin`，补上：

```sh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zprofile
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
export PATH="$HOME/.local/bin:$PATH"
```

### 2. 浏览器已经登录 GitHub，但 `git push` 还是失败

这很常见，因为网页端登录不会自动给终端里的 Git 提供认证。

你仍然需要执行：

```sh
gh auth login
```

### 3. `git push` 报 `fetch first`

这不是权限问题，而是远端分支比本地更新。

处理流程：

```sh
git fetch origin
git rebase origin/main
git push origin main
```

### 4. rebase 时提示 `index.lock`

这通常是上一次 Git 进程残留的锁文件。

确认没有别的 Git 进程在运行后，可以清掉：

```sh
rm -f .git/index.lock
```

然后再继续 rebase。

### 5. `git rebase --continue` 提示 `EDITOR unset`

可以这样继续：

```sh
GIT_EDITOR=true git rebase --continue
```

## 十二、本次机器上的实际结果

本次配置完成后，这台 Mac 的实际状态是：

```text
gh 已安装: 2.91.0
GitHub 登录账号: lc285800
Git 协议: https
认证存储: keyring
git push: 已成功
远程仓库: https://github.com/lc285800/tech-notes.git
```

到这里，这台 Mac 的 GitHub 推送权限已经配置完成，后续可以直接使用 `git push origin main`。
