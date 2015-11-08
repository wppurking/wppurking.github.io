---
layout: post
title: Ubuntu 的启动流程
date: 2015-06-11 23:49:23
tags: [tech]
---
## 起因
最近在处理 Ubuntu 服务器上的一些问题的时候, 碰到一些列的 Bash 的环境变量有关的问题, 为了弄清楚这里面的细节着实是翻了一系列的资料. 我要弄清楚 Bash 的环境变量的东西, 就要弄清楚 Ubuntu 是在何时让 Bash 启动的, 而要理解这个那么理解 Ubuntu Boot 流程是比不可少, 最详细的文档[在这可翻阅][ubuntu-sysvinit], 这里是做一些文章的简介, 关键信息的抽取.

## 历史遗留问题
Ubuntu 内的所有进程的启动是拥有两套程序的, 一套为历史的 System V 的 init 进程, 一套为 2006 后 Ubuntu 自己编写的 Upstart 的 init 进程.  因为 Ubuntu 系统使用的人众多, 所以他不可能做一刀切的形式将 System V 全部修改为 Upstart 的方式, 所以从 Ubuntu 6.10 开始提供的过度方法就为, 底层切换为使用 Upstart 的 init 进程, 但配置文件以及命令行等做了对 System V 的兼容.


## System V 的相关点

* /etc/inittab : 启动入口
* /etc/rc*.d  : 在操作系统的不同运行级别的自启动脚本的软连接.   (运行级别 0~6)
* /etc/init.d/ : 管理所有的自启动脚本
* service :  System V 的查看命令, 脚本启动可以通过这个命令完成, 例如: service nginx start

这里的原理是, 初始化的地方在 inittab, 然后不同的应用的启动脚本在 /etc/init.d 中, 然后根据不同的系统运行级别, 将启动脚本使用软连接的方式连接到 /etc/rc*.d 中


## Upstart 的相关点
如果使用 Upstart 来进行系统启动后的进程管理, 就简单多了.  将原来的在 System V 中的根据不同 runlevel(运行级别)的脚本管理, 直接通过配置文件来处理了.  例如看一个 cron 的配置文件:

![cron 配置]({{ site.img_url }}/cron_setting.jpg)

* /etc/init : 所有需要由 upstart 管理的服务配置文件(例子如上图)
* 一组命令, 但常用的都在 Controller Sercies 里面

![upstart commands]({{ site.img_url }}/upstart_commands.jpg)

## 公用配置 
/etc/default : 最后提一下这个目录, 这个是对于 /etc/init 脚本或者 /etc/init.d 脚本一起共用的配置文件, 其目的是两个脚本拥有一些公用的部分, 也是以 server name 来区分

![ubuntu_default_conf]({{ site.img_url }}/ubuntu_default_conf.jpg)

使用的例子:  Docker 
cat /etc/default
![ubuntu_docker_default]({{ site.img_url }}/ubuntu_docker_default.jpg)


## 补充
在 Ubuntu 进入到 15.04 后,  由于 systemd 的先进性, Ubuntu 又将 init 的默认进程, 从 upstart 改为 systemd. 这下与 CentOs, Fedua 等发行版统一起来了.

[SystemdForUpstartUsers](https://wiki.ubuntu.com/SystemdForUpstartUsers)



[ubuntu-sysvinit]: https://help.ubuntu.com/community/UbuntuBootupHowto#Traditional_Sysvinit_and_Before_Ubuntu_6.10