# 双系统差 8 小时：把硬件时钟交给 Windows（RTC=local）


装了 Windows + Linux 双系统后，很多人会遇到一个经典怪象：**在 Linux 里时间是对的，切回 Windows 就慢/快 8 小时，再切回 Linux 又对了**。这不是时钟坏了，而是两个系统对「主板上的硬件时钟（RTC）到底存什么」理解不一致。本文说清根因，并给出让 Linux 迁就 Windows 的做法。

## 1. 现象

- Windows 与 Linux 时间相差整小时（东八区就是 8 小时）
- 每切换一次系统，时间就跳一次
- NTP 明明同步成功，重启换系统后还是错

## 2. 根因：RTC 里存的是「UTC」还是「本地时间」

电脑关机后，时间靠主板上的 RTC（硬件时钟）维持。问题在于：

| 系统 | 默认把 RTC 当作 | 开机时 |
|---|---|---|
| Windows | **本地时间** | 直接读 RTC 当本地时间显示 |
| Linux | **UTC** | 读 RTC 当 UTC，再按时区换算成本地时间 |

于是两边对同一个 RTC 值的解读差了一个时区。你在 Linux 里同步了 NTP，Linux 按「RTC=UTC」把 UTC 写回 RTC；Windows 却把这串 UTC 当本地时间读，就慢了 8 小时。

## 3. 解法：让 Linux 也把 RTC 当本地时间

`systemd` 提供了开关（需 sudo）：

```bash
sudo timedatectl set-local-rtc 1 --adjust-system-clock
```

`--adjust-system-clock` 会在切换 RTC 语义时同步修正系统时钟，避免瞬间跳变。

验证：

```bash
timedatectl
```

关键几行应是：

```
RTC time: 2026-09-13 21:51:30
RTC in local TZ: yes
```

`RTC in local TZ: yes` 表示 Linux 现在把 RTC 当本地时间读写，与 Windows 一致。底层记录在 `/etc/adjtime` 的最后一行 `LOCAL`。

## 4. 那段 Warning 要不要管

执行后 systemd 会警告「local time zone 无法完全支持，建议用 UTC」。这是常规劝告，**双系统场景下可以忽略**：它的意思是「如果机器跨时区移动、或有夏令时，RTC 的本地时间需要外部修正」。对固定时区的双系统来说，这恰恰是正确选择。

## 5. 想改回 UTC

如果哪天不再需要迁就 Windows，或换成纯 Linux：

```bash
sudo timedatectl set-local-rtc 0 --adjust-system-clock
```

## 6. 小结

| 项 | 值 |
|---|---|
| 目标 | Windows 与 Linux 时间一致 |
| 手段 | `timedatectl set-local-rtc 1 --adjust-system-clock` |
| 验证 | `timedatectl` 显示 `RTC in local TZ: yes` |
| 代价 | systemd 会警告，但双系统下无害 |

一句话：**时间不一致不是时钟的问题，是「RTC 存 UTC 还是本地时间」的约定不同**。双系统要么让 Linux 迁就 Windows（本文），要么在 Windows 侧改注册表让 Windows 用 UTC——前者一条命令更省事。

