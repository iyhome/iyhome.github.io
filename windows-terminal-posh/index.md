# 把 Windows 终端整成 Linux 的样子：pwsh7 + oh-my-posh + scoop


在 Linux 上用顺了 zsh + Starship，回到 Windows 总觉得终端少了点味道。本文记录把 Windows 终端补齐成「同款体验」的过程：PowerShell 7 转正、oh-my-posh 定制、scoop 装工具，以及几个 Windows 特有的坑。

## 1. PowerShell 7 转正

系统自带的是 Windows PowerShell 5.1。装了 PowerShell 7 后，把它设为默认：

- Windows Terminal 默认配置文件 = PowerShell 7，隐藏 5.1
- Windows Terminal 设为系统默认终端应用
- VS Code 默认终端 = `pwsh.exe -NoLogo`
- 删掉 5.1 的开始菜单快捷方式（引擎仍是系统组件，不卸载）

## 2. oh-my-posh：agnoster 定制版

主题用 `agnoster`，但原版提示符里用户名/主机名太长。定制要点：

- 复制内置主题为本地 JSON，去掉 `session`（用户名/主机名）段
- profile 里加载：`oh-my-posh init pwsh --config <path> | Invoke-Expression`

```powershell
# $PROFILE
oh-my-posh init pwsh --config "$HOME\Documents\oh-my-posh-agnoster.json" | Invoke-Expression
Import-Module posh-git
Import-Module Terminal-Icons
```

`posh-git` 给 git 状态，`Terminal-Icons` 给文件图标。

## 3. scoop：轻量包管理

winget 之外再装 scoop，用于拿 CLI 工具：

```powershell
scoop install ripgrep fd bat eza fzf zoxide bottom lazygit lazydocker tldr jq yt-dlp mpv ffmpeg
```

**坑**：`irm get.scoop.sh | iex` 在 `pwsh -NoProfile` 下会报 Security error。改成「先把脚本下下来再执行」即可。

装完记得**新开终端**——`scoop\shims` 是加到用户 PATH 的，当前会话不会自动刷新。

## 4. 执行策略

npm 的 `.ps1` 脚本（如 `npm.ps1`）会被执行策略拦住。设成 RemoteSigned：

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

## 5. 让两端观感一致

- 配色：Catppuccin 主题，和 Linux 端统一
- 字体：JetBrainsMono Nerd Font（Nerd Font 才显示图标/箭头）
- 提示符：如果更想要跨平台一致，可以用 Starship（`~/.config/starship.toml` bash/zsh/pwsh 通用），二选一

## 6. 小结

| 项 | 做法 |
|---|---|
| Shell | PowerShell 7 设为默认（WT/VS Code） |
| 提示符 | oh-my-posh agnoster 定制去用户名 |
| 模块 | posh-git + Terminal-Icons |
| 包管理 | scoop（注意 `irm|iex` 的 Security error） |
| 脚本 | 执行策略 RemoteSigned |

核心体会：**Windows 终端不是不能像 Linux，而是每个环节都埋了一个「默认值不一样」的小坑**——curl 是别名、PATH 要新开会话、执行策略默认拦脚本。把这几处填平，体验就接上了。

