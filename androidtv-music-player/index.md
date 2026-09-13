# 给电视做一个遥控器友好的离线音乐播放器：选型与工程准备


目标很朴素：在客厅的 Android 电视上，用遥控器流畅地播放 U 盘里的离线音乐，界面好看，并且能识别歌曲**内嵌的歌词和封面**。市面上的 TV 音乐 App 要么在线为主，要么遥控器操作稀烂。于是决定拿一个成熟的开源播放器来改。本文记录选型与工程准备阶段（编译之前的活）。

## 1. 需求拆解

| 需求 | 含义 |
|---|---|
| 遥控器（DPad）流畅操作 | 列表/网格项可聚焦、方向键移动、OK 播放、Back 返回 |
| 离线本地音乐 | 扫描 U 盘/内置存储，不依赖网络 |
| 界面好看 | 十英尺 UI，字号/间距/对比度适配 |
| 内嵌歌词与封面 | 读 ID3 `USLT` / Vorbis `LYRICS` 等标签 |

## 2. 排除：Salt Player 走不通

最初想改「椒盐音乐」（Salt Player），但发现 `iyhome/SaltPlayerSourceForTV` 只是它的**发布/文档/翻译仓库，没有任何 App 源码**；Salt Player 本体是**闭源**的。这条路直接断掉——闭源意味着无法做 TV 焦点适配，也无法补内嵌歌词。

## 3. 选型：Auxio 作为底座

最终选定 **`OxygenCobalt/Auxio`**：

- Kotlin 写的现代本地音乐播放器，结构清晰：一个 `app` 模块 + 一个本地音乐库模块 `musikr`
- 维护活跃，本地音乐引擎成熟（MediaStore 扫描、内嵌封面、播放队列、Android Auto）
- **GPL-3.0**：自用/侧载无碍，公开分发修改版需开源

省去了重造播放内核的工作量。剩下的核心难点其实是两件事：**TV 的 DPad 焦点适配**，以及**内嵌歌词的读取与展示**。

## 4. 工程结构：两个必须递归初始化的子模块

```bash
git clone --depth 1 https://github.com/OxygenCobalt/Auxio.git AuxioTV
cd AuxioTV
git submodule update --init --recursive --depth 1
```

- `media/` → vendored 的 androidx.media3 全量树，**约 642MB**，clone 很慢，注意网络
- `musikr/src/main/cpp/taglib` → taglib（tag `ee1931b`），里面还有 `3rdparty/utfcpp` 子模块

**内嵌歌词/封面的解析，正是由这个 native taglib 负责的**——所以它是本项目的关键路径。

## 5. 编译前的 4 处补丁

仓库默认配置不一定能直接编，需要：

1. `build.gradle`：`target_sdk = 37` → **`36`**（SDK 仓库目前没有 android-37 正式版，否则 `compileSdk` 解析不了）
2. `app/build.gradle` 的 `defaultConfig` 加 `ndk { abiFilters 'arm64-v8a' }`
3. `musikr/build.gradle` 同样加 arm64 filter
4. 新建 `local.properties`：`sdk.dir=<你的 Android SDK 路径>`

> 只为电视出 arm64，能显著缩短编译时间。

## 6. 坑：native taglib 是用 shell 脚本构建的

`musikr/build.gradle` 注册的 `assembleTaglib` 任务会执行 `sh -c ".../build_taglib.sh ..."`。于是：

- **`sh` 必须在 PATH**（Windows 上要 Git 自带的 `usr/bin/sh.exe`；Linux/macOS 天然有）
- `build_taglib.sh` **硬编码编译 4 个 ABI**（x86 / x86_64 / armeabi-v7a / arm64-v8a），**不受 gradle 的 `abiFilters` 影响**——建议改成只编 `arm64-v8a`，否则又慢又容易失败
- 脚本里用 `-j$(nproc)`，某些 `sh` 没有 `nproc`，改成固定值更稳

## 7. 下一步待办

- **先把 debug 编译跑通**（`assembleDebug`），再动 UI——别一上来就改代码
- TV 焦点：给 item 根布局加 `focusable`，设默认焦点，保证上下左右 + OK + Back
- 遥控媒体键：`KEYCODE_MEDIA_PLAY_PAUSE/NEXT/PREVIOUS` 走 MediaSession
- 内嵌歌词：先验证 Auxio 是否读并展示，不支持就在 `musikr`（taglib 层）+ UI 补
- 出包：`assembleRelease`（arm64，R8）后侧载到电视

## 8. 小结

| 决策 | 结论 |
|---|---|
| 底座 | Auxio（GPL-3.0，本地引擎成熟） |
| 主战场 | TV 焦点适配 + 内嵌歌词 |
| 工程关键 | `media`/`taglib` 子模块 + native taglib 的 shell 构建 |
| 第一步 | 先让编译通过，再动 UI |

选型的教训：**先确认「有没有源码」再谈改造**——一个只有发布产物、没有源码的仓库，再心动也只能放弃。改造闭源 App 的时间成本，远高于从成熟开源底座出发。

