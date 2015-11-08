---
layout: default
title: "利用 EventMachine 和 Fiber 实现 Socks 5 代理(1)"
date: 2013-07-12 22:26
comments: true
categories: 
---
## 起因
最近看到 [Shadowsocks](https://github.com/clowwindy/shadowsocks) 这个项目, 发现他其实挺火的, 当然这个项目是用来科学上网用的, 核心就是通过 Socks 代理来达到翻墙的目的, 当然代理中间还做了一些混淆处理, 核心部分有:

* 用于提供本地 Socks 代理的本地代理客户端
* 用于处理客户端代理请求的远端代理服务器
* 在本地代理服务器与远端服务器之间的加密

以前了解 GoAgent 等等其他科学上网的工具, 都发现很复杂, 但了解这个工具后发现很好用很简单, 以至于 [Node.js 版本](https://github.com/clowwindy/shadowsocks-nodejs), [Golang 版本](https://github.com/shadowsocks/shadowsocks-go), [Ruby 版本](https://github.com/clowwindy/shadowsocks-ruby) 都层出不穷, 正好看到了 [stochastic-socks](https://github.com/luikore/stochastic-socks) 这个项目所以就想实现一下学习一个 Ruby 中的 Fiber 和 EventMachine.


## 目的
这个项目的目的是为了学习如何使用 Ruby 中的 Fiber 与 EventMachine 所以现已实现客户端为主要方向.

## 认识基本的 EM 知识

### EM 作为 Client
在接触 EventMachine (后称 EM) 的时候, 第一个例子一般都是教你如何将 EM 启动起来, 然后能够让其接受外部请求过来的 Socket 连接, 注意是 Socket 连接不是 Socks 哈. 首先阅读 [General Introduction](https://github.com/eventmachine/eventmachine/wiki/General-Introduction) 发现, EM 可以让我们通过 Block/Proc, Module, Class 三种方式来编写自己的代码来插入到 `EM.run` 提供的 Event Loop 中. 我现在还不了解提供这三种方式的目的是什么, 不过我现在仅仅把他理解成三种方式根据不同的情景选择. 为了测试我们的 EM 发出了一个 Socket 连接, 我们先借助 `nc -l 1080` 来简单的查看效果, 每测试一次都需要启动一次(我没找到可以持续监听的参数 - -||) 然后可以分别测试下面的代码:

**proc.rb**
{% highlight ruby %}
require "eventmachine"

proc = ->(c) {
    def c.post_init
        p 'post init'
    end

    def c.connection_completed
        p 'connection complete'
        send_data("hahahahah\n")
        p 'send data'
        close_connection_after_writing
    end


    def c.receive_data(data)
        p '对方没有发送信息, 所以不会触发 Event 不会调用显示'
    end

    def c.unbind
        p 'unbind'
        EM.stop
    end
}

EM.run do
    EM.connect '127.0.0.1', 1080, &proc
end
{% endhighlight %}

**module.rb**
{% highlight ruby %}
module Client
    def post_init
        p 'post init'
    end

    def connection_completed
        send_data("hahahahah\n")
        close_connection_after_writing
    end

    def receive_data(data)
        p 'won`t trigger'
    end

    def unbind
        p 'unbind'
        EM.stop
    end
end

EM.run do
    EM.connect '127.0.0.1', 1080, Client
end
{% endhighlight %}

**class.rb**
{% highlight ruby %}
class Clt < EM::Connection
    def initialize(*args)
        p 'class init'
    end

    def post_init
        p 'post init'
    end

    def connection_completed 
        p 'connection complete'
        send_data("hahahahah\n")
        p 'send data'
        close_connection_after_writing
    end

    def receive_data(data)
        p 'won`t trigger'
    end

    def unbind
        p 'unbind'
        EM.stop
    end
end

EM.run do
    EM.connect '127.0.0.1', 1080, Clt
end
{% endhighlight %}

如果执行正确, 开启了 `nc -l 1080` 的命令行会出现 `hahahahah` 的字符串, 表示成功将信息传给了 nc 监听的那个 Socket 中, 而同时在执行这几个 ruby 脚本的命令行中也会出现一些字符串, 代表 EM 作为 Client 去建立 Socket Connection 的时候的状态变化, Proc/Module/Class 出现的内容会有一些不一样, 为什么不一样不是这里的重点所以就不细究了, 但是这里的几个方法还是需要理解一下, 这些方法都可以到 [Rdoc](http://rubydoc.info/github/eventmachine/eventmachine/frames) 中找到:

* `post_init` : Called by the event loop immediately after the network connection has been established. 可以理解为 EM 建立的 Connection 的生命周期的开始.
* `connection_completed` : Called by the event loop when a remote TCP connection attempt completes successfully. 当 Connection 成功建立的时候触发, 如果没有建立成功则触发 unbind.
* `receive_data` : Called by the event loop whenever data has been received by the network connection. 接收 Socket 中传输过来的数据, 但并不是一次就可以接收完毕的, 所以会很容易在此方法中看到类似 `@data += data` 这样的代码, EM 在不断接收到数据后触发此方法, 而此方法不断将接收到的数据存储起来, 直到全部数据传输完成.
* `unbind` : Called by the framework whenever a connection (either a server or client connection) is closed. 当 Socket 之间的连接断开的时候触发.

理解 EM 创建的 Socket 的 Connection 的生命周期很重要, 因为你每一个请求都会经过上面的某几个阶段, 你的代码也会在其中的某几个阶段中进行插入. 在这里, 也了解到了 EM 作为 Client 向一个服务器发起一个 Socket 连接并进行信息传输是如何操作的. 可以尝试修改一下看如何让在 nc 所在的命令行中输入内容, 而 EM 中能够通过 `receive_data(data)` 来获取到数据.


### EM 作为 Server
想让 EM 作为 Server 端来监听某个端口, 其实很简单, 将上面调用的 `EM.connect` 更换为 `EM.start_server` 就可以了, 因为无论 Client 还是 Server 他们都是在操作一个一个的 Socket Connection, 所以 Connection 的 lifecyle 还是类似的, 只是在 Connection 中相关的处理逻辑不一样了.

{% highlight ruby %}
EM.connect '127.0.0.1', 1080, Client
EM.start_server '127.0.0.1', 1080, Client
{% endhighlight %}

抄一个简单的 Echo Server
**echo.rb**
{% highlight ruby %}
require "eventmachine"

module Server
    def post_init
        p 'post init'
        @data = ''
    end

    def connection_completed
        p 'connection_completed'
    end

    def receive_data(data)
        @data += data
        send_data(data)
    end

    def unbind
        p @data
        p 'unbind'
    end
end

EM.run do
    EM.start_server '0.0.0.0', 1080, Server
end
{% endhighlight %}

在一个命令行中运行这个 Server, 然后在另外一个命令行中通过 `nc 127.0.0.1 1080` 就可以交互了, 当 `CTRL + C` 关闭连接中的 nc 时, Server 会将所有接收到的信息在关闭连接触发 `unbind` 后全部传输给客户端.


## EM 中 receive_data 与 Fiber 的思考

### receive_data 与协议解析
通过阅读 EM 中关于 [receive_data 部分的文档](http://rubydoc.info/github/eventmachine/eventmachine/EventMachine/Connection#receive_data-instance_method), 因为 EM 在一个 Event Loop 中, `receive_data` 方法中接收到的数据与其触发一个 Event 被调用的时机是不固定的,

> Depending on the protocol, buffer sizes and OS networking stack configuration, incoming data may or may not be "a complete message"

所以我们需要想办法去处理如何拿到一次请求中我们所需要的所有数据, 举个极端点的例子: 对方发送过来一串 "hello, world!" 字符串信息, 在 Socket Connection 中传输的是这穿字符串的二进制数据 bytes ,而 Socket Connection 是不会知道这些 bytes 哪些部分是开始哪些部分是结束(通过 TCP 协议传输而数据重传不讨论), 所以对于在 `receive_data` 中的处理, 需要我们对这部分内容进行处理, 可以看到使用 EM 的 [thin 也逃不过这点](https://github.com/macournoyer/thin/blob/master/lib/thin/request.rb#L75) (thin 将 HTTP 的详细解析封装到 C 中去了).

想想, 如果我们要来判断不断接收到的 chunk 数据, 什么时候全部到达该如何做? 我们需要对接收到的数据进行解析, 然后根据获取到的数据一步一步的向后解析, 直到整个解析步骤完成. 例如: HTTP 协议. 

{% highlight bash %}
GET /index.html HTTP/1.1\r\n
Accept-Encoding: gzip,deflate,sdch\r\n
Host:www.google.com\r\n
\r\n
{% endhighlight %}

1. 我们需要找到第一个 `\r\n` 结束才能知道这个 HTTP 请求的 verbose 是 GET, 请求的 path 是 /index.html 使用的 HTTP 协议是 1.1 .
2. 接着继续获取数据, 后续每解析盗一个 `\r\n` 就作为 HTTP 的 Header
3. 直到解析到一个以 `\r\n` 开头的并且只有这些数据的空行完成 HTTP GET 的请求.

HTTP 解析还有很多复杂的细节, 但具体的解析会很类似, 都是处于 "接收数据" -> "解析" 这样的循环中, 这个循环可以很大, 通过 "接收数据" 直到所有数据接收完毕再进行 "解析", 也可以很小, 在 "接收数据" 到合适的数据就开始这一部分的 "解析" 然后循环直到所有数据解析完毕. 

### Socks 5 协议
OK, 了解这些后回到要编写的这个应用中, 我们需要通过 Socks 5 协议来构建这个本地的代理服务器. 为什么是 Socks 5 协议呢? 说实话, 因为前面介绍的 Shadowsocks 选择的这个, 呵呵. 不过我想使用这个协议也是因为其

1. 比较底层, 通过 Socks 与代理服务器建立一个连接, 然后其中传输的是 bytes 与上层协议无关
2. 实现简单, 了解 Socks 协议之后, 如果仅支持 Socks 5 而且不需要验证啥的, 两个来回就可以建立连接
3. 应用广泛, 无论是操作系统还是应用都有对其的支持, 例如 QQ 中的 Socks 代理, linux 的 `ss`

为了让本地服务器能够支持 Socks 5 协议, 那么 [rfc1928](http://tools.ietf.org/html/rfc1928) 是必不可少的文档了, 简化一点的 [Socks Wiki](http://en.wikipedia.org/wiki/SOCKS#SOCKS5) 这些文档得多谢[这里](https://github.com/luikore/stochastic-socks/blob/master/local.rb#L3)
再简化一下

1. 客户端连接到 Socks 服务器, 说 hello~ (greeting)
2. 服务器响应是否可以建立 Socks
3. 客户端将需要建立的 Socks 的详细信息提交给服务器, 例如验证啊, Socks 协议信息啊
4. 服务器处理后告诉客户端是否建立成功
5. 接下来这个连接建立好以后, 双方任意传输数据直到连接断开. 断开后重连回到第一步重新开始.

### Fiber 
在想到上面对 HTTP 协议解析的例子与看到 [stochastic-socks](https://github.com/luikore/stochastic-socks) 的例子后, 感觉用 Fiber 来处理这样的情景太合适了, 整个处理过程可以由我们来控制什么时候暂停一下, 什么时候继续, 在接收到合适数据之前我都不断将代码控制权交出去, 直到得到足够我解析请求的数据后进行解析, 多么合适. 

通过查看 Ruby Fiber 的 API 文档, 其拥有的方法数量非常少仅仅只有

1. Fiber.yield : 将 Fiber 将代码执行的控制权释放出去
2. Fiber.alive? : 当前 Fiber 是否还可以继续执行, 是否还活着
3. Fiber.resume : 开始执行 Fiber 或者从上次 yield 的地方重新开始执行 Fiber
4. Fiber.current : 返回当前环境下的 Fiber 实例
5. Fiber.transfer : 切换到当前 fiber 执行

最经常使用的应该就是 Fiber.yield 与 Fiber.resume 了.

前面了解 `receive_data` 方法是在不断的接收数据, 为了让程序达到 "得到必要的数据之后再继续解析" 这么一个目的, 那我们就让一个 Fiber 中有这么一个过程, 让其接收按照某一个条件接收数据, 如果满足这个条件则通过这个阶段, 否则一直让 Fiber.yield 不让其通过这个阶段, 类似的代码为

{% highlight ruby %}
def some_stage(&block)
    if not block.call
        puts 'retry...'
        Fiber.yield
    else
        puts 'passed'
    end
end
{% endhighlight %}

这里将判断是否允许通过的方法提取为一个 block 由外部传递进去, 这个相对于 HTTP 协议会比较合适, 因为其特别复杂 - -|| 对 Socks 协议, 我觉得 stochastic-socks 中的 `wait` 方法会相当合适, 因为在我们当前这个应用中, 需要解析的数据只是那么几个 bytes, 非常简单, 所以方法简化为

{% highlight ruby %}
def wait(n)
    Fiber.yield if not @data.b.size >= n
end
{% endhighlight %}

等待 `@data` 中的数据, 拥有 n bytes 以上则满足, 否则一直 Fiber.yield 让 EM 去接收数据. 

这一篇, 基本上是在叙述偏理论的东西, 这些内容理解后, 下一偏进行实际代码编写的过程中会轻松一些. 现在我们可以再回想一下几个问题:

1. EM 中 Connection 的 life cyle 中有哪些事件会发生, 如何将代码插入到这些入口点中?
2. 在 EM 的 `receive_data` 中, 可以通过什么方法得到收集完整数据?
3. 通过 Socket 接收数据解析协议如何处理?
4. Fiber 在 `receive_data` 中的应用