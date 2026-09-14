# Fedora KDE 的中国化补完：日历调休 + 一个自写的天气 ion


换到 Fedora + KDE 之后，有两件小事一直硌应人：

1. 数字时钟的日历里没有中国法定节假日，更没有调休；
2. 天气组件搜不到「东莞长安镇」——KDE 自带的天气源里根本没有中国城市，更别提小镇。

这篇记录把这两件事一起补上：节假日用 kholidays 自定义文件，天气则**手写了一个 KDE Plasma 6 的 ion 插件**接到 Open-Meteo。全部代码在 <https://github.com/iyhome/fedora-kde-cn-setup>。

## 一、日历节假日：给 kholidays 喂一个 plan2 文件

KDE 的日历节假日来自 kholidays 库。它除了内置资源，还会读用户目录：

```
$XDG_DATA_HOME/kf5/libkholidays/plan2/holiday_<区域代码>
```

所以只要把自己的节假日文件丢进去、再选中对应区域即可。数据源用 [shuyz 的中国节假日 ICS](https://www.shuyz.com/githubfiles/china-holiday-calender/master/holidayCal.ics)，但 kholidays 用的不是 ICS，而是一种叫 **plan2** 的文本格式：

```
"春节" public on february 15 2026
"补班" public on february 14 2026
```

写个脚本把 ICS 逐条转过去就行（`holiday/ics_to_kholidays.py`）。名字做了缩短——原始 ICS 里是「春节 假期 第1天/共9天」，塞进日历格子根本显示不下，索性假期只留节日名，调休上班日统一叫「补班」。装完点开时钟即可看到：

```bash
bash holiday/install.sh
```

## 二、天气：为什么不是彩云，也不是中央气象台

一开始想用彩云，理由很朴素：数据准、覆盖到镇。结果发现 KDE 天气组件只认一种叫 **ion** 的插件，系统自带 `wetter.com / noaa / dwd / envcan / bbcukmet`，**没有彩云**，整个 KDE 生态也没有现成的彩云 ion。自己写 ion 倒也行，但彩云要 API token——我没有。

退而求其次看 [plasma-ions-china](https://github.com/arenekosreal/plasma-ions-china)（中央气象台 / 和风天气）。中央气象台不需要 key，我满怀希望地拿它的搜索接口一测：

- `北京` → 搜得到
- `上海` / `深圳` → 搜得到
- `东莞` → **搜不到**（返回的还是广东其它城市）
- `长安镇` / `虎门` / `厚街` → 全搜不到

得，中央气象台的站点库里压根没有东莞。

最后落到 **Open-Meteo**：免费、无需 key、按经纬度取数、国内可直连。它的地名库里还真有「长安广场 广东 22.815,113.797」——就在长安镇。于是决定自己写 ion。

## 三、手搓一个 KDE Plasma 6 天气 ion

现代 ion 的接口很小，继承 `Ion` 实现两个方法：

```cpp
void findPlaces(promise, searchString) override;   // 搜索地点
void fetchForecast(promise, placeInfo) override;   // 取天气
```

- `findPlaces`：Open-Meteo 没有地名搜索，干脆固定返回一个位置——坐标从 `~/.config/plasma-weather-openmeteo/location` 读，默认长安镇 `113.80,22.81`；
- `fetchForecast`：请求 Open-Meteo 的 `/v1/forecast`，把 WMO 天气码映射成 KDE 图标 + 中文描述，填进 KDE 的 `Forecast` 结构。

编译安装（需要 sudo 装 `-devel` 依赖）：

```bash
bash weather-ion/build.sh
systemctl --user restart plasma-plasmashell
```

重启后天气组件里就会多出 **Open-Meteo**，选中即可。弹窗里现在是这样：**东莞·长安镇 25°C 毛毛雨**，五天预报，全中文。

## 四、踩过的坑（给后来者）

1. **插件 id 来自文件名**。KPlugin 规范明确写着「嵌入式元数据不要自己写 Id，id 从文件 basename 派生」。所以 `openmeteo.so` 对应的 provider 值就是 `openmeteo`。
2. **`QJsonArray` 没有 `.value(int)`**。Qt6 里得用 `.at(i)`，我在这上面编译报错了一轮。
3. **别用系统的 `PlasmaWeatherConfig.cmake`**。它 `find_dependency(KF6Holidays)`，而 Fedora 默认只有 kf6-kholidays 运行库、没有 `kf6-kholidays-devel`，直接配置失败。仓库里自带了 `FindPlasmaWeather.cmake` 绕开它。
4. **改 `plasma-org.kde.plasma.desktop-appletsrc` 前必须停 plasmashell**，否则退出时会被覆盖。

## 五、顺带踩坑并放弃的东西

折腾过程中还试了 **Panel Colorizer**：它能让面板做毛玻璃、圆角、托盘图标重着色，看着很诱人。但配置项多到劝退，而且它作用于**整条面板**，一不小心就把左边（开始菜单、任务栏）也一起搞乱，最后索性卸载了。想要「任务栏像 Windows」，Plasma 原生面板设置 + 一套合适的主题其实更省心。

顺带一提，KDE 系统托盘的图标排序在「配置系统托盘 → 条目」里拖动即可；但第三方应用（微信、clash 之类）的托盘图标顺序由应用自己决定，`extraItems/shownItems/hiddenItems` 只对 Plasma 内置组件生效。

---

**仓库**：<https://github.com/iyhome/fedora-kde-cn-setup>
含：Open-Meteo ion 源码、节假日 ICS→kholidays 转换脚本、以及一份「新装 Fedora 照着做就能还原」的总纲。

