# Windows10原生输入法驯化小鹤双拼全解析



> 小鹤双拼是一种高效汉字输入法，将声母、韵母压缩为两键，结合形码（鹤形）区分同音字。其双拼方案键位设计合理，支持主流输入法挂接，学习成本低且能显著提升打字速度。
> 本文介绍如何在windows自带输入法上面快速添加小鹤双拼输入法。

##### Step1. 复制以下代码到记事本，保存。

~~~reg
Windows Registry Editor Version 5.00
[HKEY_CURRENT_USER\Software\Microsoft\InputMethod\Settings\CHS]
"LangBar Force On"=dword:00000000
"Enable Double Pinyin"=dword:00000001
"EmoticonTipTriggerCount"=dword:00000001
"HapLastDownloadTime"=hex(b):eb,69,29,59,00,00,00,00
"UserDefinedDoublePinyinScheme0"="FlyPY*2*^*iuvdjhcwfg xmlnpbksqszxkrltvyovt"
"DoublePinyinScheme"=dword:0000000a
"UDLLastUpdatedTime"="2017-05-27 22:01:40"
"UDLCount"=dword:0000018b
"UDLVisibleCount"=dword:0000018b
~~~

##### Step2. 修改后缀名为reg
##### Step3. 双击运行即可

