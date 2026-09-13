# 在 Windows 里养一只 Arch：WSL 从裸装到顺手


想在 Windows 上用 Linux 工具链，最省事的是 WSL2。发行版选了 `archlinux`——裸装之后才发现，它真的是「从零开始」：默认 root、没有 sudo、没有 locale、源还慢。本文记录把它驯服成顺手开发环境的全过程。

## 1. 为什么是 WSL2 + Arch

要的是 Linux 工具链（rg/fd/bat/eza/tmux…）和原生包管理。WSL2 性能足够、与 Windows 文件互通；Arch 的 AUR 又能拿到最新工具。代价是：一切都要自己配。

## 2. 先建用户和 sudo

裸 archlinux 默认以 **root** 登录（免密）。先建普通用户并迁过去：

```bash
useradd -m -G wheel serein
passwd serein            # 密码设成单个空格（本项目一贯的极简风格）
pacman -S sudo
sed -i 's/^# %wheel ALL=(ALL:ALL) ALL/%wheel ALL=(ALL:ALL) ALL/' /etc/sudoers
```

再让 WSL 默认用这个用户（`/etc/wsl.conf`）：

```ini
[user]
default=serein
```

> 注意：发行版名是 `archlinux`，不是 `WSL-Arch`。

## 3. 换源 + AUR

官方镜像在国内常超时，换中科大：

```bash
sed -i 's|^#Server.*mirror|Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch|' /etc/pacman.d/mirrorlist
# archlinuxcn（yay 在这里）
```

装 `yay`（archlinuxcn 源里直接有），之后 AUR 也顺手了。

## 4. locale：agnoster 箭头报错的元凶

装了 zsh + oh-my-zsh 的 **agnoster** 主题后，提示符没生效、退回默认的 `主机名#`，还伴随 unicode 报错。根因是**裸 archlinux 没有生成 locale**，agnoster 用来画箭头的特殊字符解析失败。

```bash
sed -i 's/^#en_US.UTF-8/en_US.UTF-8/;s/^#zh_CN.UTF-8/zh_CN.UTF-8/' /etc/locale.gen
locale-gen
```

## 5. shell：zsh + oh-my-zsh

官方安装脚本依赖 `curl`，而 Windows 侧 `curl` 是 `Invoke-WebRequest` 的别名，脚本跑不起来。手动 clone 最稳：

```bash
git clone --depth=1 https://github.com/ohmyzsh/ohmyzsh.git ~/.oh-my-zsh
chsh -s /usr/bin/zsh
```

`~/.zshrc` 里设 `ZSH_THEME="agnoster"`，并加上常用别名（`ls`→eza、`cat`→bat、zoxide、fzf 等）。

## 6. 工具链

```bash
pacman -S ripgrep fd bat eza fzf tldr python nodejs npm tmux jq lazygit zoxide neovim stow htop tree which git
```

## 7. 两个必踩的坑

### 7.1 metadata：让 WSL 能正确改权限

默认挂载的 Windows 盘不保留 Linux 权限，`chmod` 无效。开 metadata：

```ini
# /etc/wsl.conf
[automount]
options="metadata,umask=22,fmask=11"
```

（改完 `wsl --shutdown` 重启生效。）

### 7.2 sudo 会清掉代理

本机用 Clash（TUN + Mirrored，代理 `127.0.0.1:7897`）。`sudo` 默认清空 `http_proxy` 等变量，导致 `sudo pacman` 直连、部分源超时。保留变量：

```bash
# /etc/sudoers.d/keep-proxy
Defaults env_keep += "http_proxy https_proxy HTTP_PROXY HTTPS_PROXY no_proxy NO_PROXY all_proxy ALL_PROXY"
```

## 8. 小结

| 步骤 | 关键 |
|---|---|
| 用户 | 建普通用户 + wheel sudo，`wsl.conf` 设默认用户 |
| 源 | 中科大 + archlinuxcn + yay |
| locale | 不生成的话 agnoster 直接失效 |
| shell | 手动 clone oh-my-zsh（Windows curl 别名坑） |
| 权限 | automount `metadata` |
| 代理 | sudoers `env_keep` |

一句话：**WSL 的 Arch 不是「开箱即用」，而是「给你一个干净内核，其余自己长」**。好处是长出来的东西完全可控，坏处是每一样都要自己种。

