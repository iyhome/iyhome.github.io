# 从 Omarchy 到 Fedora：把一整套桌面手感「整机复刻」的实录


换发行版，最舍不得的从来不是系统本身，而是那一套已经调顺手的环境：输入法、字体、shell、终端、工具链。这次把机器从 Arch 系的 Omarchy 换到 Fedora + KDE，本文记录如何把「手感」整套复刻过去，以及 KDE/Wayland 下和 Hyprland 不一样的那些坑。

## 0. 复刻，不是重装

配置文件和词库是可以带走的，真正不可移植的是「桌面层」：Hyprland 的窗口规则、Quickshell 的状态栏、Omarchy 的 `omarchy` CLI，在 Fedora/KDE 上统统不存在。所以思路是——**只复刻体验层**（输入法、字体、shell、终端、CLI、Wayland 行为），桌面组件交给 KDE 原生。

| 要复刻的 | Omarchy 的做法 | Fedora/KDE 的对应 |
|---|---|---|
| 输入法 | fcitx5 + Rime 小鹤双拼 | 一样，装包即可 |
| 字体 | 从 Windows 分区拷字体 + fontconfig | 一样 |
| Shell | zsh + oh-my-zsh + Starship | 一样 |
| 终端 | alacritty | 直接用 KDE 自带 Konsole |
| 状态栏小部件 | Quickshell/QML | KDE 面板小部件 |
| 窗口管理 | Hyprland | KWin |

## 1. 输入法：fcitx5-rime 在 Fedora 上的正确姿势

Fedora 打包齐全，一条命令：

```bash
sudo dnf install fcitx5 fcitx5-rime fcitx5-configtool fcitx5-qt fcitx5-gtk kcm-fcitx5 fcitx5-autostart
```

然后把 rime-ice（雾凇拼音）全量放到 `~/.local/share/fcitx5/rime/`，用 `default.custom.yaml` 把小鹤双拼收窄为唯一方案、左右 Shift 切中英。

### 1.1 KDE Wayland 下只保留 XMODIFIERS

这是和 Omarchy 最大的不同。KDE Plasma 走 Wayland 原生输入法前端（`text-input` 协议），**不该再设 `GTK_IM_MODULE` / `QT_IM_MODULE`**，否则会强制走 im 模块，导致候选项闪烁甚至重复输入。官方推荐只留：

```ini
# ~/.config/environment.d/10-fcitx.conf
XMODIFIERS=@im=fcitx
```

并在 KDE 的 `kwinrc` 里把虚拟键盘设为 Fcitx 5（`[Wayland] InputMethod=org.fcitx.Fcitx5.desktop`），这样 KWin 会负责拉起并转发输入法。

### 1.2 默认英文 + 终端自动英文

rime-ice 默认中文。想默认英文，用方案级补丁把 `ascii_mode` 的 reset 置 1：

```yaml
# double_pinyin_flypy.custom.yaml
patch:
  switches/@0/reset: 1
```

再进一步：**焦点进终端自动切英文、离开自动还原**。fcitx5-rime 会用 `InputContext::program()`（程序名）匹配 `fcitx5.yaml` 的 `app_options`：

```yaml
# fcitx5.custom.yaml
patch:
  app_options:
    konsole: { ascii_mode: true }
    org.kde.konsole: { ascii_mode: true }
```

改完不必重启 fcitx5，用 DBus 触发重部署即可：

```bash
busctl --user call org.fcitx.Fcitx5 /controller org.fcitx.Fcitx.Controller1 ReloadAddonConfig s rime
```

### 1.3 候选框圆角外的「矩形层」

换到 KDE 后，候选框沿用了 macOS-light 皮肤：四角是圆角，但圆角外还能看到一层模糊的矩形，整体看起来像矩形而不是真正的圆角。根因是皮肤 `theme.conf` 里的 `EnableBlur=True`——KDE 下 KWin 的模糊区域是**整个矩形 surface**（`BlurMask` 为空时），圆角外就露出模糊层。而这个皮肤背景本就不透明，模糊完全用不上，关掉即可、视觉无损：

```ini
# ~/.local/share/fcitx5/themes/macOS-light/theme.conf
[InputPanel]
EnableBlur=False
```

改完 `fcitx5-remote -r` 重载即可。

