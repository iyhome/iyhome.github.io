# Omarchy 4 桌面全流程中文化实录


Omarchy 是一套基于 Arch Linux + Hyprland 的现代桌面发行版，默认界面为英文。本文记录把它「整机中文化」的完整流程：从系统 locale、Chrome 界面，到启动器菜单、状态栏插件的逐层汉化，最后落到一个社区本地化项目上。

## 0. 先说结论：中文要分四层解决

很多人的误区是「改个语言设置就全中文了」。实际上一台英文 Omarchy 要变中文，涉及互不相关的四层：

| 层 | 管什么 | 怎么改 |
|---|---|---|
| locale | 系统/程序语言 | `/etc/locale.conf` + `locale-gen` |
| GTK | 图形应用控件 | gsettings + `settings.ini` |
| fontconfig | 字体回退与渲染 | `~/.config/fontconfig/` |
| 应用自身 | 菜单/面板文案 | 各应用单独汉化 |

每一层都不理会其它层，缺一层就会有一块还是英文。

## 1. 系统 locale：让整机以中文启动

Omarchy 默认只生成 `en_US`。首先生成中文：

```bash
# 1. 启用 zh_CN.UTF-8
sudo sed -i 's/^#zh_CN.UTF-8 UTF-8/zh_CN.UTF-8 UTF-8/' /etc/locale.gen

# 2. 生成
sudo locale-gen

# 3. 设为系统默认
sudo sed -i 's/^LANG=.*/LANG=zh_CN.UTF-8/' /etc/locale.conf
```

**关键**：改完必须**注销重新登录**，否则当前会话里所有已启动进程仍是旧 `en_US`。这是整机中文化的第一步，也是很多人改了没反应的原因——没重登。

## 2. GTK 界面字体：为何 Chrome 中文了、系统设置还是英文

重登后会发现 Chrome（如果单独给它加过 `--lang=zh-CN`）中文了，但有些系统界面仍是英文。

Omarchy 的启动器菜单、状态栏是 Quickshell/QML 写的，**界面文案是硬编码英文**，不读 locale。要汉化它只能走配置覆盖或社区汉化项目（见第 5 节），靠改 locale 永远无效。

而 GTK 应用的默认字体、界面语言走的是 gsettings：

```bash
gsettings set org.gnome.desktop.interface font-name 'Microsoft YaHei UI 13'
```

同时建议写一份 `~/.config/gtk-3.0/settings.ini` 与 `gtk-4.0/settings.ini`，让不走 gsettings 的 GTK 应用也能读到：

```ini
[Settings]
gtk-font-name=Microsoft YaHei UI 13
```

## 3. fontconfig：终端中文为什么会发虚像宋体

给系统装了 Windows 字体（微软雅黑、宋体等）后，很可能出现一种怪现象：**终端里中文发虚、像宋体**。原因是 fontconfig 的字符回退扫描里，刚装的 `SimSun`（衬线宋体）在等宽字体缺中文字形时抢在了思源黑体前面。

定位方法：用 fc-match 看 JetBrainsMono 缺字时的回退顺序

```bash
fc-match -s "JetBrainsMono Nerd Font" | head -5
```

若 `simsun.ttc` 排得很靠前，就是它在作怪。解法：用 fontconfig 规则强制 zh-cn 文本落到思源黑体，避免终端中文用宋体渲染。

```xml
<!-- ~/.config/fontconfig/conf.d/90-zh-sans-cjk.conf -->
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>
  <match target="pattern">
    <test name="lang" compare="contains">
      <string>zh-cn</string>
    </test>
    <edit name="family" mode="prepend_first" binding="strong">
      <string>Noto Sans CJK SC</string>
    </edit>
  </match>
</fontconfig>
```

> 经验：用户级 `fonts.conf` 未必被加载，`~/.config/fontconfig/conf.d/` 下的高编号文件才稳定生效；系统级 `50-omarchy.conf` 用 `assign` 抢占了 `sans-serif`，要在 `conf.d/` 用更高优先级文件覆盖。

## 4. 把系统默认字体换成微软雅黑（又换回去的经历）

从 Windows 分区复制字体到 Linux 供个人使用，是可行的（注意版权，勿再分发）：

```bash
# 挂载 Windows 盘后，把 ttf/ttc 复制到用户字体目录
cp /path/to/msyh.ttc /path/to/simsun.ttc ~/.local/share/fonts/windows/zh/
fc-cache -f
```

但随后你会发现：微软雅黑装上后，系统默认中文可能被它或宋体接管，观感未必更好。经过反复对比，**默认中文还是思源黑体最耐看**——于是用 fontconfig 把默认文本钉回 `Noto Sans CJK SC`，Windows 字体仅作「需要时才指名使用」的备选。

这节的教训：不是字体越多越好，而是要让 fontconfig 的默认回退符合预期。

## 5. 启动器菜单与状态栏：交给社区本地化项目

既然 Quickshell 界面文案硬编码，手写覆盖几百条菜单项不现实。Omarchy 社区已有现成的简体中文本地化项目（思路：把系统插件 clone 成用户级副本再逐文件替换字符串）：

```bash
git clone <本地化项目仓库>
cd <项目目录>
./install.sh --dry-run   # 先看会改什么
./install.sh             # 正式安装
```

这类项目通常做这些事：

- 把官方插件 clone 为 `<用户名>.<插件名>` 的用户级副本
- 逐文件做中文字符串替换（菜单、快捷键面板、天气单位、日期星期）
- 装 `post-update` 钩子，系统升级后自动重新同步汉化
- 全程只写用户目录、不动 `/usr/share/omarchy`，自带备份与一键卸载

**收益**：菜单、Super+K 快捷键面板、音频/蓝牙/网络/电源面板、锁屏认证等全部中文，覆盖度远非手改可比。

## 6. 避坑清单

- 改 locale 必须重登，进程不会自己刷新。
- fcitx5 若被多处启动（autostart + systemd 服务并存）会互相覆盖配置、甚至丢输入法——确保只有一个实例。
- fontconfig 用户配置放 `conf.d/` 且编号高于系统文件才稳。
- 第三方汉化/美化项目安装前先 `--dry-run`，确认只写用户目录。

