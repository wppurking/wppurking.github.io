---
title: "利用 EventMachine 和 Fiber 实现 Socks 5 代理(2)"
date: 2013-07-15 22:23
comments: true
categories: 
---
前面已经将一些理论上的内容讲解了, 这一篇则是将自己实现这个本地 Socks 代理的过程给记录下来. 

## 启动这个 EM Server
首先我们需要利用 EventMachine(EM) 启动一个本地的 Server 用来监听进来的连接. 我们可以选择 Proc/Module/Class 来实现, 因为是一个 LocalServer 并且不需要每一次都实例化这个 LocalServer, 所以就没理由的选择使用 Module 来实现了, 虽然 Modlue 在 ruby 也就是一个 Class, 少写几行代码, 哈哈.

**local.rb**
{% highlight ruby %}
require "eventmachine"

module LocalServer
    def post_init
        @data = ''
    end

    def receive_data(data)
    end

    def unbind
        puts 'connection is closing...'
    end
end

stop = -> {
    EM.stop
    puts 'Server Stop..'
}

EM.run do
    # 关于 linux 信号: http://rdc.taobao.com/blog/cs/?p=1540
    trap(:SIGINT, &stop)
    trap(:SIGTERM, &stop)
    trap(:SIGKILL, &stop)

    addr = "0.0.0.0"
    port = 1080
    puts "Server listen at #{addr}:#{port}"
    EM.start_server addr, port, LocalServer
end
{% endhighlight %}

这时, 我们能够通过 `ruby local.rb` 启动这个 Server 了.

## Socks 协议
这里有两个文档需要详细阅读:

1. [rfc1928](http://tools.ietf.org/html/rfc1928)
2. [Socks wiki](http://en.wikipedia.org/wiki/SOCKS)

### 这两个文档中详细记录 Socks5 协议需要处理的内容, 下面我把他简化一下

1. 客户端向服务器发起建立 Socks 代理的请求
2. 服务器响应 Socks 代理请求, 并且告诉客户端是否支持这次代理请求 (greeting)
3. 客户端向服务器发起这次代理请求的详细内容, 例如需要访问的 domain 或者 ip 地址, 如果需要验证用户, 还有验证用的账户密码信息等等
4. 服务器响应客户端发送过来的详细内容, 处理用户验证或者根据客户端的 domain 或 ip 地址将请求进行代理请求.
5. 通过 Socks 服务器建立好请求后, 客户端通过 Socks 通道与目标服务器之间信息来往发送,接收与连接关闭等


### 客户端 Socks 请求与服务端的 gretting
> The client connects to the server, and sends a version identifier/method selection message

对于 Socks 的启动由 Client 触发, 首先由其发送一条确认 Socks 版本/方法等等的 greeting 消息, 从上篇内容中了解, 我们可以通过 Fiber 结合 EM 来实现对于数据的异步获取

{% highlight ruby %}
def receive_data(data)
    p data
    @data << data
    @fiber.resume
end

def wait(n)
    Fiber.yield if not @data.b.size >= n
end
{% endhighlight %}

在 `receive_data` 中接收数据, 然后重新启动 fiber, 同时定义一个 `wait` 方法, 用于判断是否让 Fiber 停留在接收数据阶段, 看到上面的 `@fiber` 你肯定也会猜到, 它应该在 `post_init` 中初始化. 我们如何测试我们正在编写的 LocalServer 处理的二进制数据正确? 我需要一个实际使用 Socks 代理的应用来发送请求. 对于这个可以使用 Chrome 的 Proxy 插件, 也可以使用 Mac OS 的系统代理. 我设置了 Mac OS 中的系统代理为 Socks, 然后用 Safari 浏览器来访问互联网来查看实际情况下的 Socks greeting 信息.

{% highlight bash %}
wyatt em_tunnel$ ruby local2.rb
Server listen at 0.0.0.0:1080
"\x05\x01\x00"
{% endhighlight %}

我们看到总共三个字节, 与 rfc1928 中记录的一样, 第一个字节代表 Version, 第二个字节代表选择的 Methods 会拥有的长度, 第三个字节(最长 255 个字节)表示可选的方法. 根据协议可知道, 这个请求的 greeting 最少应该拥有 3 个 bytes, 所以在 Fiber 中先最少读取 3 个字节的内容再进行处理.

{% highlight ruby %}
def post_init
    @data = ''
    @fiber = Fiber.new do
        greeting
    end
end

def greeting
    wait(3)
end
{% endhighlight %}

将第一次需要读取 3 bytes 的数据放入 greeting 方法中, 是因为用这个方法来处理 Server 与 Socks Client 第一次交互的所有内容.

{% highlight ruby %}
def greeting
    wait(3)
    ver, nmethod = @data.unpack('C2')
    if ver != 5
       send_data("\x05\xFF")
       close_connection
    end
    
    # 中间的数据不要了
    send_data("\x05\x00")
    @data = ''
end
{% endhighlight %}

仅仅让 greeting 对传递进来的消息做了是否为 Socks 5 协议的判断, 如果不是 Socks 5 协议则直接返回不接受并且 `close_connection`, 否则就全部通过返回 VER: \x05 表示 Socks 5 协议与 METHOD: \X00 表示 "NO AUTHENTICATION REQUIRED" 两个字节并且等着客户端发送新的信息过来.

现在, 我借用 Mac OS 的系统 Socks 代理与 Safir 看到了 Socks 的第一步 "客户端发起建立 Socks 代理的请求" 与利用 EM 编写的 "服务器响应 Socks 代理请求, 并且告诉客户端是否支持这次代理请求" 这两个步骤, 接下来还有三个步骤将会在后续的文章中详解.


## 其他
当我将这个小例子项目写完以后, 也让我对利用 Socket 编程和与底层的协议相关的编程内容有了新的理解. 让我理解到 Socket 通信在现在经常编写的 Web 应用所使用的 HTTP 协议外还有很多其他的协议, 而他们之间的通信都是建立在 Socket 通信这个基础之上的, 这也让我理解了为什么说 beanstalkd 的协议很简单, 为什么 redis 通信很简单, 让我直接体会到了使用 `nc -C 0.0.0.0 11300` 连接 beanstalkd 直接和他通信是在发生些什么. 