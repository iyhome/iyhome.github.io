# oh-my-zsh 中 agnoster主题 之 隐藏用户名信息


本文所讨论内容的前提是已经安装好了oh-my-zsh。
当oh-my-zsh使用了agnoster主题之后，每一行路径之前都会出现 用户名@主机名 的无用信息，我们可将其隐藏。
直接编辑agnoster.zsh-theme主题配置文件，命令如下：
~~~bash
vim ~/.oh-my-zsh/themes/agnoster.zsh-theme
~~~

在vim打开的文件中找到以下代码行：
~~~bash
# Context: user@hostname (who am I and where am I)
prompt_context() {
  if [[ "$USER" != "$DEFAULT_USER" || -n "$SSH_CLIENT" ]]; then
  prompt_segment black default "%(!.%{%F{yellow}%}.)%n@%m"
~~~
将 **prompt_segment black default "%(!.%{%F{yellow}%}.)%n@%m"** 注释即可，即  **prompt 前面加 " # "**。
