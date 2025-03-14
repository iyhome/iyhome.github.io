# Arch Linux极简主义安装指南



> 先说说我为什么开始入坑archlinux的吧，我最喜欢这个系统的一点就是简洁，DIY程度高，可以定制真正属于自己的专用系统。像gentoo的话，就为了日常使用也没必要那么折腾，除非你是想在折腾的过程中搞清楚其中的原理，那也还是可以的。在这里，我就不说怎么装archlinux了，官方文档和各类教程（视频的，图片加文字的）在网上都有。我就简单说一下一些小想法吧。

##### 设置大号字体
~~~bash
setfont /usr/share/kbd/consolefonts/LatGrkCyr-12*22.psfu.gz
~~~

##### Archlinux一般需要分四个分区
- EFI引导分区(100~300Mb)
- 更目录分区
- Home目录(尽可能大)
- Swap交换分区(1~2Gb,64位系统可以不要)

##### 查看当前磁盘分区情况
~~~bash
lsblk -f
~~~
![](https://raw.githubusercontent.com/iyhome/iyhome.github.io/refs/heads/master/_posts/assets/20030301.png)

##### 对新手比较友好的分区方式
~~~bash
cgdisk /dev/sda
~~~
这个分区方式就对新手比较友好，按照官方文档的分区，新手真的会把脑子搞晕。上面的“sda”是目标分区，你自己看着来，别一葫芦画瓢按我的来。然后efi分区的编号是ef00,这个东西也不是很重要，你可以自己查找的。
用上面的方法分完区后,还需要对分区进行格式化。

##### 格式化分区
- EFI分区:
    ~~~bash
    mkfs.fat -F32 /dev/sda1
    ~~~
- 根目录和家目录:
    ~~~bash
    mkfs.ext4 /dev/sda2
    ~~~

##### 开机自启DHCPCD
~~~bash
Systemctl enable dhcpcd 
~~~
> 没错，Systemctl enable 软件名可以设置你想要开机自启的软件包。

##### 在u盘系统给硬盘的系统装软件
~~~bash
pacstrap /mnt vim
~~~
当然在此之前，你需要先挂载分区。
另外，根据本人经历，有好几次安装完ArchLinux重启后都连接不了网络，或其他问题，保险起见，建议事先安装的软件包：
- 文本编辑器
- dialog
- wpa_supplicant
- netctl
- iw
- dhcpcd

##### 修改镜像源的新姿势
~~~bash
wget -O /etc/pacman.d/mirrorlist https://www.archlinux.org/mirrorlist/?country-CN&protocol=http&protocol=https&ip_version=4
~~~
上面的操作是直接下载一个只有大陆源的镜像文件替换掉原来的，省去了查找的麻烦，但是有点长。这个文件的每一行都是有#注释的，所以需要将注释符去掉。你可以用文本编辑器手动去，也可以用命令去。这里以用vim手动去除为例：
~~~bash
vim /etc/pacman.d/mirrorlist
~~~
然后在命令模式下，用x来删除单个字符，然后用:wq或者:x保存退出。你也可以用 cat 或者nl等命令查看是否修改成功。最后更新源好了。

##### Archlinux 删除软件命令
- 删除单个软件包，保留其全部已经安装的依赖关系：
~~~bash
Pacman -R 软件名
~~~
- 删除指定软件包，及其所有没有被其他已安装软件包使用的依赖关系
~~~bash
pacman -Rs 软件名
~~~

##### Arch Linux中文乱码解决
~~~bash
# 1.安装中文字体:
#  乱码的原因就是缺少中文字体的支持，下载文泉驿xx，OK了
pacman -S wqy-zenhei ttf-fireflysung
#  设置en_US.UTF8 UTF-8 zh_CN.UTF8 UTF-8
/etc/locale.gen

# 2.执行下面命令看看是否配置好了中文编码:
locale-gen 
locale
locale -a
# 3.在/etc/rc.conf 中编辑下列内容：
LOCALE=en_US.UTF-8
# 操作完毕，如果没效果，那么请重启。
~~~

##### 查看当前GUI
~~~bash
echo $GDMSESSION
~~~

##### 安装图形界面
###### KDE
1. 先安装xorg
   ~~~bash
    pacman -S xorg xorg-server xorg-xinit xorg-apps
   ~~~
2. 安装完毕后，使用下面的命令安装KDE
   ~~~bash
    pacman -S plasma kde-applications
    pacman -S sddm sddm-kcm
    # 如果是双显卡游戏本,还需安装以下软件
    pacman -S nvidia bumblebee
   ~~~
3. 此时在~/.xinitrc文件中添加exec startkde一句，此时已经可以通过startx命令进入桌面环境，但推荐使用启动管理器代替这一步startx
   ~~~bash
   systemctl enable sddm
   ~~~
4. 即刻启动桌面
~~~bash
systemctl start sddm
~~~
###### Deepin-Desktop
1. 先安装xorg
   ~~~bash
    pacman -S xorg xorg-server xorg-xinit xorg-apps
   ~~~
2. 安装Deepin软件包
    ~~~bash
    pacman -S deepin deepin-extra lightdm
    ~~~
3. 开机自启
   ~~~bash
    systemctl enable lightdm
   ~~~
4. 即刻启动
   ~~~bash
    systemctl enable lightdm
   ~~~
