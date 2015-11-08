---
layout: post
title: Bash 的文件加载流程
date: 2015-06-17 22:29:23
tags: [tech ubuntu linux bash]
---
## 起因
在弄明白 Ubuntu 的启动流程后(其实是粗略的理解整儿计算机的启动流程), 知道 Bash 的运行也有其自己的一个文件加载的流程, 其中涉及的文件还是有几个的, 那么记录下关键点来提醒自己. 

在理解 Bash 的加载流程, 则需要先理解 Bash 的两个概念:

* 交互模式
* Login shell


## 交互模式

### 区别
这里定义了两个交互方式, 他们分别对应着不同的需求

1.  Interactive (交互模式):  在交互模式下, shell 可以接受用户从键盘的输入. bash 默认进入交互模式(或 bash -i). 你通过 ssh 登陆到远程机器输入命令是交互模式. 你在机器本地登陆通过键盘输入命令是交互模式. 你安装 Ubuntu 桌面端, 登陆图形界面也是交互模式.
1. non-interactively (非交互模式): 在非交互模式, shell 用于执行命令或者读取已经存在的文件 bash -c 自动进入非交互模式. 使用 `bash hello.sh` 默认是非交互模式. 执行单行简单 bash 脚本 `bash -c 'echo "hello"'` 是飞交互模式

### 判断是否交互模式
上面只是提到一些常用的场景, 那么如果想自己尝试一下是否真的为交互模式怎么去弄呢? 在 Bash 中的交互模式会使用 `$-` 这个标示符号, 如果是那么则个就存在, 在 Ubuntu 中也被扩展了一个 `$PS1` 也可以标示是否交互模式. 这段文件出自 GUN Bash 的官方文档.

![ubuntu_bash_interactive]({{ site.img_url }}/ubuntu_bash_interactive.jpg)

### 行为的差异
在交互模式下, bash 增加了很多其他的行为, 例如 job control,  exec 失败不会退出 shell 等等, 摘抄几条重要的:

1. Job Control (see Job Control) is enabled by default. When job control is in effect, Bash ignores the keyboard-generated job control signals SIGTTIN, SIGTTOU, and SIGTSTP.
1. Bash executes the value of the PROMPT_COMMAND variable as a command before printing the primary prompt, $PS1 (see Bash Variables).
1. Readline (see Command Line Editing) is used to read commands from the user’s terminal.

详细的还是请耐心[翻阅一下文档](https://www.gnu.org/software/bash/manual/bash.html#Interactive-Shell-Behavior).


## Shell 是否登陆
login shell 是指用户以非图形化界面或者以 ssh 登陆到机器上时获得的 `第一个` shell，简单些说就是需要输入用户名和密码的 shell。因此通常不管以何种方式登陆机器后用户获得的第一个 shell 就是 login shell。

就是在执行 bash 的时候, 到底要不要进行一次登陆.  区分一下是否需要登陆的情况:

### Login Shell

1. 服务器重启以后,  那你第一次进入系统 的 shell (bash) 则一定是 login shell
1. ssh 连接到远程服务器上,  需要输入用户密码, 登陆了  login shell

### Non-login shell

1. 已经登陆后, 重新开启一个 bash 执行脚本
1. 类似 iterm 这样软件, 开启多窗口

如果到这里还有一些疑惑, 请阅读[这篇 Blog](http://feihu.me/blog/2014/env-problem-when-ssh-executing-command-on-remote/#non-interactive--login-shell).


## 图表化文件加载流程

![ubuntu_bash_file_flow]({{ site.img_url }}/ubuntu_bash_file_flow.jpg)

## Ubuntu 中的 Bash init 特例
如果仅仅是论 Bash 的话, 他的加载行为就应该是上面图标所示的. 但在具体在 Ubuntu 14.04 LTS 服务上实践的时候发现与真实情况还是有一些差别. 这些行为不一样的原因其实来来自 Ubuntu 的版本更新与老脚本的兼容, 例如在 Ubuntu 上使用 shebang 如 `#!/bin/sh` 的时候, 虽然 Ubuntu 14.04 LTS 已经使用 Bash 为默认的 Shell 了, 但他会去模拟 sh 的行为. 就是这些行为使得 Ubuntu 上的 Bash 文件加载流程会与默认不一样, 但却通过代码的方式在行为上接近.

![ubuntu_bash_file_flow_default]({{ site.img_url }}/ubuntu_bash_file_flow_default.jpg)

**脚本中退出** 的意思如下图

![ubuntu_bash_quit]({{ site.img_url }}/ubuntu_bash_quit.jpg)

在 /etc/bash.bashrc 与 ~/.bashrc 中都有  `[ -z "$PS1" ] && return` 的语句, 如果没有 `$PS1` 啥都不做,
这也变相的符合了默认的 bash 的模式.
