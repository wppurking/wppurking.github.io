---
layout: default
title: "Nginx 反向代理自定义域名访问 Cloudfoundry"
date: 2013-03-13 13:18
comments: true
categories: 
---
## 背景
在尝试了一些 PaaS 的产品后, 对 CloudFoundry 所提供的服务有了很大的兴趣, 但苦于其现在还没有正是的商业化, 仍然处于 beta 版本, 也出处于公开测试的状态. 
虽然现在肯定无法重要的系统部署到这个上面, 不过对一些小公司部署一些简单的页面, 例如简单的公司介绍, 简单的公开页面使用他再合适不过了, 而使用这些服务, 肯
定是想帮顶一个自己的域名, 而在公开测试中的 CloudFoundry 却将自定义域名的功能给关闭了, 其实也就是想推广 CloudFoundry. 反而去看 appfog 建立在 CloudFoundry
代码上的已经商业化的服务看, 他的确提供域名帮顶服务, 不过可能也正是因为其提供的域名帮顶服务加上 2G RAM 的免费使用空间, 结果导致通过 CLI 进行项目部署的时候
响应速度非常非常的慢, 对比一下 CloudFoundry 的速度, 直接让我选择 CloudFoundry 了.


## 问题
知道 CloudFoundry 不会提供集成的自定义域名的功能, 可是我拥有自己的服务器, 我通过 Nginx 的反向代理, 再配置一下 Godaddy 的域名应该是可以达到自定义域名
访问的效果的. 可能会感觉, 既然有自己的服务器为什么还要用 CloudFoundry 呢? 答案是因为简单啊, 在自己的服务器上部署要一系列动作, 而在 CloudFoundry + Nginx 
反向代理的话我只需要操作一下, 以后就直接向 CloudFoundry push 代码就好了. 可是在配置完成以后发现, 似乎 CloudFoundry 考虑到了这种情况, 所以不让用 Nginx 的
反响代理, 下面是刚开始的配置文件:

**cf.conf**
{% highlight bash %}
upstream cf {
  server xxxx.cloudfoundry.com;
}

server {
  listen 80;
  server_name customer.com;

  location / {
    proxy_pass http://cf;
    proxy_set_header Host xxxx.cloudfoundry.com;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}
{% endhighlight %}


## 探索
我就不解了, 查看 CloudFoundry 的路由其实也是使用到了 Nginx 去转发啊? 并且按照 Http/1.1 协议来理解的话, 在通过 Nginx 的反向代理的时候, 向对应的 IP 提交
了 Host 响应头啊? 当然同时还携带了另外两个 Header 头信息为了让 Rails 能够判断反向代理的是客户端 IP 是什么. 哪里出了问题? 我不死心, 然后我在 CLI 中用纯命令
行测试了一下:

{% highlight bash %}
wget customer.com
# xxxx.cloundfoundry.com 的 ip
wget 173.243.49.35
{% endhighlight %}

最后返回的结果是一样的 404 错误, 这个我大概能够猜到是 Http Header 中没有 Host 信息的原因, 然后我尝试另外增加上 Host 头:

{% highlight bash %}
$> wget --header='Host: xxxx.cloudfoundry.com' 173.243.49.35
--2013-03-13 13:51:42--  http://173.243.49.35/
Connecting to 173.243.49.35:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 6516 (6.4K) [text/html]
Saving to: ‘index.html’
{% endhighlight %}

哈哈, 这回正确了, 看来的确是 Host 没有提交. 等等, 我在 Nginx 的反向代理中也提交了 Host Header 头信息啊? 为什么哪? 

{% highlight bash %}
$> wget --header='Host: xxxx.cloudfoundry.com' --header='X-Forwarded-For=220.202.111.7, 220.202.29.7' --header='X-Real-IP: 220.202.111.7' 173.243.49.35
{% endhighlight %}

奇怪了? 这样也能够正常通过. 这下有好多问题了, 在 Nginx 反向代理的时候提交给 CloudFoundry 的具体是什么内容? [X-Forwarded-For, $remote_addr, $proxy_add_x_forwarded_for](http://lavafree.iteye.com/blog/1559183)
查看了这篇文章, 还有 [X-Forwarded-For Wiki](https://en.wikipedia.org/wiki/X-Forwarded-For), [X-Forwarded-For 稍微详细点的解释](http://rod.vagg.org/2011/07/wrangling-the-x-forwarded-for-header/)
不过最后通过 Nginx 能够达到我的效果却是下面这样:

**cf.conf**
{% highlight bash %}
upstream cf {
  server xxxx.cloudfoundry.com;
}

server {
  listen 80;
  server_name customer.com;

  location / {
    proxy_pass http://cf;
    proxy_set_header Host xxxx.cloudfoundry.com;
  }
}
{% endhighlight %}


## 待续
通过 Nginx 反向代理 CloudFoundry 的问题虽然解决了, 可是还有个问题没解决, 为什么在 Nginx 中我添加了 `X-Forwarded-For` 参数就不起作用了呢? 想了想准备做一下下面的实验:

1. Nginx 配置只添加 Host header
2. Nginx 配置中添加 Host, X-Forwarded-For + $proxy_add_x_forwarded_for
3. Nginx 配置中添加 Host, X-Forwarded-For + 固定 IP
4. Nginx 配置中添加 Host, X-Real-IP + $remote_addr
5. Nginx 配置中添加 Host, X-Real-IP + 固定 IP
6. Nginx 配置中添加 Host, X-Real-IP, X-Forwarded-For, $proxy_add_x_forwarded_for, $remote_addr;
7. Nginx 配置中添加 Host, X-Real-IP, X-Forwarded-For, 两个固定 IP

这个实验等我抽出点时间再来弄好了.


