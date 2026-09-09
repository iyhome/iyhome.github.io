# 给 Omarchy 状态栏写一个 DeepSeek 余额小部件


用 opencode 这类 AI 编码工具时，DeepSeek 按 token 计费，余额常被忽略。社区已有显示「OpenCode Go」订阅用量的状态栏插件，但它读的是订阅账户、**读不到 DeepSeek 这类自充值 API 的余额**。本文记录如何照 Omarchy 的插件规范，从零写一个显示 DeepSeek 余额的 bar 小部件。

## 1. 需求与选型

想要的：状态栏里直接看到 DeepSeek 余额，比如「¥10.3」，几秒自动刷新、点击手动刷新。

一个现成的思路是把某个读 opencode-go 的插件改造成读 DeepSeek 余额，但那个插件的 UI（进度条、限额、重置倒计时）全是按订阅语义设计的，改造风险高。**更干净的做法是新建一个极简插件**，只做一件事：查余额 → 显示文本。

## 2. 余额查询 API

DeepSeek 官方有余额接口，用 API key 作 Bearer 认证：

```bash
curl -s https://api.deepseek.com/user/balance \
  -H "Authorization: Bearer $DEEPSEEK_API_KEY"
```

返回形如：

```json
{
  "is_available": true,
  "balance_infos": [
    { "currency": "CNY", "total_balance": "10.32",
      "granted_balance": "0.00", "topped_up_balance": "10.32" }
  ]
}
```

API key 从哪里取？如果用 opencode 管理 provider，key 存在 `~/.local/share/opencode/auth.json` 的 `deepseek` 条目下，插件直接读它即可，用户无需再配置一次。

## 3. Omarchy 插件结构

Omarchy 的 shell 插件就是一个目录 + `manifest.json` + 若干 QML/脚本，放在 `~/.config/omarchy/plugins/` 下。

### 3.1 采集脚本 collector.sh

```bash
#!/usr/bin/env bash
set -uo pipefail
AUTH_JSON="${OPENCODE_AUTH_JSON:-$HOME/.local/share/opencode/auth.json}"
URL="https://api.deepseek.com/user/balance"
key=$(jq -r '.deepseek.key // empty' "$AUTH_JSON" 2>/dev/null || true)

if [[ -z "$key" ]]; then
  echo '{"status":"no-key"}'; exit 0
fi

out=$(curl -sS -m 10 -w $'\n%{http_code}' \
  -H "Authorization: Bearer $key" -H "Accept: application/json" "$URL")
code=${out##*$'\n'}; body=${out%$'\n'*}
[[ "$code" != "200" ]] && { echo "{\"status\":\"http-$code\"}"; exit 0; }

jq -c --arg now "$(date -u +%Y-%m-%dT%H:%M:%SZ)" '
  .balance_infos[0] as $b |
  { status:"ok", currency:($b.currency//"CNY"),
    total:($b.total_balance//"0"),
    granted:($b.granted_balance//"0"),
    updatedAt:$now }' <<<"$body"
```

### 3.2 manifest.json

```json
{
  "schemaVersion": 1,
  "id": "local.deepseek-balance",
  "name": "DeepSeek Balance",
  "version": "1.0.0",
  "description": "Shows your DeepSeek API balance in the Omarchy bar.",
  "kinds": ["bar-widget"],
  "entryPoints": { "barWidget": "BalanceWidget.qml" },
  "barWidget": {
    "displayName": "DeepSeek Balance",
    "description": "Shows your DeepSeek API account balance.",
    "category": "AI",
    "allowMultiple": false,
    "defaultSection": "right"
  }
}
```

## 4. QML：一个能显示文本的 BarWidget

关键是要知道 Omarchy bar widget 该怎么写。参考系统自带 widget（如 clock 用 `WidgetButton` 的 `text` 显示时间），照葫芦画瓢：

- 继承 `BarWidget`，设 `moduleName` 与 manifest id 一致
- 用 `WidgetButton` 承载文本
- 用 `Process` + `StdioCollector` 调脚本读输出（**不是** `readStdout`，那是错的——Quickshell 用 `stdout: StdioCollector`）
- 用 `Timer` 定时刷新

核心片段：

```qml
BarWidget {
  id: root
  moduleName: "local.deepseek-balance"

  property string status: ""
  property string currency: ""
  property string total: ""

  readonly property string displayText: {
    if (status === "ok") {
      var sym = currency === "USD" ? "$" : "¥"
      var n = parseFloat(total)
      return isFinite(n) ? sym + n.toFixed(1) : sym + total
    }
    if (status === "no-key") return "DS"
    return "DS·…"
  }

  function refresh() { if (!collector.running) collector.running = true }

  Process {
    id: collector
    command: ["bash", decodeURIComponent(String(Qt.resolvedUrl("collector.sh")).replace(/^file:\/\//, ""))]
    stdout: StdioCollector { id: collectorOutput; waitForEnd: true }
    onExited: function(exitCode) {
      if (exitCode !== 0) { root.status = "exit-" + exitCode; return }
      var out
      try { out = JSON.parse(String(collectorOutput.text || "{}")) } catch (e) { root.status = "parse-error"; return }
      root.status = out.status || ""
      if (out.status === "ok") {
        root.currency = out.currency || "CNY"
        root.total = out.total || "0"
      }
    }
  }

  Timer {
    interval: 120000        // 每 2 分钟
    running: true
    repeat: true
    triggeredOnStart: true
    onTriggered: root.refresh()
  }

  WidgetButton {
    anchors.fill: parent
    bar: root.bar
    text: root.displayText
    labelVisible: true
    hasVisualContent: text !== ""
    horizontalMargin: 8
    onPressed: root.refresh()
  }
}
```

## 5. 放进状态栏

```bash
# 把插件目录放进 ~/.config/omarchy/plugins/ 后
omarchy-shell shell rescanPlugins
omarchy plugin enable local.deepseek-balance
# 加到 bar 的某个 section
omarchy bar move local.deepseek-balance --section right
omarchy restart shell
```

## 6. 踩坑记录

- **`Process.readStdout` 不存在**：Quickshell 的进程输出要挂 `stdout: StdioCollector { waitForEnd: true }`，在 `onExited` 里读 `collector.text`。
- **manifest 的 `entryPoints.barWidget` 必须指向 QML 文件名**，QML 内 `moduleName` 与 id 一致。
- 脚本路径用 `Qt.resolvedUrl("collector.sh")` 解析，插件装到哪都能找到。
- 刷新别太频繁，DeepSeek 余额接口有缓存延迟，2~5 分钟足够。

## 7. 延伸

这个模式（脚本采集 + QML 展示）可复用到任意「把某个 API/命令输出显示到状态栏」的场景——例如网速、天气、股票、服务器状态。Omarchy 的状态栏定制门槛其实很低，核心是搞懂 `BarWidget`/`WidgetButton`/`Process`+`StdioCollector` 这套组合。

