---
layout: post
title: 使用 Monit 来监控进程安全
date: 2015-09-29 23:28:30
tags: [tech linux monit]
---
## 起因
最近在使用 localtunnel.me 这个工具了建设组内简单的 Gitlab hook 加自动部署, 需要将内网的服务暴露给外网. 最终确定下来可用的通过 tunnel 来暴露外网的工具就 localtunnel 比较靠谱(ngrok 被墙). 但在使用的过程中遇到的问题是, 连接上服务后总会在几个小时后自动的 throw err 断开, 这在网内的自动部署服务上是不可接受的呀, 不能让人隔一会去重启一次吧.

## 确定 Monit
在最终确定使用 Monit 之前, 其实有过好几个其他的选择. Ruby 的 [god](https://github.com/mojombo/god) (有人说内存泄露, 这个问题现在已经没有了, 也很稳定了), 也有 Python 的 [supervisor](https://github.com/Supervisor/supervisor) , 还有确定下来的 C 的 [Monit](https://bitbucket.org/tildeslash/monit). 最终选择 Monit 是因为以下几个点:

1. 非常小巧, 但核心功能完备
1. 除了对进程级别控制, 还能够对系统级别资源, 进程级别资源进行控制做反应
1. 资源占用小, 配置好启动后就可不用再理睬
1. 外部扩展也足够, 可扩展执行脚本, 也可直接接入 Email 通知等等.
1. 软件能够稳定的持久, 可发展的维护.

最终选择的是 Monit, 他从一个小的开源工具做起, 到现在已经有一个商业的 M/Monit 产品在支持着, 这足够, 而且他的理念与我想的非常符合 "开启后, 就让你感觉不到他的存在, 一个放心的后背"


## Monit 的基本使用
其实如果去解释 Monit 的基本使用, 基本就是在翻译文档. 我将阅读文档后的内容整理了一番, 并在重点的地方给予了一些例子, 这样能够用于整理这里的思路. 这些内容整理到 Xmind 中, 可见下面的图. 同时将重要的五个重点理解抽取出来:

* 配置文件基本概念:  配置文件中的基本写法是怎样的
* 基本参数 : monit 的 日志/周期/ssl/http 配置
* Email 通知参数:  需要 alert 外部的时候的相关配置. (这部分也挺多内容的)
* 服务公用参数 :  这是每一个 check block 中都可以通用的内容
* Service Test : 这部分是最多最复杂的, 也是最需要查询文档的.

<iframe id="xmindshare_embedviewer" src="http://www.xmind.net/embed/Cdmq?size=small" width="750px" height="450px" frameborder="0" scrolling="no"></iframe>

