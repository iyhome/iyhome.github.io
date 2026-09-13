# Windows 11 遥测瘦身：把回传开关逐个关掉


新装的 Windows 11 会持续回传诊断数据、广告 ID、搜索建议、键入历史等。这些开关散落在服务、注册表、组策略、浏览器策略各处，一个个点太累。本文记录一次系统性的「遥测瘦身」清单，以及可回滚的做法。

## 1. 服务层

禁用三个主要负责回传的服务：

```powershell
Stop-Service DiagTrack, dmwappushservice, WerSvc
Set-Service  DiagTrack -StartupType Disabled
Set-Service  dmwappushservice -StartupType Disabled
Set-Service  WerSvc -StartupType Disabled
```

- `DiagTrack`：Connected User Experiences and Telemetry
- `dmwappushservice`：设备管理 WAP 推送
- `WerSvc`：Windows Error Reporting

## 2. 注册表策略层

一组关键值：

| 项 | 作用 |
|---|---|
| `AllowTelemetry=0` | 遥测级别降到最低 |
| 广告 ID 关闭 | 个性化广告追踪 |
| 活动历史/时间线关闭 | 跨设备活动同步 |
| 云搜索/网页搜索关闭 | 开始菜单的联网搜索 |
| 墨迹与键入数据关闭 | 打字习惯上传 |
| 量身定制的体验关闭 | 基于使用习惯的推荐 |
| 开始菜单建议/提示关闭 | 推广位 |

## 3. 浏览器策略

- **Edge**（未安装也先预置策略）：遥测/诊断数据/导航错误 Web 服务关闭、浏览器登录与同步禁用、安全浏览关闭、购物助手/奖励/推荐/侧边栏关闭。
- **Chrome**：指标/云报告/清理工具报告/URL 匿名化收集/设备指标/远程访问穿越关闭；在线拼写检查、搜索建议、翻译、网络预测、新标签页内容建议、后台运行、地理位置关闭。**但保留 Google 登录与同步**。

## 4. 可回滚：备份与日志

改注册表前先导出：

```powershell
reg export HKLM\SOFTWARE\Policies\Microsoft\Windows\DataCollection backup.reg
```

操作日志也留一份，方便日后核对改了什么。

## 5. 小结

| 层 | 手段 |
|---|---|
| 服务 | 停用并禁用 DiagTrack / dmwappushservice / WerSvc |
| 注册表 | AllowTelemetry=0 + 广告/历史/云搜索/墨迹等 |
| 浏览器 | Edge/Chrome 策略预置 |

原则：**只关「回传与个性化」，不动系统功能**；每步都先导出注册表备份，保持可回滚。遥测瘦身不是玄学，就是把一张散落的清单一次性勾完。

