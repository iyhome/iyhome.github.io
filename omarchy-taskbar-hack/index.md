# 把 Linux 平铺桌面的任务栏改成 Windows 手感：一次开源贡献


Hyprland 默认没有「任务栏」概念。Omarchy 提供了 Dock（悬浮应用坞）但它是独立悬浮层，嵌不进状态栏。本文记录：从「想要状态栏中间有个 Windows 式任务栏」出发，先试了两个现成插件，最后改造第三方 `omarchy-taskbar` 并向上游发 PR 的完整过程——顺带覆盖了 Hyprland 窗口贴边、圆角、layer surface 弹窗等一堆桌面开发知识点。

## 1. 想要什么

状态栏（bar）在屏幕底部，希望中间有一块「任务栏」：

- 固定常用应用，点击启动/聚焦
- **正在运行但没固定的应用也自动出现**，每应用一个图标
- 同一应用开多个窗口，**只显示一个图标**；鼠标悬停弹出窗口列表，点哪个聚焦哪个（像 Windows 任务栏）

## 2. 试错：两个现成插件都不完美

### 2.1 joeyvigil/omarchy-taskbar —— 只显示固定的

它是个「Pinned app launcher」：固定应用常驻，图标下有运行指示点。但它**不显示未固定的运行应用**，多窗口只计数，没有悬停选窗。装了之后并不像 Windows。

### 2.2 IntelligentWolf-Ltd/omarchy-taskbar —— 每个窗口一个图标

这个反而「太忠实」：**每个窗口一个图标**，开两个终端就出两个图标。我想要的是按应用聚合，不是按窗口铺开。不符。

试错结论：现成插件都不满足「固定 + 运行 + 按应用聚合 + 悬停选窗」的组合，只能自己改。

## 3. 改造 joeyvigil/omarchy-taskbar

它的代码质量不错：图标、运行指示、聚焦高亮、点击循环切窗都是现成的，且 delegate 完全按「一条 record」泛化——只要给 record 填对 `desktopId/match/icon/label`，图标和交互全自动。所以要加两个能力：

### 3.1 显示「运行中但未固定」的应用

思路：对每个运行窗口，用它的 appId/class 反查系统里的 desktop entry；凡是没被任何固定项「认领」的窗口，按 desktop entry 聚合成一个临时 slot。

- 反查靠 `appId`（Wayland）与 `class`（XWayland），结合 desktop entry 的 `id`、`StartupWMClass`、`exec` 二进制、webapp 域名等多层评分
- 临时项记 `temp: true`，**绝不写回 shell.json 的 apps 持久化**（只在内存 model 里）
- 固定项运行时通过「认领（claimed）地址集合」避免重复：先收集固定项已匹配的窗口，剩下的才归临时项

核心伪逻辑（AppModel.js 里加纯函数）：

```js
function runningGroups(records, windows, entries) {
  // 1. 收集固定项已认领的窗口地址
  // 2. 遍历剩余窗口，对每个窗口的 appId/class 反查 entry（打分取最高）
  // 3. 按 entry 聚合成组，组内窗口的 cls/appId token 求并集
  // 4. 输出 record：{ desktopId, icon, label, match:"^(token|…)$", temp:true }
}
```

### 3.2 悬停弹出窗口列表

这是 QML 层最大的活。参考 Omarchy 自带的 `PopupCard`（Tray/clock 都在用，封装了 Wayland `PopupWindow` + 四向锚定 + 越界 clamp + 主题外观），在 BarWidget 里加一个共享 PopupCard + hover 状态机：

- 进入图标 → 320ms 延迟后打开（防横扫时闪烁）
- 离开图标 → 240ms 缓冲，期间指针移进弹窗就续命
- 弹窗内每行一个窗口标题，点击「先关弹窗再 focus」（focus 会 warp 指针/重建，别让弹窗活过 anchor）
- 只对多窗口（≥2，可配）弹列表；单窗口仍用 tooltip

关键的坑：**Repeater 的 model 不能绑到一个会写缓存的函数**，否则 QML 报 binding loop。解法是把 model 绑到一个普通数组属性，由窗口/固定变化事件**命令式**地 `rebuildSlots()` 更新它。

## 4. 让改动被上游接受：拆分与规范化

改造完、自测没问题后，想贡献回原作者。做对了几件事：

1. **清掉本地备份/调试残留**，diff 只剩有意义的改动
2. **新功能都做成可配置开关**：`showRunningApps`、`hoverWindowList`、`hoverListMinWindows`，默认开启但不强加
3. **同步更新 manifest**：新 setting 加进 `defaults` 和 `schema`，UI 能调
4. **写清晰的两段式 commit message**（做了什么 + 为什么），PR 描述带动机、改动文件、测试方法

提交链路：

```bash
# 目录本身是原仓库的 git clone，直接在本地 commit
git commit -m "feat: show running (unpinned) apps and hover window list ..."

# fork 到自己的账号再推送分支
gh repo fork joeyvigil/omarchy-taskbar
git remote add fork https://github.com/<you>/omarchy-taskbar.git
git push fork feat/running-apps-and-hover-window-list

# 发 PR
gh pr create --repo joeyvigil/omarchy-taskbar --base master \
  --head <you>:feat/running-apps-and-hover-window-list --title "..." --body "..."
```

## 5. 顺带收获：Hyprland 窗口管理的几个知识点

### 5.1 全浮动布局下的「最大化」会盖住状态栏

Omarchy 默认新窗口平铺。改成全局浮动（`o.window(".*", { float = true })`）后，写贴边脚本「最大化」若直接填满 monitor，会盖住顶部状态栏。要读 Hyprland 的 **reserved 区域**（状态栏占用的顶部像素）并避开：

```bash
# monitors 的 reserved 是物理像素，要除以 scale 换成逻辑像素
hyprctl -j monitors | jq '.[0].reserved'
```

脚本让窗口从 `reserved_top + 边距` 开始、高度减去 reserved，状态栏就露出来了。

### 5.2 圆角统一走 Hyprland 的 decoration:rounding

Omarchy 的 dock 圆角、窗口圆角都联动 `decoration:rounding`。默认是 0（方角），设个值 dock 才有 macOS 那种胶囊圆角：

```lua
-- ~/.config/hypr/looknfeel.lua
hl.config({ decoration = { rounding = 12 } })
```

### 5.3 鼠标边缘改窗口大小

浮动布局下要像 Windows 那样拖窗口边缘缩放，需开：

```lua
hl.config({ general = { resize_on_border = true } })
```

## 6. 小结

| 需求 | 方案 |
|---|---|
| 固定应用 | 沿用 pinned 槽位 |
| 运行中的未固定应用 | 反查 desktop entry + 认领去重，临时 slot |
| 每应用一个图标 | 按 entry 聚合，不按窗口 |
| 悬停选窗 | PopupCard + hover 状态机，列出窗口标题点选 |
| 回馈社区 | fork + 清晰 commit + PR |

核心经验：改第三方插件前先读透它的 delegate/模型设计，往往能发现「加一个 record 就能复用整套交互」的扩展点，比从头写省太多。而贡献上游时，把个人偏好硬编码改成「可配置 + 文档化 + 清晰 commit」，是 PR 能被接受的三个关键。

