---
layout: default
title: "Sinatra 探险 (2)"
date: 2013-01-09 23:34
comments: true
categories: 
---
在上一篇的结束, 在一个 Sinatra 项目中, 我用 Bundler 管理了所有 gem, 编写了 config.ru 让支持 Rack 的 Web 应用服务器(Puma)跑了起来, 同时使用 Sinatra::Base 的方式搭建了这个 Sinatra 应用, 接下来要处理其他的东西了.

## Mysql 与连接池
在 Gemfile 文件中, 我已经将熟悉的 `gem 'mysql'` 添加进去了(因为熟悉 mysql 所以没使用 mysql2), 想想, 如果我使用的是 puma, 多线程处理的话是不是得为 mysql 的链接创建一个 pool ? 让 mysql 链接能够重复使用呢? 恩, 然后就在 github 中搜索 "[pool](https://github.com/search?q=pool&p=1&ref=searchbar&type=Repositories&l=Ruby)", 我找到了 [connection_pool](https://github.com/mperham/connection_pool), 看了一下 commit 中有 ![alt mperam](https://secure.gravatar.com/avatar/af54a0871600db7fbdbb5c558a6e29a3?s=40), 因为知道他是 Sidekiq 的作者, 所以相信这个 gem 应该挺可靠的. 粗略看了一下, connection_pool 的使用通用性质的 pool 化的 gem, 他可以很多你想象得到的东西给 pool 化, 例如各种 conn (redis? memcached? mysql conn?), 再次感叹一下 ruby 提供的 magic. 因为这里会使用纯 SQL 查询, 所以就简化了, 将 DB 的初始化放到了 config.ru 文件中.

{% highlight ruby %}
require "bundler"
Bundler.require

require './app'
Dir['./model/**/*.rb'].map { |f| require f }

class DB
  @pool = ConnectionPool.new(size: 3, timeout: 5) do
    conn = Mysql.new('localhost', ENV['username'], ENV['password'], 'dbname') 
    conn.query("SET NAMES UTF8")
    conn
  end

  # 下面的方法从 connection_pool 暴露出来的
  class << self

    # 使用闭包的形式调用
    def with(&block)
      @pool.with &block
    end

    # 直接调用
    def method_missing(name, *args, &block)
      @pool.with do |connection|
        connection.send(name, *args, &block)
      end
    end

  end
end

run App
{% endhighlight %}


这样以后, 我在这个 Sinatra 项目的关联的代码中可以这样使用:

{% highlight ruby %}
DB.query("SELECT now()").fetch_row
DB.with { |conn| … }
{% endhighlight %}


## 返回 JSON
因为此应用定位的是使用 Sinatra 计算并暴露一些 DB 数据, 建立 JSON API 来进行交互, 所以需要返回 JSON. 在刚刚开始的时候, 我相当直接的 `gem 'json'` 然后在需要的地方用个 `object.to_json` 或者 `JSON.generte(obj)` 来返回 JSON 值, 当我看过 [Sinatra::JSON](http://www.sinatrarb.com/contrib/json.html) 我发现我错过了它.. 然后, 我就在 app.rb 中重新认识了它, 同样 Gemfile 也修改了:

**app.rb**
{% highlight ruby %}
class App < Sinatra::Base
  register Sinatra::Contrib

  get '/' do
  	json {foo: "hello"}
  end

  get '/h' do "Hello World!" end
end
{% endhighlight %}


**Gemfile**

{% highlight ruby %}
# A sample Gemfile
source "http://ruby.taobao.org"

gem 'sinatra', '~> 1.3.3'
gem 'sinatra-contrib', '~> 1.3.2'

gem 'puma', '~> 1.6.3'
gem 'connection_pool', '~> 0.9.3'
gem 'mysql', '~> 2.9.0'
gem 'json'
{% endhighlight %}

使用 Sinatra::JSON , 除了会自行转换 json 字符串, 而且还会自动帮我设置正确的 Content-Type 哦.

## Hot reload with rerun
这些弄好以后, 我就开始编写 models/ 目录中的详细业务逻辑了, 也就在这时, 我发现我改动了 model 中的代码刷新页面而没有变化 T_T 我感觉很不习惯, 有点失落的感觉.. 然后 Google 搜索 "[Sinatra hot reload](https://www.google.com/#hl=en&newwindow=1&qscrl=1&sclient=psy-ab&q=Sinatra%20hot%20reload&oq=Sinatra%20hot%20reload&fp=1&bav=on.2,or.r_gc.r_pw.r_cp.r_qf.&cad=b)" 就找到了 [Frequently Asked Questions](l1) 然后就顺藤摸瓜, 知道了一个叫 [rerun](https://github.com/alexch/rerun) 的 gem, 定睛一看介绍, 他就是为使用 Sinatra 而出现的 [囧](https://github.com/alexch/rerun#why-did-you-write-this) .
`gem install rerun` 安装好 rerun , 接下来运行的时候与 Sinatra 的文档中有点不一样, 因为我需要重复执行的命令不是 `rakeup` 而应该是 `puma`, 所以将 `rerun 'rackup'` 改为 `rerun 'puma -p 3000'`(习惯 3000 端口了)

{% highlight bash %}
wyatt sample$ rerun 'puma -p 3000'

00:18:37 [rerun] sample launched
Puma 1.6.3 starting...
* Min threads: 0, max threads: 16
* Environment: development
* Listening on tcp://0.0.0.0:3000
Use Ctrl-C to stop

[Listen warning]:
Missing dependency 'rb-fsevent' (version '~> 0.9.1')!
Please add the following to your Gemfile to satisfy the dependency:
  gem 'rb-fsevent', '~> 0.9.1'

For a better performance, it's recommended that you satisfy the missing dependency.
Listen will be polling changes. Learn more at https://github.com/guard/listen#polling-fallback.

00:18:39 [rerun] Watching ./**/*.{rb,js,css,scss,sass,erb,html,haml,ru} using Polling adapter
{% endhighlight %}


好吧, 看到一个 missing 的警告, 就像他介绍的一样, 用不用自行决定吧, 我就没弄了因为应用文件不多.
我们来修改一下 app.rb 文件, 然后就会看到新增加

{% highlight bash %}
00:20:57 [rerun] Change detected: 1 modified
00:20:57 [rerun] Sending signal TERM to 981
 - Gracefully stopping, waiting for requests to finish

00:20:57 [rerun] Osticketadapter restarted
Puma 1.6.3 starting...
* Min threads: 0, max threads: 16
* Environment: development
* Listening on tcp://0.0.0.0:3000
Use Ctrl-C to stop
{% endhighlight %}

恩, 成功啦~ 

其实在看 [Frequently Asked Questions](l1) 的时候有一个 [Sinatra::Reloader](http://www.sinatrarb.com/contrib/reloader), 为什么没有用他呢? 因为他不会 reload 整个 app 的代码 T_T 也就是说, 我修改 app.rb 可以 reload, 可是我修改 models 中的代码就不行了, 所以还是使用了 rerun.

## 动态添加实例变量
由于我想将查询出来的结果动态的设置进入 model 所以, 就去寻找 ruby 文档看有没有动态设置元素的方法, 然后就寻找到一个 `instance_variable_set` 方法

**ticket.rb**
{% highlight ruby %}
class Ticket
  include JSON

  # 需要在构造函数这样做, 否则 messages/responses 会失效
  def initialize(row)
    # 自动根据查询出来的 attr 设置实例变量, 为了进行 json 序列化
    row.each { |k,v| self.instance_variable_set("@#{k}", v.force_encoding("utf-8")) }
  end 

  class << self
    def id
      tickets = []
      DB.query("SELECT ticket_id, ticketId, email, name, subject FROM ost_ticket
       LIMIT 10, 10").each_hash { |row| tickets << Ticket.new(row) }
      tickets
    end      
  end
end
{% endhighlight %}


然后将修改 app.rb

**app.rb**
{% highlight ruby %}
class App < Sinatra::Base
  register Sinatra::Contrib

  get '/' do
      JSON.generate(Ticket.id)
  end

  get '/h' do "Hello World!" end
end
{% endhighlight %}


这样, 当服务器启动的之后, 访问根目录就能得到结果了.

ps: `instance_variable_set` 部分是很长时候以后在阅读 **Ruby 元编程** 后过来写的, 这个时候发现自己原来的理解是相当相当的简单, 也不太值得详细的去写, 也为了让这篇文章有一个结尾, 还是回来就原来理解的 `instance_variable_set` 部分做了一个总结, 随着对 Ruby 的不断学习, 感觉真的太棒了.


[l1]: http://www.sinatrarb.com/faq.html#reloading