## 2. 字体：搬过来，再把回退钉死

把 Windows 分区里的 ttf/ttc 复制到用户字体目录（仅个人自用，注意版权）：

```bash
mkdir -p ~/.local/share/fonts/windows
cp -r /path/to/fonts/{zh,latin} ~/.local/share/fonts/windows/
fc-cache -f
```

装了微软雅黑后，很可能出现「终端中文发虚像宋体」——根因是 fontconfig 回退里 `SimSun` 抢在了思源黑体前。用高编号的 `conf.d` 规则把 zh-cn 钉回思源黑体：

```xml
<!-- ~/.config/fontconfig/conf.d/90-zh-sans-cjk.conf -->
<match target="pattern">
  <test name="lang" compare="contains"><string>zh-cn</string></test>
  <edit name="family" mode="prepend_first" binding="strong">
    <string>Noto Sans CJK SC</string>
  </edit>
</match>
```

验证：

```bash
fc-match -s "JetBrainsMono Nerd Font" | head -3
```

## 3. Shell：zsh + Starship 的「PATH 陷阱」

```bash
sudo dnf install zsh zsh-syntax-highlighting zsh-autosuggestions
git clone --depth=1 https://github.com/ohmyzsh/ohmyzsh.git ~/.oh-my-zsh
sudo chsh -s /usr/bin/zsh serein
```

**大坑**：`chsh` 后必须**注销重登**（图形会话的 `$SHELL` 登录时固化）。重登后新问题来了——Starship 装了却报「未找到命令」，提示符退回默认。根因是重登后 `~/.local/bin` 掉出了 `PATH`（Starship/lazygit 这类用户级二进制装在这里）。在 `.zshrc` 顶部补一行即可：

```zsh
export PATH="$HOME/.local/bin:$HOME/bin:$PATH"
```

Fedora 的语法高亮/自动建议包路径和 Arch 不同，source 时要用 Fedora 路径：

```zsh
source /usr/share/zsh-autosuggestions/zsh-autosuggestions.zsh
source /usr/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
```

## 4. CLI 工具链与 Starship 主题

Fedora 源里没有 `starship` 和 `lazygit`，直接下官方二进制到 `~/.local/bin` 最省事。其余一条 dnf 搞定：

```bash
sudo dnf install ripgrep fd-find bat eza fzf zoxide tmux btop jq tealdeer yt-dlp mpv neovim fastfetch
```

Starship 想要 Catppuccin 的 powerline 风格，一行预设：

```bash
starship preset catppuccin-powerline -o ~/.config/starship.toml --force
```

再按喜好删掉 `format` 里的 `$username` / `$time` / `$os` 等段即可（改 `format` 字符串，不是删模块配置）。

## 5. Wayland 环境变量：别让 Qt5/Electron 赖在 XWayland

Omarchy 在 `envs.lua` 里有一组强制 Wayland 的变量。搬到 KDE 要**去掉 Hyprland 专属项、且绝不覆盖 `XDG_CURRENT_DESKTOP`**：

```ini
# ~/.config/environment.d/20-wayland-apps.conf
GDK_BACKEND=wayland,x11
QT_QPA_PLATFORM=wayland;xcb
MOZ_ENABLE_WAYLAND=1
ELECTRON_OZONE_PLATFORM_HINT=wayland
```

其中 `QT_QPA_PLATFORM` 对 Qt5 应用（比如微信）尤其关键——不设它，Qt5 会默认走 XWayland，高分屏下发虚、右键菜单闪烁。

## 6. 小结

| 事项 | 关键点 |
|---|---|
| 输入法 | KDE Wayland 只留 `XMODIFIERS`；`app_options` 做终端自动英文 |
| 字体 | 高编号 `conf.d` 规则把 zh-cn 钉到思源黑体 |
| Shell | 重登后补 `~/.local/bin` 到 PATH |
| 工具链 | starship/lazygit 下官方二进制 |
| Wayland | 不覆盖 `XDG_CURRENT_DESKTOP`，Qt5 要显式设 `QT_QPA_PLATFORM` |

最大的体会：**发行版迁移真正花时间的不是「装什么」，而是那些「默认值不一样」的地方**。桌面层的组件交给目标发行版的原生方案，比硬搬更省心。

