# 把系统回滚做进开机菜单：Limine 引导器与 Btrfs 快照


「系统搞坏了能不能一键回到昨天」——这件事最优雅的实现，是让快照直接出现在**开机引导菜单**里。Omarchy 用 Limine 引导器 + Btrfs 快照做到了这一点。本文介绍 Limine，以及它比 GRUB 强在哪。

## 1. Limine 是什么

**Limine** 是一个现代、可移植的 bootloader，支持 BIOS/UEFI、多系统、Btrfs 快照回滚，界面是简洁的文本菜单。定位是「替代 GRUB 的新一代轻量引导器」，被 Omarchy 选为默认。

| | Limine | GRUB |
|---|---|---|
| 复杂度 | 简单，配置直观 | 复杂，脚本化 |
| 依赖 | 极少 | 较多 |
| 配置格式 | 简洁的 `limine.conf` | `grub.cfg`（脚本生成） |
| 快照集成 | 官方 `limine-snapper-sync` | `grub-btrfs` |

## 2. 最大亮点：与 Btrfs 快照集成

配合 `limine-snapper-sync` + Btrfs/Snapper：

- 系统每次变更（如 `pacman` 更新）自动创建快照
- 快照自动出现在**开机引导菜单**
- 开机时可选任意快照启动 → 系统坏了能一键回到过去

常用命令：

```bash
limine-snapper-list      # 列出快照
limine-snapper-info      # 快照信息
limine-snapper-restore   # 恢复
limine-snapper-sync      # 手动同步快照进引导菜单
```

> 根分区必须是 Btrfs。Ubuntu/Fedora 默认 ext4，要用快照得先装成 Btrfs，或改用 Timeshift（rsync 模式，无引导菜单集成）。

## 3. 配置集中在两处

```bash
# /etc/default/limine —— 主配置
ESP_PATH="/boot"
KERNEL_CMDLINE[default]+="cryptdevice=UUID=...:omarchy_root root=/dev/mapper/omarchy_root rootflags=subvol=@ rw rootfstype=btrfs"

# /etc/limine-entry-tool.d/omarchy-defaults.conf —— 分项
FIND_BOOTLOADERS=yes                 # 自动发现其它系统(如 Windows)
BOOT_ORDER="*, *fallback, Snapshots"
MAX_SNAPSHOT_ENTRIES=6
```

- `FIND_BOOTLOADERS=yes`：自动把 Windows 等其它系统加进菜单
- `BOOT_ORDER`：菜单排序，`Snapshots` 放最后
- UKI（`ENABLE_UKI=yes`）：内核 + initramfs + 命令行打包成单个 `.efi`，更干净、适合 Secure Boot

## 4. 多系统：删 Linux 不会误伤 Windows

本机 Windows 引导在 `nvme1n1p1` 的 ESP，Limine 在 `nvme1n1p4` 的 `OMARCHY_EFI`，**两者在不同分区**。所以删掉 Linux 分区不会删掉 Windows 引导：

```bash
efibootmgr                          # 查看启动项
sudo efibootmgr -o 0000,0003        # 调整顺序(Windows 第一)
sudo efibootmgr -b 0003 -B          # 删除 Limine 残留项
```

## 5. 小结

- Limine = 现代轻量引导器，比 GRUB 简单，被 Omarchy 采用
- 最大亮点：**Btrfs 快照集成（开机可回滚）+ 多系统自动发现**
- 配置集中在 `/etc/default/limine` + `/etc/limine-entry-tool.d/`
- 删 Linux 后切回 Windows 引导很简单，不会误伤

**一句话**：把「系统快照」变成引导菜单里的一个选项，是给「折腾党」最好的保险——手滑改坏系统时，重启选一个快照即可。

