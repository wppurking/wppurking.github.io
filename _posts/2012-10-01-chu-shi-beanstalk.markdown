---
layout: post
title: "初识 Beanstalk"
date: 2012-10-01 18:14
comments: true
tags: [rails ruby queue]
---
这几天一直在想一个问题 "如何利用 Ruby 维持一个量大的 Queue", 一直在考虑这样的问题, 其实本质上也就是因为 background job 引来的, 当然在使用 [Play!][l2] 的使用对 Job 的理解与在 Rails 中看到的 [Delayed_job][l1] 对 Job 的理解, 也会突然多了另外一个思路.

在 [Play!][l2] 中, 我将 Job 定义为一个个在后台运行的任务进程, 每一个 Job 会自己去数据库寻找需要处理的数据, 然后进行定时或者周期性处理. 而在 Rails 的 [Delayed_job][l1] 中, 是将一个个需要执行的小任务存储到数据库中, 然后定时拿出这一个个任务进行执行, 反过来想想和我解决的另外一个功能比较像, 按照需求申请一个个需要执行的 JobRequest 然后另外一个线程去周期性处理这些 Request. 而这次需要考虑的是因为是比较量大的任务, 所以直接使用数据库可能无法满足. 绞尽脑汁的想来想去, 在互联网上找来找去, 不记得从什么地方找到了[这篇文章](http://adam.heroku.com/past/2010/4/24/beanstalk_a_simple_and_fast_queueing_backend/) 然后就找到了 [Beanstalk][l3] ,突然感觉, 哈~ 这不就是我一直寻找的吗? 一个类似中间件的 Queue 一个可进行分布式部署的 Queue, 再一看这文章中的数据分析, 满足, 满足, 太满足了, 大多数情况就这样一个中间件的 Queue 完全满足了, 而且这家伙就是针对 background job 这样的需求的(当然也看做是 messaging queue). 接下来肯定是拿下来折腾, 弄清楚怎么用.

安装非常简单, 按照 [Beanstalkd 的 Downloads 页面](http://kr.github.com/beanstalkd/download.html) 来就好了, 因为我是 Mac OS 所以就直接使用 `brew install beanstalkd` 安装了.

接下来:

* 如何启动 beanstalkd (包括常用参数)
* 如何与 beanstalkd 交互
* 如何监控 beanstalk 的状态

### 如何启动 beanstalkd
在 *nix 下面, 启动 beanstalkd 真的很容易, 就一条命令足以 `beanstalkd -b ./ &`, `beanstalkd` 为命令, `-b` 为开启 binlog 用来持久化 Queue 中的任务, 避免 Power off 队列任务丢失. `./` 指定输出 binlog 的目录为当前目录, `&` 这个就是让他后台运行.

* `-b DIR` 输出 binlog 保持任务不丢失, DIR 指定路径
* `-l ADDR` 绑定的地址, 默认为 (0.0.0.0)
* `-p PORT` 绑定的端口, 默认为 (11300)
* `-z BYTES` 每一个 job 消息的最大大小, 默认 65535 bytes (64kb, 足够了吧)
* `-s BYTES` 设置单个 binlog 文件的最大大小, 超过会自动切分文件, 默认 10485760 (10mb)
* `-V` 查看输出的日志. (我简单 debug 用)

其中有一个我一直没有弄明白, `-f MS` 的 "fsync at most once every MS milliseconds (use -f0 for "always fsync")" 我不知道这里的 fsync 的是什么内容? 作用是什么? [这个功能的 issue](https://github.com/kr/beanstalkd/issues/16).

### 如何与 beanstalkd 交互
因为在他首页的例子代码是利用的 ruby 中的 [beanstalk-client][l4] gem, 刚好这段时间在学习 Rails, 所以就直接拿 Ruby 测试交互了. 也不知道是不是因为这个太简单了的缘故, 无论是 beanstalkd 或者是他的 beanstalk-client 都没一个 document, 结果上手的时候很尴尬,只好去找 beanstalk-client 的[源代码](https://github.com/kr/beanstalk-client-ruby/blob/master/lib/beanstalk-client/connection.rb). . . 

在使用前, 还是阅读了一下 beanstalkd 的一些细节问题, 由于官方的针对 beanstalk 协议的文档实在太难阅读了, 所以就阅读了一篇[淘宝对 beanstalkd 的介绍](http://rdc.taobao.com/blog/cs/?p=1201) 和 [另外一篇博客](http://devpoga.wordpress.com/2010/10/16/beanstalkd_background_job/), 了解了 Job 的生命周期也就了解了几个需要执行的几个关键方法.

#### 简单的例子代码
{% highlight ruby linenos %}
require "beanstalk-client"
require "pp"

# 初始化一个 tube 的 Queue
# 第二个参数为 tube 的名字, beanstalkd 可以拥有多个不同的 tube
BK = Beanstalk::Pool.new(['localhost:11300'], "test")

def put(job)
	BK.put(job.to_s)
	# 第二个参数为优先级, 数值越小优先级越高
	BK.yput(job, 0)
end

def get()
	begin
		job = BK.reserve(0.1)
		# 或者可以直接那解析 ymal 程 hash. job.ybody
		puts job.body

		# 任务处理完成了, 告诉 beanstalkd 将这个 job 删除.
		job.delete
	rescue Exception => e
		puts e
	end
end

def status()
	# 查看整个 beanstalkd 的状态
	#BK.stats()
	BK.stats_tube('test')
end

job = {id: 1, act: :email, type: :notice, to: "foo@bar.com"}

put(job)
get()

pp status()["total-jobs"]
{% endhighlight %}

知道那几个函数的入口, 其他的详细代码自己怎么想就可以怎么写了, 这个接口设计得真的很简单.



### 如何监控 beanstalk 的状态
在上面的代码例子中, 可以通过程序代码调用 `stats()` 与 `stats_tube(tube_name)` 来查看整个 beanstalkd 与其中某一个 tube 的状态, 因为任务执行的状态也是在使用中大家肯定会非常关心的点, 所以就有一些非常棒的 beanstalk tools 来完成这些功能, [beanstalkd_view](https://github.com/denniskuczynski/beanstalkd_view) 就是利用 Sinatra 实现的非常棒的监控程序, 可以单独运行也可以集成到 Rails 中使用.


好了, beanstalk 的认识已经差不多了, 接下来会将这个引入到系统中正常的使用了.


[l1]: https://github.com/collectiveidea/delayed_job "Delayed_job"
[l2]: http://www.playframework.org "Play!"
[l3]: http://kr.github.com/beanstalkd/ "Beanstalk"
[l4]: https://github.com/kr/beanstalk-client-ruby "beanstalk-client"