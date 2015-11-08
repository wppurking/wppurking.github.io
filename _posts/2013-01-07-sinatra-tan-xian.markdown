---
layout: default
title: "Sinatra 探险 (1)"
date: 2013-01-07 23:43
comments: true
categories: 
---
## 起因
因为需要写一个简单的 API 程序来暴露一部分 DB 中的数据, 因为这种情况下使用 Rails 太重了, 所以就想到了使用 [Sinatra](l1). 这也是我第一次正式接触它 ^_^

## Bundler with Sinatra
打开 [Sinatra 官网](l1) 收入眼帘的应该是一个足够简单的

{% highlight ruby %}
require 'sinatra'
get '/hi' do "Hello World!" end
{% endhighlight %}


可是说实话, 这可能还是太简单了, 还是需要将一些逻辑代码挪到另外的目录下, 例如一个简单的 models/ 目录. 那这个时候就有一点点问题, 当我有几个 model 的时候拿到一个一个将他们 require 进来? 当然不是, 这回想到的是 [Bundler](http://gembundler.com/), 然后就在 Bundler 的网站上找到了典型的[例子](http://gembundler.com/v1.2/sinatra.html), 新增加了一个 config.ru 文件, 然后在这个文件中 `run MySinatrApp` 当然, 所有其他的依赖也都在这个文件中进行了初始化, 例如添加了一句:

{% highlight ruby %}
# 将 models 与其子目录下的所有 .rb 文件引入
Dir['./models/**/*.rb'].map { |f| require f }
{% endhighlight %}

## config.ru and Rack
看着这个 config.ru 文件, 越发的眼熟, 我在 rails 的项目中也看到过这个东东啊, 我就很奇怪, 为什么这里会需要一个 config.ru 文件呢? 而且与 rails 中的 config.ru 做得事情基本上一致. 然后搜索一番来到了 ["一点点心得: Rack 在 Rails 中的使用"](http://ruby-china.org/topics/4840), 原来这是 Rack 所使用的一个文件. 哈, 找到这里, 突然感觉自己快从原本一个 Sinatra 应用跑题了, 所以我就将 Rack 的研究推后了, 暂时知道他是 *Web 应用服务器(puma, unicorn)* 与 *Web 服务器(nginx, apache)* 之间的一层, 可以理解成接口吧.
此时我的代码已经变成了类似下面的样子

**app.rb**
{% highlight ruby %}
class App < Sinatra::Base

  get '/' do
    Ticket.find
  end
end
{% endhighlight %}

**config.ru**
{% highlight ruby %}
require "bundler"
Bundler.require

require './app'
Dir['./model/**/*.rb'].map { |f| require f }
run App
{% endhighlight %}


**Gemfile**
{% highlight ruby %}
# A sample Gemfile
source "http://ruby.taobao.org"

gem 'sinatra', '~> 1.3.3'
# 要用到的访问数据库嘛
gem 'mysql', '~> 2.9.0'
# 准备用这个作为 Web 应用服务器, 多线程处理
gem 'puma', '~> 1.6.3'
{% endhighlight %}

**models\ticket.rb**
{% highlight ruby %}
class Ticket

  class << self
    def find
      "Bala bala"
    end
  end
end
{% endhighlight %}


这会要写的这个应用的文件目录的基本结构就差不多了, 当然我可以直接在项目目录下运行 `puma -p 3000` 看到可以访问的应用了. 

还有关键字 rerun, json, connection_pool, sinatra-contrib, instance_variable_set 请等下回分晓 :P

##### 可以将 rack 想象成 Java 中接口那样的东西, 而不同的 Web 应用服务器都是 rack 接口的实现, 而 puma 也是其中一个, 所以只要含有 config.ru 并且按照规矩来编写, 就可以通过支持 rack 的 Web 应用服务器启动项目


[l1]: http://www.sinatrarb.com/ "Sinatra"