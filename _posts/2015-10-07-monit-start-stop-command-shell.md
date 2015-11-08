---
layout: post
title: Monit  的 start/stop 所产生的 shell 环境
date: 2015-10-07 13:34:21
---
## 问题
这个问题产生在一次使用 Monit 为 ruby 的 thin 设置进程的重复启动的问题. 在我的理解里面, 我认为 Monit 使用 start/stop 等命令是启动了一个 non-interactive, non-login 的 shell, 因为他不需要认为输入什么信息, 也不用登陆, 因为从当前进程直接启动的.

## 猜测
按照 Ubuntu 中 Bash 的加载路径来猜测的话,  会加载 /etc/bash.bashrc 以及 ~/.bashrc , 但都会因为脚本中第一行拥有 $PS1 (non-interactive) 的判断, 进入脚本就会返回退出不做处理, 但我可以将 rbenv 中的判断放到 ~/.bashrc 的判断的前面:

![monit_start_stop_1]({{ site.img_url }}/monit_start_stop_1.jpg)

但是经过测试, 这样做也是无效!  可以看到执行 env 后的结果

![monit_start_stop_2]({{ site.img_url }}/monit_start_stop_2.jpg)

rbenv 并没有被添加到执行路径中, 而且也没有 $HOME 变量, 但有很多 Monit 的变量. 

然后我想了会, 难道 Ubuntu 做了 sh 的兼容, 没有加载 ~/.bashrc ? 但感觉这也不对啊, 我使用 /bin/bash -c 执行也是这样啊? 


## 原因
想不出, 猜不到, 最后只能去翻代码...  然后就找到了这个坑爹的文档中没有注明的结论: 
[env.c](https://bitbucket.org/tildeslash/monit/src/d5b6cc92c6714e3c66d83e3ecf7f45dfbef6ab59/src/env.c?at=master&fileviewer=file-view-default)

![monit_start_stop_3]({{ site.img_url }}/monit_start_stop_3.jpg)

以及这个详细的 [commit 描述](https://bitbucket.org/tildeslash/monit/commits/cd545838378517f84bdb0989cadf461a19d8ba11)

![monit_start_stop_4]({{ site.img_url }}/monit_start_stop_4.jpg)

找到问题原因了, 那最后的解决就有方法了.  只能在执行命令的时候, 手动将 rbenv 加入到 PATH 里面去进行手动的初始化工作, 那么可以有两种方案:

1. 使用固定的代码将 rbenv 与 rbenv 的 shims 添加到 PATH 变量中, 然后调用 thin
1. 将 thin 的启动/停止写到一个单独的 bash 脚本中, 同时这个 bash 脚本内做 rbenv 的初始化

这个看喜好吧, 其实最终这个项目利用 monit 对 thin 的进程监控的配置文件也会添加到项目的代码 repo 中, 那么再编写一个 bash 脚本其实也无妨.
