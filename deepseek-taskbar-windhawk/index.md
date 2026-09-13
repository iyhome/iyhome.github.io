# 把 DeepSeek 余额嵌进 Windows 任务栏：自写一个 Windhawk mod


之前给 Linux 的状态栏写过 DeepSeek 余额小部件。回到 Windows，也想在任务栏上随时看到余额。本文记录从「托盘图标太小」一路试错，最后自写一个 **Windhawk mod** 把余额嵌进任务栏时钟的过程。

## 1. 试错：三个方案都不行

| 方案 | 问题 |
|---|---|
| 系统托盘纯字符图标 | 尺寸约 16px，字太小，看不清 |
| 桌面悬浮窗 | 大字号清晰，但用户讨厌悬浮窗 |
| 任务栏贴窗（TrafficMonitor 式） | 会被图标/托盘挤压或覆盖，非真正嵌入 |

## 2. 最终方案：嵌入任务栏时钟

Win11 时钟区只有两行（上=时间，下=日期），且不响应文本内换行。于是把**余额拼到上行时间后面**（去掉秒省宽度），下行保留完整日期。

实现是一个 Windhawk mod：

- `@include explorer.exe`，钩住 kernel32 的 **`GetTimeFormatEx`**
- 先 `FindWindowEx(Shell_TrayWnd, TrayClockWClass)` 拿到时钟窗口线程 id（**必须延迟查找**——mod init 时任务栏还没建好）
- 只在**时钟线程**上，把返回值替换为「原时间去秒 + 空格 + 余额」，其余调用原样放行
- 余额从 `balance.txt` 读（解析 `<ds>…</ds>`，缓存 1 秒）

效果：上行 `上午 HH:mm ¥82.6`，下行原始日期。

## 3. 数据来源：脚本 + 计划任务

余额脚本查 DeepSeek 余额接口，写成 `balance.txt`；计划任务每 2 分钟跑一次。查询失败时**保留上次的值**，不清成 `--`，3 次重试、超时 30s。

## 4. 最大的坑：编译链接用哪个 dll

Windhawk 不会自动编译 `ModsSource` 里的源码（验证过），本地 mod 只能自己编译 + 手动注册。关键：

- 编译器：`Windhawk\Compiler\bin\clang++.exe`，`-include windhawk_api.h`、`-target x86_64-w64-mingw32`、`-DWH_MOD`
- **链接必须针对 `Engine\<版本>\64\windhawk.dll`**，而**不是** `Compiler\...\windhawk-mod-shim.dll`——后者缺 `InternalWh_SetFunctionHook` 等符号，链了会加载失败

```bash
clang++.exe -c <flags> -o mod.o mod.wh.cpp
clang++.exe -shared -target x86_64-w64-mingw32 -o mod.dll mod.o "<Engine>\1.7.3\64\windhawk.dll"
```

安装：DLL 放进 `Engine\Mods\64\`，写注册表项，`Restart-Service Windhawk`。

## 5. 小结

| 环节 | 要点 |
|---|---|
| 展示位置 | 任务栏时钟上行（原生只两行） |
| 技术 | Windhawk mod 钩 `GetTimeFormatEx`，只改时钟线程 |
| 数据 | 脚本 + 计划任务写 `balance.txt` |
| 编译 | 链接 Engine 的 `windhawk.dll`，不是 shim |

核心体会：**Windows 上「把自定义信息塞进系统 UI」的最优解，往往不是贴一个窗口，而是 hook 系统绘制函数**。代价是要摸清目标 API 在哪个线程、哪个进程里跑。

