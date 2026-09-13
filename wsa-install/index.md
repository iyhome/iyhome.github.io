# 在 Windows 上跑安卓：WSA 安装记（WSABuilds）


想在 Windows 上直接跑几个安卓 App，于是折腾了 WSA（Windows Subsystem for Android）。微软官方渠道已停，社区维护的 **WSABuilds** 成了主力。本文记录一次从下载到冒烟测试的全过程。

## 1. 选变体

WSABuilds 提供多个变体，按需选：

- **含 GApps**（带 Google 服务）还是无 GApps
- **无 Root** 还是带 Root
- 架构：x64

最终选定：`WSA_2407.40000.4.0_x64_Release-Nightly-GApps-13.0.7z`（约 776MB，含 GApps，无 Root）。

## 2. 前置条件

- Windows 11 x64，**VirtualMachinePlatform** 已启用
- BIOS 里 VT（虚拟化）已开
- 目标盘空间充足

## 3. 下载：先解决被墙

GitHub 的 `/releases/download` 在国内常直连超时。实测可用代理：

```
https://ghfast.top/<原始 github url>
```

用 `curl` 配合即可（实测 ~35MB/s）。注意：其他几个常见镜像当时都失效了，多试几个。

## 4. 解压与注册

- 解压用 Windows 原生 `7zr.exe`（避免走 WSL 的 9P 慢写）；740MB → 2.3GB 只用了几秒
- 注册用 `Add-AppxPackage -Register .\AppxManifest.xml`
- 官方 `Install.ps1` 有多层进程 + 交互式 `ReadKey`，不适合自动化；**自写一个提权脚本更干净**

## 5. 验证

```powershell
Get-AppxPackage MicrosoftCorporationII.WindowsSubsystemForAndroid
```

看到 `Status=Ok`、`InstallLocation=C:\Programs\Android\WSA` 即可。再做个冒烟测试：启动 `wsa://com.android.settings`，确认 `WsaClient` 正常、关闭无残留进程。

## 6. 小结

| 步骤 | 关键 |
|---|---|
| 选型 | GApps / 无 Root / x64 |
| 下载 | 走 `ghfast.top` 代理 |
| 解压 | 原生 `7zr`，别走 9P |
| 注册 | 自写提权脚本，绕开交互式 Install.ps1 |
| 验证 | AppxPackage 状态 + 启动冒烟 |

一句话：**WSA 本身不难，难的是「在国内把安装包弄下来」和「绕开官方脚本的交互」**。把这两点解决，剩下的就是标准的 Appx 注册流程。

