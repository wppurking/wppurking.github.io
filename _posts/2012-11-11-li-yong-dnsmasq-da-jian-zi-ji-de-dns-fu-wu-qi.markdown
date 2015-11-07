---
title: "利用 Dnsmasq 搭建自己的 DNS 服务器"
date: 2012-10-01 18:14
comments: true
categories: ruby beanstalkd
---
## 前言
最近新买了 iPad mini, 在不断折腾的过程中发现一点让我非常非常的无语… 还是与网络有关, 那就是通常情况下, iPad mini 中的 App Store 连接速度那是相当相当相当的慢的, 慢到我打开 App Store 软件的首页需要登上 20~30s 才会有可能访问成功, 那么下载一个应用的情况可想而知了. 因为在编写 [wyatt_hosts](l1) 的时候, 也为了解决在 iMac 上进行 Mac OS 更新慢的问题, 所以关于 iPad mini 基本上也就同样的解决方法了.

在自己想到折腾一个 dnsmasq 之前, 也在网络上搜索过其他的解决方案, 比如 V2EX DNS 可是对我来说, 在 iPad mini 上我也需要其他的比如 Google Drive 等也能够快速的访问, 而这些问题的解决办法都在那个 hosts 文件中, 要是我能够将 iPad mini 给越狱就最好了…

