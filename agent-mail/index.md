# 给 AI 配一个邮箱：Agent Mail 上手


让 AI 能收发邮件，是「把 agent 接进真实工作流」很自然的一步。本文记录腾讯 QQ 邮箱的 Agent Mail（`agently-cli`）从安装到授权的完整流程——重点是**授权时密码不经过 AI**。

## 1. 安装 CLI

通过 npm 全局装（装在用户级的 node 全局目录里，无需 sudo）：

```bash
npm install -g @tencent-qqmail/agently-cli
```

验证：

```bash
agently-cli --help
```

## 2. 安装 Skill

给 agent 用的技能包：

```bash
npx skills add https://agent.qq.com --skill -g -y
```

装到 `~/.agents/skills/agently-mail/`，agent 就能按它的说明调用邮件能力。

## 3. 授权：OAuth 设备码

```bash
agently-cli auth login
```

它会打印一个授权链接（设备码流程），**由你本人在浏览器里登录并授权**——密码不经过 AI，凭证存进系统 keychain。授权完成后 CLI 才拿到 token。

## 4. 验证

```bash
agently-cli +me
```

返回当前用户信息与别名列表，例如：

```json
{
  "ok": true,
  "data": {
    "aliases": [
      { "email": "xxxx@agent.qq.com", "is_primary": true, "name": "xxxx" }
    ]
  }
}
```

## 5. 常用命令

```bash
agently-cli message +list --limit 10
agently-cli message +read --id msg_001
agently-cli message +search --q "keyword"
agently-cli message +send --to a@example.com --subject "Hi" --body "Hello"
agently-cli message +reply --id msg_001 --body "Thanks!"
agently-cli attachment +upload --file ./report.pdf
```

## 6. 小结

| 步骤 | 命令 |
|---|---|
| 装 CLI | `npm i -g @tencent-qqmail/agently-cli` |
| 装 Skill | `npx skills add https://agent.qq.com --skill -g -y` |
| 授权 | `agently-cli auth login`（设备码 OAuth） |
| 验证 | `agently-cli +me` |

**核心**：把「凭证授权」这一步交给人在浏览器完成（OAuth 设备码），AI 只拿到 token、碰不到密码——这是让 agent 接触真实服务时比较稳妥的模式。

