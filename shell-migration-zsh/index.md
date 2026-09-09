# Omarchy 下的 shell 环境重构：zsh + oh-my-zsh + Starship 排坑


把默认 shell 从 bash 迁到 zsh，本以为是小事，结果牵扯出终端字体、补全引擎、提示符、输入法环境变量一连串问题。本文是一份完整排坑记录，多数问题与「Omarchy 预置了很多 bash 专属配置」有关。

## 1. 缘起：想要 zsh 的体验，先是装了 blesh

想要命令行语法高亮 + 菜单补全。bash 没有原生插件体系，先装了 ble.sh（Bash Line Editor，让 bash 有 fish/zsh 式体验）。能用，但**配色怎么调都别扭**——ble.sh 的高亮 face 用命令行 `ble-face` 设置，face 名要对着源码 `ble/color/defface` 逐个核对，0.3.x 版本才支持哪些 face 很隐蔽。折腾一阵后放弃，转投 zsh 正统方案。

## 2. 安装 zsh + oh-my-zsh

```bash
sudo pacman -S zsh
# oh-my-zsh（GitHub 直连慢的话走加速镜像拉 install.sh）
bash <(curl -fsSL .../install.sh) --unattended

# 系统级语法高亮/自动建议（比 oh-my-zsh 自带的更好维护）
sudo pacman -S zsh-syntax-highlighting zsh-autosuggestions
```

## 3. 第一个大坑：默认 shell 切了却还是 bash

`chsh -s /usr/bin/zsh` 后，passwd 里确实是 zsh，但**新开的终端还是 bash**。而且报了 `Oh My Zsh can't be loaded from: bash`。

原因：**当前图形会话的 `SHELL` 环境变量在登录时就固化了**。alacritty 开新终端读的是 `$SHELL` 环境变量，不是 passwd——会话是切 shell 之前启动的，`$SHELL` 仍是 `/usr/bin/bash`。

解决：**注销重新登录**，让会话以新默认 shell 启动。在那之前无论开多少新终端都是旧 shell。

## 4. 第二个坑：报「Oh My Zsh can't be loaded from bash」

新终端若仍是 bash 环境里误 source 了 `.zshrc`，会触发 oh-my-zsh 的自我保护报错。本质还是「bash 启动链里混入了 zsh 配置」。排查思路：看 `.zshrc` 有没有被 bash 的 `.bashrc`/`.profile` 引用（正常不该有），并确认没有 BASH_ENV 之类把 `.zshrc` 喂给 bash。最终靠重登解决。

## 5. 让 zsh 继承 Omarchy 的 aliases 与提示符

### 5.1 复用 Omarchy 的 aliases

Omarchy 在 bash 里把 `ls` 别名成 `eza`、`cd` 接 zoxide、还有一堆工具别名。zsh 不会读 `.bashrc`，但这些 aliases 文件本身是 bash 语法、zsh 也能 source（函数部分兼容）：

```zsh
# ~/.zshrc
source /usr/share/omarchy/default/bash/aliases
```

这样 `ls`→eza、`cd`→zd 等就和 bash 里一致了。

### 5.2 Starship 提示符跨 shell 生效

Starship 是跨 shell 的提示符引擎，配置文件 `~/.config/starship.toml` **bash/zsh 共用**，不用迁移。zsh 里启用：

```zsh
eval "$(starship init zsh)"
```

记得先把 oh-my-zsh 的主题设空（`ZSH_THEME=""`），避免两个提示符打架。Starship 要放在 oh-my-zsh 加载之后、语法高亮之后。

### 5.3 启动顺序

.zshrc 里的顺序很重要：

```zsh
# 1. Omarchy 环境（PATH 等）
# 2. oh-my-zsh
source $ZSH/oh-my-zsh.sh
# 3. Omarchy aliases
source /usr/share/omarchy/default/bash/aliases
# 4. 语法高亮 + 自动建议（系统包）
source /usr/share/zsh/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
source /usr/share/zsh/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh
# 5. Starship 提示符（最后）
eval "$(starship init zsh)"
```

顺序错了（比如高亮在 oh-my-zsh 前）会导致提示符/高亮互相覆盖。

## 6. 顺带：终端复制粘贴改成 Windows 习惯

用 alacritty，想把复制粘贴改成 Ctrl+C/Ctrl+V。技巧是：**`Ctrl+C` 绑到 `Copy`，它只在有选区时生效，无选区时按键会继续传给 shell 作中断信号**——正好实现 Windows 那种行为：

```toml
# ~/.config/alacritty/alacritty.toml
[keyboard]
bindings = [
  { key = "C", mods = "Control", action = "Copy" },   # 有选区=复制，无选区=中断
  { key = "V", mods = "Control", action = "Paste" },
]
```

## 7. 终端中文字体发虚的另一层原因

zsh/Starship 显示中文时如果发虚像宋体，除了 fontconfig 回退问题（见中文化一文），也可能是等宽字体（JetBrainsMono）不含中文字形、回退选错。用 fc-match 验证并让 zh-cn 落到思源黑体即可。

## 8. 小结

| 问题 | 根因 | 解法 |
|---|---|---|
| chsh 后还是 bash | 会话 `$SHELL` 登录时固化 | 注销重登 |
| Oh My Zsh can't load from bash | bash 链误 source .zshrc | 排查启动链 + 重登 |
| 提示符/别名消失 | zsh 不读 .bashrc | source Omarchy aliases |
| Starship 不生效 | 顺序/主题未置空 | ZSH_THEME="" + eval starship |
| Ctrl+C 想复制 | 终端语义冲突 | Copy action 只在有选区时触发 |

最大的教训：**图形会话里「默认 shell」是登录时定死的**，改 passwd 不重登就是白改；以及 Omarchy 这类预置大量 bash 配置的系统，迁 zsh 要主动「继承」而不是指望自动。

