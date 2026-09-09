# Linux 输入法迁移记：从 fcitx5 拼音到 Rime 小鹤双拼


在 Hyprland / Wayland 环境下配中文输入法，踩过的坑比想象中多。本文记录最终落在 Rime + 小鹤双拼（flypy）的完整过程，以及为什么放弃 fcitx5 自带拼音。

## 1. 起点：fcitx5 + 智能拼音

Omarchy 默认环境已装好 fcitx5 和输入法前端环境变量。装上拼音引擎就能打中文：

```bash
sudo pacman -S fcitx5-chinese-addons
```

然后在 `~/.config/fcitx5/profile` 里把 `pinyin` 加进输入法组。fcitx5 的默认切换键是 **Ctrl+Space**。

## 2. 踩坑：fcitx5 拼音配置为何反复丢失

配置过程中反复出现一个诡异现象：明明在 `~/.config/fcitx5/conf/pinyin.conf` 里改好了双拼方案、候选数，**重启 fcitx5 后全部被还原**。

排查后确认根因不是配置写错，而是**进程冲突**：

- 系统 autostart 里有一份 `org.fcitx.Fcitx5.desktop`
- 用户 autostart 里又有一份同样的
- Omarchy 的 systemd 服务 `omarchy-fcitx5.service`（带 `Restart=always`）还会再拉起一个

多个实例同时启动，彼此竞争 bus、互相用默认配置覆盖用户的 profile/conf。解决：

```bash
# 只保留 systemd 服务这一个实例
rm -f ~/.config/autostart/org.fcitx.Fcitx5.desktop
systemctl --user reset-failed omarchy-fcitx5.service
systemctl --user restart omarchy-fcitx5.service
```

确认只有单一 `fcitx5 --disable notificationitem` 进程后，配置就稳定了。

## 3. 放弃 fcitx5 拼音：双拼方案不生效

想要小鹤双拼，在 fcitx5 的 libpinyin 引擎里 `ShuangpinProfile=Xiaohe` 反复配置，实际输入仍是全拼。加上曾观察到拼音引擎加载异常（日志里始终没有主 pinyin addon），最终决定换更成熟的方案。

## 4. Rime：小鹤双拼的正确打开方式

Rime（中州韵）本身不含小鹤方案，需安装引擎并补充 flypy 方案文件。

### 4.1 安装引擎

```bash
sudo pacman -S fcitx5-rime
```

### 4.2 获取小鹤双拼方案

librime 自带方案里没有 flypy，需要从 Rime 的 double-pinyin 仓库拿：

```bash
git clone --depth 1 https://github.com/rime/rime-double-pinyin.git
cp rime-double-pinyin/double_pinyin_flypy.schema.yaml \
   ~/.local/share/fcitx5/rime/
```

### 4.3 设为默认方案、候选 5、默认简体

Rime 的用户配置在 `~/.local/share/fcitx5/rime/`，用 `*.custom.yaml` 做补丁：

```yaml
# ~/.local/share/fcitx5/rime/default.custom.yaml
patch:
  schema_list:
    - schema: double_pinyin_flypy
  menu:
    page_size: 5
  ascii_composer:
    switch_key:
      Shift_L: commit_code
      Shift_R: commit_code
```

```yaml
# ~/.local/share/fcitx5/rime/double_pinyin_flypy.custom.yaml
# 方案级开关：默认简体（switches 里 simplification 置 1）
patch:
  "switches/@2/reset": 1
```

改完重启 fcitx5（或重部署 Rime）即可。验证编译产物：

```bash
# 编译后的方案应含小鹤开关与简体默认
grep -A4 switches ~/.local/share/fcitx5/rime/build/double_pinyin_flypy.schema.yaml
```

### 4.4 一个坑：简体又变回繁体

flypy 方案默认输出繁体（它基于朙月拼音词库），需确保 `simplification` 开关默认置为「汉字」。若只在全局 `default.custom.yaml` 里乱改 `switches` 的索引，容易把结构改坏（编译产物里 switches 只剩 reset 行）——**方案级定制应写在方案同名 `.custom.yaml` 里**，而不是全局补丁。

## 5. 默认英文、Shift 切中英（未完全如愿的部分）

理想目标是「默认英文，Shift 点按切换中英」，但 Wayland 的输入法协议对**单独的 Shift 键事件**支持不佳——很多工具链不把裸修饰键发给输入法，导致 Rime 收不到「单独按 Shift」而无法可靠切换。可用的折中：

- Rime 内置组合键 `Ctrl+Shift+2` 切换 ascii_mode（可靠）
- 或接受「按住 Shift 临时英文、松开回中文」的 inline_ascii 行为

若坚持 Shift 点按切换，需要在合成器/输入法协议层面处理，成本高、收益低。

## 6. 最终形态

- 输入法：Rime（fcitx5-rime 承载）
- 方案：小鹤双拼（double_pinyin_flypy）
- 候选：每页 5 个
- 输出：默认简体
- 候选框字体：随 GTK/主题，思源黑体 18

Rime 的整句联想、词库同步、方案可编程性都远胜 fcitx5 拼音，迁移后输入体验明显更顺。唯一要适应的是双拼键位本身。

