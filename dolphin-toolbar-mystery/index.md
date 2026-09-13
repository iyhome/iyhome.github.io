# Dolphin 工具栏凭空消失之谜：Qt saveState 的坑


「我不小心把 Dolphin 的工具栏关了，重启怎么还在？」——一个看似很小的问题，最后挖到 Qt 的 `QMainWindow::saveState`。本文记录这次排查，顺便说清 KDE 应用「界面状态到底存在哪」。

## 1. 现象

Dolphin 的工具栏（或菜单栏）被隐藏后，**关掉重开还是隐藏的**。直觉上「重启就恢复默认」，但并没有。

## 2. 第一层误判：以为存在 `dolphinui.rc`

KDE 的 XMLGUI 状态文件是 `~/.local/share/kxmlgui5/dolphin/dolphinui.rc`。但翻遍它，`<ToolBar name="mainToolBar">` 上**没有 `hidden` 属性**。而 `KToolBar::saveSettings` 只写 `IconSize` / `ToolButtonStyle`，**根本不写可见性**。所以状态不在这。

## 3. 真正的藏身处：`dolphinstaterc` 的 QMainWindow state

工具栏/停靠面板的布局状态，其实由 Qt 的 `QMainWindow::saveState()` 序列化后，存在 `dolphinstaterc` 的 `[State]` 里，是一段 base64：

```
[State]
State=AAAA/wAAAAD9AAAAAwAAAAAAAACyAAACQPwCAAAAAvsAAAAW...（内含 mainToolBar）
```

解码后能看到 `mainToolBar` 及其状态位。这就是「重启也恢复不了」的原因——**隐藏状态被持久化了**。

## 4. 解法

### 4.1 重置状态（推荐）

关掉 Dolphin，删掉它的 state 文件，再打开：

```bash
# 关 Dolphin
kquitapp6 dolphin
# 删除窗口/工具栏布局状态
rm -f ~/.local/state/dolphinstaterc
# 重新打开
dolphin &
```

### 4.2 用菜单找回

其实不用删文件：Dolphin 的菜单栏 → **设置 → 工具栏 → 勾选「主工具栏」**。（所以平时留个菜单栏当「安全出口」更稳。）

## 5. 一个 agent 环境的坑

排查时还遇到：状态文件竟然出现在 `~/.config/ai.opencode.desktop/dolphinstaterc`。原因是该 shell 里有：

```
XDG_STATE_HOME=/home/<user>/.config/ai.opencode.desktop
```

KDE 应用的状态文件位置 = `$XDG_STATE_HOME/dolphinstaterc`（默认 `~/.local/state`）。**环境变量不同，就会写到不同文件**——改错文件当然修不好。所以排查这类问题，先确认进程的 `XDG_STATE_HOME`。

## 6. 小结

| 线索 | 结论 |
|---|---|
| `dolphinui.rc` 无 `hidden` | 可见性不存 XMLGUI |
| `KToolBar::saveSettings` | 只存图标/按钮样式 |
| `dolphinstaterc` 的 `State=` | 真正的布局/可见性状态（Qt saveState） |
| `XDG_STATE_HOME` | 决定状态文件落点 |

**一句话**：KDE 应用的「窗口布局/工具栏可见性」藏在 `$XDG_STATE_HOME/<app>staterc` 的 Qt `saveState` 里，而不是 XMLGUI 的 `.rc`。下次界面「莫名恢复不了」，先去找这个 staterc。