## 初认
由于 iPad mini 没有越狱, 所以我现在能想到的办法就只有自己搭建一台 dns 服务器了, 在网络上搜索 linux dns, mac os dns 很多情况下是使用 [bind 9](https://www.isc.org/software/bind) 的软件, 当我阅读完一些配置指南以后([例如](http://www.ubuntugeek.com/dns-server-setup-using-bind-in-ubuntu.html)), 说实话, 我吓倒了 T.T 超级复杂…… 然后, 我在某一篇文章中看到另外一段话”使用 dnsmasq 做 dns 缓存”吸引过去了, 这才让我去搜索了一下 [dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html), 哈哈, It is designed to provide DNS and, optionally, DHCP, to a small network. 这太棒了, 和我的目的一样, bind 9 虽然是非常流行并且久经沙场的 dns 服务器, 同时这也意味着我需要为那用不着的 20% 功能去折腾一个庞然大物吗? 所以最后选择了更加轻便的 dnsmasq.

## 安装
在 *nix 下安装软件非常的方便, 因为我是 Mac OS 所以先 `brew search dnsm` 搜索看看有没有现成的, 果然找到, 接下来 `brew install dnsmasq`, 经过一系列脚本的安装, 得到一段话:

> To configure dnsmasq, copy the example configuration to /usr/local/etc/dnsmasq.conf and edit to taste.

> cp /usr/local/Cellar/dnsmasq/2.61/dnsmasq.conf.example /usr/local/etc/dnsmasq.conf

> To load dnsmasq automatically on startup, install and load the provided launchd item as follows:

> sudo cp /usr/local/Cellar/dnsmasq/2.61/homebrew.mxcl.dnsmasq.plist /Library/LaunchDaemons sudo launchctl load -w /Library/LaunchDaemons/homebrew.mxcl.dnsmasq.plist

在 Mac OS 配置自启动除非看了这文档, 否则我还真会糊涂…


## 配置
看到一个 dnsmasq.conf 的文件了吧, 还是需要一点点配置的, 但非常非常的少.

* 配置 dnsmasq 的上游 dns 服务器;(这是一个 dns 缓存, 那么其还是需要有上游服务器进行一次域名解析的)
* 配置系统的 dns 服务器, 将 dnsmasq 设置在首位寻找
* 设置 dnsmasq 需要监听的 IP 地址, 让其他服务器能够找到他

对应上面的三个事情, 只有 4 条配置即可, 不要打开 dnsmasq.conf 看到一大片内容就吓到了.

1. 首先配置 resolv-file=/etc/resolv.dnsmasq.conf 这个参数表示 dnsmasq 会从这个指定的文件中寻找上游 dns 服务器
1. 将 127.0.0.1 添加到 /etc/resolv.conf 文件的第一行中, 让系统首先寻找本地的 dnsmasq 服务器
取消注释的 `strict-order` 表示严格安装 resolv-file 文件中的顺序从上到下进行 DNS 解析, 直到第一个成功解析成功为止
1. 确保注释掉 `no-hosts`, 默认情况下这是注释掉的, dnsmasq 会首先寻找本地的 hosts 文件再去寻找缓存下来的域名, 最后去上游 dns 服务器寻找.
1. 设置 `listen-address=127.0.0.1`, 表示这个 dnsmasq 本机自己使用有效.
1. 这里有一个坑 **listen-addres** , 我爬了好长时间才爬出来..


在这些配置中, listen-address 的参数坑了我好长时间, 最后才能明白如何配置. 例如, 我还需要让局域网内其他的服务器也能够首先访问这个 dnsmasq 来进行域名解析如何配置? `listen-address=192.168.1.100` (dnsmasq 所在服务器局域网内 ip), 好吧, 这样你本机配置的 127.0.0.1 就没效果了… 如果设置为 `listen-address=127.0.0.1` 那局域网内其他服务器就无法访问到这个 dnsmasq 了, 其实应该这样设置 `listen-address=192.168.1.100,127.0.0.1` 这样你就能双方都满足了, 不过需要注意的一点是, 如果 dnsmasq 所在服务器在局域网的 ip 地址变更了与配置文件中的不一样, 那么理所当然的再使用配置文件中的那个 ip, 局域网内其他服务器也就找不到这台 dnsmasq ,也就无法利用本地的 dns 缓存了.

## 汇总
最后来汇总一下, 能够快速的部署起来.

**resolv.conf**
{% highlight bash linenos %}
# 让操作系统去 127.0.0.1 找 dnsmasq
nameserver 127.0.0.1
{% endhighlight %}

**resolv.dnsmasq.conf**
{% highlight bash linenos %}
# 让 v2ex(这些) dns 的地址成为 dnsmasq 的上游 DNS
nameserver 199.91.73.222
nameserver 8.8.8.8
nameserver 8.8.4.4
{% endhighlight %}


**dnsmasq.conf**
{% highlight bash linenos %}
resolv-file=/etc/resolv.dnsmasq.conf
strict-order
listen-address=192.168.1.100,127.0.0.1
{% endhighlight %}


###上面设置好以后, 让我们把 dnsmasq 启动起来吧: 

1. 手动启动: `sudo dnsmasq` just it.
2. Mac OS 开机自启动, 这也是我现在设置的方式. 首先运行 `brew info dnsmasq` 查看软件信息, 看到有一句话 
> To load dnsmasq:
>   sudo launchctl load /Library/LaunchDaemons/homebrew.mxcl.dnsmasq.plist

照做就好了, **注意** 使用不同版本的 Homebrew 会略有不同, 在版本 0.9.4 的时候, 简化了, 不需要自己去 copy 一个 Mac OS 下的 plist 文件了(具体答案可在 /usr/local/Library/Formula/dnsmasq.rb 中找到).

###关闭
如果想关闭, 就按照 *nix 的常规用 `ps ax | grep dns` 找到 pid 然后 `kill  -9 [pid]`就行, 因为 -9 是 SIGKILL 信号. 同时如果想让 dnsmasq 清理掉所有缓存的 dns 记录, 发送一个 SIGHUP(1) 信号给 pid 就好 `kill -1 [pid]` 这些可通过 `man dnsmasq` 来看到.

## 测试
最后就是来测试测试是否起作用了. 这个最简单啦, 直接使用 `dig` 命令吧, 执行大于 2 次, 查看其中返回的 “Query time: x msec” 的结果就好, 如果第二次以后 x=0, 那么就配置成功了.

例子: `dig google.com` 查看详细的结果

{% highlight bash %}
wyatt ~$ dig google.com

; <<>> DiG 9.7.3-P3 <<>> google.com.hk
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 28003
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;google.com.          IN  A

;; ANSWER SECTION:
google.com.       0   IN  A   74.125.235.131

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Thu Nov 22 22:49:42 2012
;; MSG SIZE  rcvd: 47
{% endhighlight %}


`dig baidu.com +short` 仅仅查看解析的 ip

{% highlight bash %}
220.181.111.85
220.181.111.86
123.125.114.144
{% endhighlight %}


将电脑开启 dnsmasq, 然后将 iPad mini 连接进入局域网, 将 DNS 服务处第一个设置成 dnsmasq 服务器所在的 ip, 第二个设置成 V2EX 的, 第三个设置为 8.8.8.8 (用英文,隔开). 这回, 从 App Store 下载应用的时候, 总算能够看到明显的进度条滚动了 T.T







[l1]:https://github.com/wppurking/wyatt_hosts