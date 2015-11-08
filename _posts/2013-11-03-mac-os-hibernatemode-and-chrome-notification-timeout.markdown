---
layout: default
title: "Mac OS hibernatemode and Chrome Notification Timeout"
date: 2013-11-03 13:08
comments: true
categories: mountain lion Chrome hibernatemode
---
### 起因和问题
最近将 MacBook Air 从原来的 10.7 升级到了 10.8 (没有升级 10.9 是因为太新有些工具不兼容), 然后就碰到类似 ["Mountain Lion 关机速度慢"](http://blog.csdn.net/cneducation/article/details/8293224), ["Mountain Lion 休眠速度慢"](http://www.mobileai.tw/2012/09/20/mac-osx-ssd-optimization/) 这样的问题困扰, 其实这些问题都挺好解决的, 是一些配置问题, 只是修改这些配置的地方没有那么友好. 不过我在修改了 

![pmset -g live](http://wyatt.qiniudn.com/pmset.png "pmset -g live")

hibernatemode 为 0 之后, 发现休眠的速度仍然很慢, 按道理来说应该不再将数据向硬盘上写了, 应该不会有那么慢的休眠速度的啊, 仔细检查了一下休眠发现:

1. 点击左上角苹果图标后点击 Sleep,  苹果背后灯会亮着直到成功休眠, 屏幕会亮着鼠标加黑色背景直到成功休眠.
2. 盖上盖子, 苹果背后的灯会灭, 并且显示器会黑, 但在 10s 内打开盖子发现还在休眠中… 屏幕会亮着鼠标加黑色背景, 直到成功休眠后才关闭.

### 寻找问题
发现这个情况我就很奇怪了, 这是咋回事? 怎么修改了 `hibernatemode 0` 不起作用吗? 然后我尝试重启电脑后休眠一次, 发现这次很快就休眠过去了. 然够根据这个现象我一时无解了, 我想通过查查 log 信息看看? 可这休眠的 log 哪里找? 我知道 Mac 下有一个 Console 应用能够看到整个系统的 log 但我不知道哪个 log 和这个问题相关啊. 然后我就 `man pmset` 查看设置 `hibernatemode` 的命令, 通过 `pmset -g log` 找到了如下的内容

![pmset -g log](http://wyatt.qiniudn.com/pmset_log.png)

发现问题了! 当出发系统的 Sleep 后, Kernel 在等 Chrome 一直等到超时. 上网找了一会, 发现 [Sleep delay on MBA 2013 after Chrome update](http://forums.macrumors.com/showthread.php?t=1649152) 和 [chromium issues 讨论列表](https://code.google.com/p/chromium/issues/detail?id=132336#c61)  [Issue 25954005](https://codereview.chromium.org/25954005/)

原来这是从 2013.6.12 就被报告出来的一个 bug, 到 2013.10.8 解决.

### 结论
这个是 Chrome 本身的问题, 看来在 Chrome 再次更新前 (现在版本为 30.0.1599.101) 我还是先关闭 Chrome 然后再休眠吧.  如果是盖盖子休眠的话, 其实还真感觉不出来.

PS: 我是开着盖子连着外显, 所以休眠时看到屏幕一直黑着有鼠标不关不爽 - -||

PPS: 日志内容可能是 "PM notification timeout.." 也可能是 "Response from Google Chrome…" 不过都是有 30s 的 timeout 时间.