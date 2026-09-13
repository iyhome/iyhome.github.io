# 把 DeepSeek 余额搬上 KDE 面板：Command Output 小部件实践


之前给 Omarchy 的状态栏写过 DeepSeek 余额小部件（Quickshell/QML）。换了 Fedora + KDE 之后，Quickshell 没了，余额也就看不见了。本文记录怎么用 KDE 的「Command Output」小部件 + 一个采集脚本，把余额重新钉到面板上。

## 1. 背景：Omarchy 那套搬不过来

Omarchy 的余额小部件是 Quickshell 插件：`manifest.json` + `collector.sh` + `BalanceWidget.qml`。KDE Plasma 用的是完全不同的插件体系，QML 直接搬不过来。而 KDE 自带面板里并没有「显示一条命令输出」的现成组件——需要从 KDE Store 装一个。

## 2. 采集脚本：读 opencode 的 key，输出一行文本

核心就一个脚本，从 opencode 的 `auth.json` 取 DeepSeek key，查余额接口，输出形如 `¥78.9`：

```bash
#!/usr/bin/env bash
# ~/.local/bin/deepseek-balance.sh
set -uo pipefail
AUTH_JSON="${OPENCODE_AUTH_JSON:-$HOME/.local/share/opencode/auth.json}"
URL="https://api.deepseek.com/user/balance"

key=$(jq -r '.deepseek.key // empty' "$AUTH_JSON" 2>/dev/null || true)
[ -z "$key" ] && { echo "DS?"; exit 0; }

out=$(curl -sS -m 10 -w $'\n%{http_code}' \
  -H "Authorization: Bearer $key" -H "Accept: application/json" "$URL" 2>/dev/null)
code=${out##*$'\n'}; body=${out%$'\n'*}
[ "$code" != "200" ] && { echo "DS·err"; exit 0; }

total=$(jq -r '.balance_infos[0].total_balance // empty' <<<"$body")
currency=$(jq -r '.balance_infos[0].currency // "CNY"' <<<"$body")
[ -z "$total" ] && { echo "DS·?"; exit 0; }

sym="¥"; [ "$currency" = "USD" ] && sym="$"
printf '%s%.1f\n' "$sym" "$total"
```

失败时返回一个短占位符（`DS?` / `DS·err`）而不是报错崩掉，这样面板不会突然变空。

## 3. 装「Command Output」小部件

Fedora 默认没打包它。它来自 KDE Store 的 `com.github.zren.commandoutput`，源码在 GitHub，可直接装：

```bash
git clone --depth 1 https://github.com/Zren/plasma-applet-commandoutput.git
kpackagetool6 --type Plasma/Applet --install plasma-applet-commandoutput/package
```

它的配置项里，关键是 `command`（要跑的命令）和 `interval`（毫秒）。

## 4. 脚本化地把它加进面板

不想手点，可以用 Plasma 的脚本接口直接加。先找面板 id：

```bash
busctl --user call org.kde.plasmashell /PlasmaShell org.kde.PlasmaShell evaluateScript s \
  'print(panelIds.join(","))'
```

再加小部件并写配置（注意 `evaluateScript` 只有用 `print()` 才有回显）：

```bash
busctl --user call org.kde.plasmashell /PlasmaShell org.kde.PlasmaShell evaluateScript s '
var p = panelById(panelIds[0]);
var w = p.addWidget("com.github.zren.commandoutput");
w.currentConfigGroup = ["General"];
w.writeConfig("command", "/home/me/.local/bin/deepseek-balance.sh");
w.writeConfig("interval", 120000);
print("added id=" + w.id);
'
```

再配一个点击动作，弹通知显示详情（`clickCommand` 指向一个包了 `notify-send` 的脚本）。

## 5. 效果与小结

面板右侧就会出现 `¥78.9`，每 2 分钟自动刷新，点击弹出余额详情。截图里它挨着时钟，存在感和原来的 Omarchy 状态栏差不多。

| 组件 | 作用 |
|---|---|
| `deepseek-balance.sh` | 查 API，输出一行余额 |
| Command Output 小部件 | 定时跑脚本、把输出渲染到面板 |
| `clickCommand` + notify-send | 点击看详情 |

**要点**：KDE 缺什么组件，就去 KDE Store 找对应的 plasmoid，再用 `evaluateScript` 脚本化装配——比手动拖拽可复现得多。而「脚本采集 + 组件展示」这个模式，从 Quickshell 换到 Plasma 依然通用。

