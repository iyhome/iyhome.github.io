# Deepin 15.11 闭源N卡驱动安装指南：双显卡避坑手册



> deepin（深度）是一款由武汉深之度公司研发的一款适合国人日常学习的linux系统，其UI精美，美过Mac。它对于中国用户的一个亮点就是QQ微信等国软件傻瓜式安装（类似安卓应用商店安装），如果在Ubuntu上自己来安装wine的话就麻烦了。而且，它自带的应用商店拥有许多我们国人经常要用的各方面的软件，一键安装，比windows安装还要简单，大大节省了我们的时间成本。所以，本人看中了这款系统。但是对于双显卡（n卡）电脑，n卡驱动是需要自己装的。（本人电脑Dell 游匣5577，双显卡，Inter(R) HD Graphics 630 和 英伟达 1050。不装N卡的话，关机和关机卡死）。本人综合各大论坛的相关内容，实践出了以下方法。

##### 一、驱动下载
本教程以GTX1050为例，前往[N卡官网](https://www.nvidia.cn/Download/index.aspx?lang=cn)下载对应的驱动;
下载好的驱动名应该与这个类似：NVIDIA-Linux-x86_64-430.26.run。为方便起见，建议更改文件名为a.run，千万别忘了.run，改文件名只是为了好敲入命令。

#### 二、禁用nouveau驱动
1. 如果之前在Deepin中安装过NVIDIA驱动，请将其全部删除：
> 没有安装过该驱动请跳过

~~~bash
sudo apt autoremove nvidia　　　　
~~~
2. 用dedit修改文件:
> dedit是deepin自带的文本编辑器，gedit是Ubuntu自带的文本编辑器，当然你vim足够熟练你也可以用vim.

~~~ bash 
sudo dedit /etc/modprobe.d/blacklist.conf
~~~
3. 然后在将以下内容复制到文件中,保存退出
~~~bash
# /etc/modprobe.d/blacklist.conf
  blacklist nouveau
  blacklist lbm-nouveau
  options nouveau modeset=0
  alias nouveau off
  alias lbm-nouveau off
~~~　　　　
4. 接下来在终端执行命令:
~~~bash
   sudo update-initramfs -u
~~~
5. 重启系统，再次进入系统，可能会发现分辨率异常。
> 分辨率异常就说明成功禁用nouveau驱动

#### 三、NVIDIA安装过程
1. 进入超级终端：
  使用快捷键CTRL+ALT+F2进入超级终端，登录自己的账号。
  > 就是自己deepin下的用户名和密码

2. 暂时关闭图形界面：
~~~bash 
sudo service lightdm stop
~~~~
3. 给下载好的nvidia驱动文件设置执行权限：
  所需操作命令：
  - cd 进入目录
  - ls 查看目录内的内容
> 若没有改浏览器的下载路径的话，那路径一般是/home/（你的用户名）/Downloads , 如果不明白什么意思，那事先在图形界面找到放驱动的文件夹，然后在该文件夹里面鼠标右键，打开终端，然后输入pwd，然后回车，就会显示出驱动的当前目录。

4. 用以上两个命令进入驱动文件的目录，然后输入以下命令：
~~~bash
# 看你之前给文件的命名是什么，这里就输什么
sudo chmod a+x a.run（）
~~~

5. 驱动安装：
~~~bash
# 这个时候会出现一个页面，一系列yes，还有一个界面选择install and cover，意为安装和覆盖。然后等待几分钟。
sudo sh a.run
~~~

6. 重启系统：
~~~ bash
sudo reboot
~~~

#### 完成N卡驱动的安装🎉
