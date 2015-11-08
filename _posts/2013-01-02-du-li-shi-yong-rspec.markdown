---
layout: default
title: "独立使用 Rspec"
date: 2013-01-02 22:02
comments: true
categories: 
---
第一次使用 Rspec 是跟着 [Ruby On Rails Tutorial](http://ruby.railstutorial.org/) 书中的例子做的. 而在今天之前的时间里, 只要我使用 Rspec 就都离不开 Rails, 一条 `rails generate rspec:install` 自动帮我生成好 Rspec 配置文件, 然后我就可以开始写测试代码了. 现在, 我有一个非 rails 项目也想使用 Rspec 编写测试代码, 这时我发现我连 Rspec 最基础的使用方法还不会呢…

## 安装
Rspec 分成了三个部分

* 为让整个 Rsepc 能够跑起来的核心 [rspec-core](l1)
* 为 Rspec 提供了 DBB/TDD 各种 DSL 的 Matcher 等等 [rspec-expectations](http://github.com/rspec/rspec-expectations)
* 为了完成 Mock 测试提供支持的 [rspec-mocks](http://github.com/rspec/rspec-mocks)

对于使用来说, 基本上都会使用到, 所以有一个 rspec 的 meta-gem 依赖这三个模块, 这也是 Rspec 项目的模块划分形式.

`gem install rspec`



## 配置

### Rspec 配置
在这里, 我配置了两个文件:

* 一个为 [rspec-core](l1) README 中介绍的项目根目录下的 .rspec 文件. 这个文件用来记录 Rspec 提供的用来执行测试的 `rspec` 命令的配置文件(如果用过 [spork](https://github.com/sporkrb/spork) 是需要设置 -drb 的 ).

{% highlight bash %}
$ cat .rspec
# 每一条配置一行. 我都用的配置简写, 具体可查询 rspec -h
# 设置了测试结果的输出格式, [d]ocument 当然是最好阅读的.
-fd
# 这是是执行进度, 就是会看到一个一个的点点. 
# [文档以及例子](https://www.relishapp.com/rspec/rspec-core/v/2-4/docs/command-line/format-option)
-fp
# 设置需要颜色输出
-c
# 设置需要分布式执行测试, 结合 spork 使用
# 如果没有设置 spork 则不可以使用这个参数
#-X
{% endhighlight %}

* 一个为初始化配置 Rspec 的 spec/spec_helper.rb 的配置文件(例如 spork 的配置会配置在这里面).
这个文件文档我没有找到, 在 [rspec-core](l1) 的源代码中也只找到一些片段, 所以就学着 rails-rspec 中生成的 spec_helper.rb 来参考, 创建 ./spec/spec_helper.rb 文件, 并且在某一个 *_sepc.rb 测试文件的第一行 `require spec_helper`. 在利用 `rspec` 命令执行测试代码的时候, 其实已经将当前项目的根路径(根据./spec 判断的)添加到了 Ruby 的 gem 的搜索路径中去了(可添加 `puts $:` 查看 ruby 的所有 gem 加载路径).



### Spork (Option)
这个是用来对 Rspec 加载的文件进行缓存加速测试启动用的, 一般小项目不需要, 可做选择使用.在利用 RubyGem 安装了 spork 后, 配置 spork 也就一条命令(详细参数的个性配置请参考文档):

`spork rspec --bootstrap`

然后你可以在 ./spec/ 目录下看到 spec_helper.rb 文件的上面部分多了

{% highlight ruby %}
require 'rubygems'
require 'spork'
#uncomment the following line to use spork with the debugger
#require 'spork/ext/ruby-debug'

Spork.prefork do
  # Loading more in this block will cause your tests to run faster. However,
  # if you change any configuration or code from libraries loaded here, you'll
  # need to restart spork for it take effect.
end

Spork.each_run do
  # This code will be run each time you run your specs.
end
{% endhighlight %}

这些就是 spork 的配置. 使用 spork 的缺点是, 如果启动 spork server 之后, 你修改了已经被 spork server 加载缓存的 *.rb 文件, 那么则需要对 spork server 重启, 在 Rails 中使用典型的就是修改了 models 需要重启 spork server.


## 测试代码依赖
此时, 已经可以通过 rspec 命令执行测试了, 但是当你多写了几个测试以后会发现问题. "想测试的 Class 无法 require". 这个时候就是 spec_helper.rb 文件中配置的时候了, 例如使用下面的例子:

{% highlight ruby %}
require 'rspec'
Dir['./models/*.rb'].map { |f| require f }
{% endhighlight %}

这里将 Dir 的路径设置为 ./ 开始的相对路径, 是因为在执行 rspec 命令的时候, 基本只会从项目的根路径下执行 `rspec spec/xxx.rb` 来进行测试, 如果非项目根路径, 则会报告路径错误.



## 总结
作为独立使用 Rspec 其实非常的简单, 如果是测试简单的代码, 其实没有配置文件也可以成功, Rspec 都会使用默认值. 在这里花费时间最多的应该是如何将需要测试的依赖 require 到 Rspec 测试环境中来, 作为一个 Rails 新手, 甚至 Ruby 新手, 我想在这里花上一些时间应该是很必要的. 使用 Rspec 编写测试代码是非常舒畅的一件事情.

最后再理一遍顺序:

1. 使用 rubygem 安装 rspec, spork
2. 在项目根目录创建 rspec 命令的配置文件 ./spec/.rspec 
3. 创建 spec_helper.rb 文件, 可选设置好 spork
4. 将依赖的文件添加进入 spec_helper.rb 文件中
5. `rspec spec/xxx.rb` 测试~



[l1]:http://github.com/rspec/rspec-core