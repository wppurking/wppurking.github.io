---
layout: default
title: "wrap_parameters 初识"
date: 2013-09-22 23:09
comments: true
tags: [rails angularjs payload]
---
## 问题
最近在使用 angularjs 与 rails 练习一个 Todolist 的例子程序, 在利用 angularjs 的 resources 模块创建一个前端的 Resource Model 向 Rails 提交创建请求的时候, 发生了个奇怪的问题.

"前端代码 POST 一个 {title: 'aaa'} 的数据给后端 TodoController, 竟然拿到 {title: 'aaa', todo: {title: 'aaa'}} 这样的 params", 我惊讶了, 我说, 这到底是哪里进行了处理? 是 http 协议处理的还是 rails 自己添加了一个 `todo` 的 params ?


## 寻找
看到这个问题我满脑子的疑问, 这个 `todo: {title: 'aaa'}` 参数到底是哪里来的? 我先查看了我向后端提交的参数, 通过 Chrome 查看如下

{% highlight bash %}
# HTTP header
POST /todoitems HTTP/1.1
Host: localhost:3000
Connection: keep-alive
Content-Length: 35
Accept: application/json, text/plain, */*
Origin: http://localhost:3000
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_7_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/29.0.1547.65 Safari/537.36
Content-Type: application/json;charset=UTF-8
DNT: 1
Referer: http://localhost:3000/
Accept-Encoding: gzip,deflate,sdch
Accept-Language: en-US,en;q=0.8

# Request Payload 
{"title":"123","is_complete":false}

# Response Header
HTTP/1.1 200 OK
X-Frame-Options: SAMEORIGIN
X-XSS-Protection: 1; mode=block
X-Content-Type-Options: nosniff
X-UA-Compatible: chrome=1
Content-Type: application/json; charset=utf-8
ETag: "419adbd54153d295384cda2ea472df60"
Cache-Control: max-age=0, private, must-revalidate
X-Request-Id: 24ab1689-4d8e-40fc-b5f3-b8cd792166c1
X-Runtime: 503.361373
Connection: close
Server: thin 1.5.1 codename Straight Razor
{% endhighlight %}


我发现了一个奇怪的地方, 原本 Post 用来传输数据的 `Request Body` 被替换为了 `Request Payload`, 这个 Payload 是什么东西啊? 通过 Google 寻找了半天, 无论是 [wiki](http://en.wikipedia.org/wiki/Payload_(computing\)) 还是 [stackoverflow](http://stackoverflow.com/questions/5905916/payloads-of-http-request-methods) 都感觉不解决疑惑, 直到找到[这个 RFC 草案](http://tools.ietf.org/html/draft-ietf-httpbis-p3-payload-19#section-3) 才有点解惑.

### Payload
我暂时还无法给 Payload 是什么下定义, 不过我把它理解为传递的一部分信息, 而这类信息在 HTTP 协议中被推荐包含在 request header 或者 request body 中(如果有的话).

了解了什么是 Payload 可是还没有解决为什么在 Rails 后端中突然多出来一个 `{todo: {title: 'aaa'}}` 这样的参数. 当我排除浏览器, 直接通过命令行输入 

{% highlight bash %}
curl -X POST -H "Content-Type: application/json;charset=UTF-8" -H "X-Requested-With: XMLHttpRequest" localhost:3000/todoitems -d '{"title":"sdf","is_complete":false}'
{% endhighlight %}

并且在后端得到 
{% highlight javascript %}
"{title: 'sdf', is_complete: false, todoitem: {title: 'sdf', is_complete: false}}"
{% endhighlight %}

这样的结果时, 我确定! 这一定是 Rails 在某个地方对 params 进行了处理, 并且判断的标准一定与 `Content-Type:application/json` 有关.

### 运气呀
可我不知道去翻看那一部分 Rails 代码, 于是乎上 [github 搜索一下](https://github.com/rails/rails/search?p=1&q=params&ref=cmdform), 可是通过 `params` 这个关键字在 Rails 项目中搜索出来的内容太多了, 而且大多数与我的问题一点关系也没有. 然后我冥冥中打开 RubyMine , 然后通过 `class name search` 功能进行类名搜索, 也是 params 关键字不过这回找到了两个类 `ActionDispatch::ParamsParser` 和 `ActionController::ParamsWrapper` 而当我看完这两份代码的时候发现, 原来就是这里进行了处理!  感叹, 找到这里真的是有蛮巧合的.

这一切的秘密就在 [params_wrapper.rb](https://github.com/rails/rails/blob/master/actionpack/lib/action_controller/metal/params_wrapper.rb) 和 [initializers/wrap_parameters.rb](https://github.com/rails/rails/blob/2214237c3950445208635a332d520d6aa530c1de/guides/code/getting_started/config/initializers/wrap_parameters.rb) 这两个文件.


{% highlight ruby %}
# Performs parameters wrapping upon the request. Will be called automatically
# by the metal call stack.
def process_action(*args)
  if _wrapper_enabled?
    # 这里从 request.request_parameters 结合 wrap_parameters api 设置的参数, 
    # 将传递上来的 hash params 包装成为一个独立的 hash -> wrapped_hash
    wrapped_hash = _wrap_parameters request.request_parameters
    # 对其他的基础 params 进行处理, 过滤掉 wrapped_keys 之外的 params
    wrapped_keys = request.request_parameters.keys
    wrapped_filtered_hash = _wrap_parameters request.filtered_parameters.slice(*wrapped_keys)

    # 最后将这些 params 合并到 request 的 params 中去.
    # This will make the wrapped hash accessible from controller and view
    request.parameters.merge! wrapped_hash
    request.request_parameters.merge! wrapped_hash

    # This will make the wrapped hash displayed in the log file
    request.filtered_parameters.merge! wrapped_filtered_hash
  end
  super
end
{% endhighlight %}

代码跟踪到这里问题我的疑问已经解决了, 在 [405-angularjs](https://github.com/railscasts/405-angularjs) 项目中通过 $resource 封装 angularjs 的 rest 资源, 然后通过 `resource.save({title:'xx'})` 或者 `new resource({title:'xx'}).$save()` 所触发的请求, 能够在 rails 后端直接 `resource.new(params['model_name'])` 的原因也找到了.