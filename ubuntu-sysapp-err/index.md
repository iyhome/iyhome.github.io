# 彻底解决Ubuntu中 “检测到系统程序错误”


> 想要彻底告别这种错误弹窗，就必须禁止`Apport`。

我们直接修改他的配置文件，命令如下：
~~~ bash
sudo gedit /etc/default/apport
~~~
　　
用vim打开的apport内容大致如下：
~~~ bash
sudo service apport start force_start=1
enabled=1
~~~
找到  `enabled=1` ，将其改为 `enabled=0` 。保存关闭即可。
