# Omarchy 的「体验层」模块：哪些值得移植，哪些交给系统


Omarchy 不只是一个 Hyprland 配置，它在**应用体验层**做了大量细节优化。但这些优化并非都要照搬——本文按模块梳理它「好在哪、怎么移植」，并给出结论：抓三样就够。

## 1. Wayland 环境变量（★★★ 核心）

**做了什么**：一组环境变量强制应用走 Wayland 原生渲染，避免 XWayland 的闪烁/延迟/模糊。

**效果**：微信/QQ 等 Qt 应用的右键菜单、弹窗顺滑不闪；高分屏下 XWayland 应用不模糊。

```ini
# 任意 + GNOME/KDE（systemd 会话）：~/.config/environment.d/10-wayland-apps.conf
GDK_BACKEND=wayland,x11
QT_QPA_PLATFORM=wayland;xcb
MOZ_ENABLE_WAYLAND=1
ELECTRON_OZONE_PLATFORM_HINT=wayland
```

> 重要：**不要覆盖 `XDG_CURRENT_DESKTOP`**（要保持 KDE/GNOME）；`QT_QPA_PLATFORM` 对 Qt5 应用尤其关键。

## 2. 截图 / 录屏（★★☆）

`grim`（截）+ `slurp`（选区）+ `wl-clipboard`；录屏用 `gpu-screen-recorder`（GPU 加速，几乎不掉帧）。

```bash
sudo dnf install grim slurp wl-clipboard
```

> GNOME/KDE 自带截图录屏，可不装（KDE 用 Spectacle）。

## 3. 通知 / OSD（★★☆）

Omarchy 有统一的通知 + 屏幕中央 OSD（调音量/亮度时显示进度条）。通用替代：`mako`（轻量通知守护）。

```bash
sudo pacman -S mako    # Arch；Ubuntu/Fedora 也有
```

> GNOME/KDE 有自带 OSD，够用。

## 4. 显示器管理（★★☆）

多屏/缩放/合盖这些高频痛点，Omarchy 用 `omarchy hyprland monitor *` 统一命令。通用替代：`kanshi`（Wayland 通用，按配置自动设屏）或 `nwg-displays`。

## 5. Web App（★★☆）

`omarchy webapp install <名字> <url>` 把网站变成独立应用。Chrome 系通用做法：

```bash
google-chrome --app="https://chat.openai.com" --class=webapp-chatgpt
```

## 6. 各类 Toggle（★☆☆）

夜间护眼、免打扰、保持唤醒……通用替代：`gammastep`（护眼）、`makoctl mode -t do-not-disturb`（免打扰）、`systemd-inhibit`（保持唤醒）。

## 7. 结论：抓三样就够

| 优先级 | 模块 | 理由 |
|---|---|---|
| ★★★ | Wayland 环境变量 | 一处设置，全部应用受益 |
| ★★★ | 跨应用主题（Catppuccin/Tokyo Night） | 整机观感统一 |
| ★★☆ | 截图 + 通知工具链 | 高频使用 |

**核心结论**：Omarchy 的价值 = **统一的体验层**。非 Omarchy 系统不必逐个复刻，抓住「Wayland 环境变量 + 跨应用主题 + 截图/通知工具链」这三样，就能获得 80% 的体验提升；其余交给目标桌面（GNOME/KDE）的原生方案。

