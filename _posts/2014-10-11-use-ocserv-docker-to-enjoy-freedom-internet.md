---
layout: post
title: "使用 ocserv-docker 享受自由网络"
date: 2014-10-11 11:42:44
description: "ocserv docker anyconnect 搭建 部署 配置 教程 翻墙 证书 iphone android"
category: 
tags: [ocserv docker anyconnect]
---
# 起因和问题
在家的时候, 为了能够看到 twitter, youtube 等国外网站, 我会购买一个商业的 VPN 或者科学上网的服务来解决自己的问题, 虽然有一点点不适应我自己的所有需求, 但总该能解决我绝大部分问题. 在公司的时候, 为公司建立了通过 SSL 通道来进行科学上网, 业务部门都只需要网页加特定翻墙的问题也可以完美解决. 让我折腾出 ocserv-docker 的原因是因为:

1. 我不想又经历一次 "十八大" 让我原来的 hosts 方案失效后的无奈
1. 给公司再提供一个备用的科学上网的方案
1. 顺便将我现在手机端科学上网的体验更加流畅
1. 能够让我自己与我老婆以及朋友等等各种各样的手持设备上能方便科学上网
1. 能够让拥有自己服务器的我那些朋友们, 能极度方便搭建自己的科学上网

所以在寻找, 探索, 尝试, 学习过后, 有了现在这样一个使用 [docker][docker-site] 来部署 [ocserv][ocserv-site] 的科学上网方案.

#### 他带来的特点是

1. [ocserv][ocserv-site] 相对 OpenVPN 来说其拥有较强稳定, 不会轻易被封. 因为其使用 Cisco AnyConnect 的协议, 有很多大型商业公司用着.
1. Android, iOS, Mac OS, Windows 都有 Cisco 所提供的 AnyConnect 的客户端, 天然全平台支持啊, **并且 iOS 无需越狱, Android 无需 Root**.
1. 完美的自动重链, 手机上开启后就可不管了
1. 可服务器分发路由, 客户端啥都不用做就可选择性翻墙
1. [docker][docker-site] 给部署 [ocserv][ocserv-site] 带来了 "Step1: 安装 docker; Step2: docker run; Step3: 客户端链接使用" 的极简单部署.


# About [ocserv-docker][github]
寻找到 ocserv 这个项目其实挺偶然的, 最初是从 twitter 上看到 AnyConnect 这个词, 然后顺藤摸瓜到 [ocserv][ocserv-site] 这个开源实现. 第一次部署一个 ocserv 给自己使用的时候那是相当的痛苦, 从下载源代码编译, 到生成自用的证书, 到生成账户密码, 到寻找如何自动重连, 一个一个的坑摆在面前. 当我把这些弄完之后, 发现整个安装步骤如果使用 docker 来组织一个 image 然后直接从 Docker Hub 上下载部署该是多么方便. 也正是因为那段时间刚刚将公司的抓取服务器上的项目全部从全脚本安装迁移为 docker + sshkit 管理, 对 docker 有了一点熟悉才决定这么干.

因为 ocserv-docker 是使用 docker 进行部署的, 所以安装步骤就如 [ocserv-docker 在 GitHub][github] 中 README 所描述的那样, 保证联网 Copy-Paste 三行代码执行就完成了, 下面说对一些特殊点的设计的想法介绍一下.

## 证书, 密钥
第一次折腾 ocserv 所需要的证书, 密钥的时候感觉好麻烦, 好繁琐, 但是当你自己从源代码编译部署过 5/6 次后, 其实也就基本上理解了. 在 [ocserv-docker][dochub] 中, 为了让第一次使用他的人不用被这繁杂的东西所烦恼所以只留下一个入口, 那就是 `./certs/` 中的两个 tmp 文件, 修改好你想要修改的基础信息(或者保持不变), 最终会使用这两个模板文件, 在 image 中的 `/opt/certs` 目录中存放生成好的 certs 文件, 一个 `ca-key.pem` 一个 `server-key.pem`. 单独将 certs 文件放出来的原因是, 可以在运行 images 通过 volumn 重新挂载这个位置将自己的新的 certs 文件应用上去, 如果没有自己的 certs 文件, 那么就无需关心了.

## 用户名, 密码
其实我也知道将密码存在代码中不好, 也可以将其优化成为每一次 build 都自动生成一个新密码, 或者完全将密码文件取消. 选择将初始化两个账户, 并且使用简单密码的原因在于两个:

1. 对新用户而言, 我不想他们在第一次使用 ocserv 的时候, 还要先去了解下 `ocpasswd` 命令怎么用, 先给他们一个默认账户密码可以直接使用为好.
1. 对于我自己的需求而言, 我会部署好多个, hk/jp/us/sg 都会有服务器, 我不想每个服务器都去设置一下用户密码.

所以对账号密码有特别安全要求的, 那么自己了解我一下 `ocpasswd` 命令, 然后编辑一下 `./ocserv/ocpasswd` 即可完成自己的定制.

## 自动分发的路由
这一块是我现在的短板, 我是直接借用了 [kevinzhow/route.sh](https://gist.github.com/kevinzhow/9661732), 因为自己还不知道如何去寻找这样的国外的顶级 IP, 所以我偷了点懒, 直接使用了他人弄好的, 只是会在自己的使用过程中碰到新的 IP 进行一点补充.

因为 ocserv 对自动分发的路由数量有限制(具体数量忘了, 大概是 30~50 条之间), 所以想要完美的解决国内国外翻墙是比较困难的, 但囊括大多数的国外的子网掩码为 255.0.0.0 的 IP 应该还是可以解决很多问题的, 这也算这个方案的一点点缺陷, 好在这个影响现在对我不明显, 基本上没有~

# PS
因为最近将自己的 Blog 从 octopress 更换为 Jekyll, 所以原本集成的 Disqus 插件没了, 需要花点时间处理一下. 在这之前, 有啥问题就直接 twitter: @wyatt_pan 吧

请[安装 Docker 1.0+](https://gist.github.com/wppurking/55db8651a88425e0f977) 并且操作系统所使用的 Linux Kernal 在 3.8+  如果无法选择, 那么请使用 Ubuntu 14.04 LTS 的 OS

对于什么是 ocserv 或者什么是 AnyConnect 这样的问题就交给 [度娘](http://www.baidu.com/s?ie=utf-8&f=8&rsv_bp=1&tn=baiduhome_pg&wd=anyconnect%20ocserv&rsv_spt=1&rsv_enter=1&rsv_sug3=9&rsv_sug2=0&inputT=1331) 和 [谷哥](https://www.google.com/#newwindow=1&hl=en&qscrl=1&q=what+is+anyconnect) 吧 :P


[docker-site]: https://docker.com/ "Docker Inc."
[ocserv-site]: http://www.infradead.org/ocserv/ "OpenConnect VPN Server"
[github]: https://github.com/wppurking/ocserv-docker "GitHub ocserv-docker"
[dochub]: https://registry.hub.docker.com/u/wppurking/ocserv/ "Docker ocserv"
