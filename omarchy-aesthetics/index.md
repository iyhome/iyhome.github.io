# 从开机到锁屏：复刻一套「终端风」桌面美学


一套桌面「好看」的关键，往往不是某个组件，而是**一致性**。Omarchy 的美学是一套贯穿「开机 → 引导 → 登录 → 锁屏」的终端风极简深色。本文拆解它的构成，以及如何在别的发行版上复刻。

## 0. 核心：统一色 + 等宽字体

- 统一色：`#1a1b26`（Tokyo Night 背景），贯穿 Plymouth、SDDM、锁屏、屏保
- 统一字体：等宽（JetBrainsMono / Cantarell）
- 统一语言：留白、克制、无多余装饰

四个阶段视觉连贯、不割裂，才显得「整」。下面逐个说。

## 1. 开机动画：Plymouth

它做了两件有质感的事：

- **假进度条**：系统加载时平滑前进到 70%，带 ease-out 缓动（`1-(1-x)²`），视觉上不卡顿
- **LUKS 解锁界面**：加密盘时切到同风格的密码输入

移植（三平台通用）：

```bash
sudo cp -r aesthetics/plymouth /usr/share/plymouth/themes/omarchy
sudo plymouth-set-default-theme -R omarchy   # -R 重建 initramfs
```

> 关键：装完必须**重建 initramfs** 才生效；内核参数要加 `quiet splash`。

## 2. 登录：SDDM

终端风极简登录：深色背景、居中 logo、密码输入，无花哨壁纸。装主题并启用：

```bash
sudo cp -r aesthetics/sddm /usr/share/sddm/themes/omarchy
echo -e "[Theme]\nCurrent=omarchy" | sudo tee /etc/sddm.conf.d/10-theme.conf
sudo systemctl enable sddm
```

（Fedora 默认 GDM，换 SDDM 需先 disable GDM。）

## 3. 锁屏与屏保

- 锁屏：Quickshell 的 LockView（密码圆点动态缩放）；其它系统可用 `hyprlock` / `swaylock-effects`
- 屏保：在终端里跑 `ttfx`（终端文字特效）；替代品 `cmatrix`、`pipes.sh`、`cava`

## 4. 主题色系统：一处改，全体变

Omarchy 用一份 `colors.toml` 同时驱动终端、编辑器、浏览器、锁屏、Plymouth、SDDM。非 Omarchy 系统不必重造，直接用官方多应用主题包：

- **Tokyo Night**：https://github.com/folke/tokyonight.nvim
- **Catppuccin**：https://github.com/catppuccin/catppuccin（覆盖 300+ 应用）

## 5. 复刻优先级

| 优先级 | 组件 | 理由 | 难度 |
|---|---|---|---|
| ★★★ | 主题色统一（Catppuccin/Tokyo Night） | 一处改全体，观感提升最大 | 易 |
| ★★★ | SDDM 登录主题 | 每天见 | 中 |
| ★★☆ | Plymouth 开机动画 | 开机第一印象 | 中 |
| ★★☆ | 锁屏 | 高频交互 | 易-中 |
| ★☆☆ | 屏保 | 锦上添花 | 易 |

## 6. 小结

一句话：**Omarchy 美学的精髓是「统一」**——同一套深色 + 等宽字体贯穿始终。复刻时不必逐组件照搬，用「跨应用主题包 + 一个终端风 SDDM 主题」，就能拿到大部分观感。

