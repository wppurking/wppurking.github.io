---
layout: post
title: "为 Mac OS 添加 Bash Completion"
date: 2013-03-03 15:37
comments: true
categories: 
---
其实我对 Shell 一般都没有什么特别大的需求, 无论是 Linux 还是 Mac OS 上用着默认的 Bash 就好了, 最多在 Mac OS 上使用 iTerm2 来进行窗口的 Split, 因为 bash 已经足够好用, 而且也没有影响我啥. 而促使我去寻找 Bash Completion 的是因为在探索 [git flow](https://github.com/nvie/gitflow) 发现了 [git-flow-completion](https://github.com/bobthecow/git-flow-completion), 然后..然后发现在 Mac OS 上竟然没有 git [tab] 补全而在 Linux 上则有! 所以就上网搜索了一番.

## Bash Completion

### 寻找
找到了这篇文章 [An introduction to bash completion: part1](http://www.debian-administration.org/articles/316). 看了文章以后大概了解到 bash 中的补全是利用 `complete` 来完成的, 并且相关的指令的 completion 文件会放在 `/etc/bash_completion.d` 目录下. 然后我去看 Mac OS 的 `/etc` 目录, 可是没有找到的 `bash_completion.d` 目录, 因为系统使用的 [Homebrew](http://mxcl.github.com/homebrew/) 安装软件, 所以下意识的用 `brew search comple` 搜索了一下

{% highlight bash %}
wyatt ~$ brew search comple
bash-completion	  zsh-completions
^C
wyatt ~$ 
{% endhighlight %}

哈哈, 找到两个 completion, 然后上网搜索了一下 [bash-completion](bc) 然后我就….

### 安装
 `brew install bash-completion` 了, 这个太有用了, 就是将很多常用的命令的 bash completion 带到了 Mac OS (其实应该说是维护了一个 Linux 上通用的 bash completion, 可以到 Ubuntu 的 /etc/bash_completion.d 目录中看一下, 很有一样的代码). 通过 `brew install` 后还需要手动添加一个初始化语句才会生效

{% highlight bash %}
wyatt ~$ brew info bash-completion 
bash-completion: stable 1.3
http://bash-completion.alioth.debian.org/
/usr/local/Cellar/bash-completion/1.3 (190 files, 1.1M) *
https://github.com/mxcl/homebrew/commits/master/Library/Formula/bash-completion.rb
==> Caveats
############# 就是这里 ###########
Add the following lines to your ~/.bash_profile:
  if [ -f $(brew --prefix)/etc/bash_completion ]; then
    . $(brew --prefix)/etc/bash_completion
  fi

Homebrew's own bash completion script has been installed to
  /usr/local/etc/bash_completion.d

Bash completion has been installed to:
  /usr/local/etc/bash_completion.d
wyatt ~$ 
{% endhighlight %}

`brew --prefix` 是找到 brew 所安装的目录, 在我这里为 `/usr/local`, 将 `/usr/local/Cellar/bash-completion/1.3/etc/bash_completion` 初始化脚本添加到 `~/.bashrc` 然后重启(或者 source)后我们输入 `dd [tab][tab]`

{% highlight bash %}
wyatt ~$ dd 
--help     bs=        conv=      ibs=       obs=       seek=      
--version  cbs=       count=     if=        of=        skip=      
{% endhighlight %}

恩, 这下 Mac OS 下拥有很多 bash completion 了 :)

### bash-completion 2.0?
细心的看 `brew info bash-completion` 会发现 brew 所指向的版本是 1.3 , 而最新的 [bash-completion](bc) 是 2.0 版本, 我就很奇怪为什么 brew 不升级成为最新版本呢? 所以带着疑问[去 Github](https://github.com/mxcl/homebrew/issues/search?q=bash-completion) 找到了[这个 issue](https://github.com/mxcl/homebrew/issues/13212) 其实是 Mac OS 现在的 bash 版本还是 3.x

{% highlight bash %}
wyatt ~$ bash --version
GNU bash, version 3.2.48(1)-release (x86_64-apple-darwin11)
Copyright (C) 2007 Free Software Foundation, Inc.
wyatt ~$ 
{% endhighlight %}

而 [bash-completion](bc) 2.0 则会需要 bash 4+ , 我上 Ubuntu 12.04 看了下, bash 的版本已经更新到了 4.2.24

{% highlight bash %}
wyatt:/proc# cat /etc/issue
Ubuntu 12.04 LTS \n \l

wyatt:/proc# bash --version
GNU bash, version 4.2.24(1)-release (x86_64-pc-linux-gnu)
Copyright (C) 2011 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>

This is free software; you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
wyatt:/proc# 
{% endhighlight %}

好吧, 那我还是乖乖的用老版本好了.

## 缺失的常用命令补全
此时其实我已经能够对很多常用的命令使用 bash-completion 了, 比如 `ssh root@e.e[tab][tab]` 就会自动补全我想要 login 的网站, 比如 `brew info bas[tab][tab]` 能提示我开头为 bas 的几个应用. 但还是会有几个工具缺失 bash completion. 例如: git, git-flow, rake, rvm

### 利用 brew 安装的 bash-completion 安装
查看 `brew info bash-completion` 知晓所有的 bash_completion 脚本都被安装到了 `/usr/local/etc/bash_completion.d` 目录中, 那我其实也就只要找到对应命令的 bash completion 脚本然后添加进入就好. 的确, 总结起来就四步:

1. 寻找对应指令的 bash completion 脚本
2. 放入 `/usr/local/Cellar/bash-completion/1.3/etc/bash_completion.d` 目录
3. `brew unlink bash-completion && brew link bash-completion` 安装 bash completion 脚本生效
4. 重启新 bash

以 `git` 为例:

1. 我们可以在 github 上找到 [git-completion.bash](https://github.com/git/git/blob/master/contrib/completion/git-completion.bash)
2. 我们把它下载下来 `cd /usr/local/Cellar/bash-completion/1.3/etc/bash_completion.d;wget https://raw.github.com/git/git/master/contrib/completion/git-completion.bash`
3. `brew unlink bash[tab][tab] && brew link bash[tab][tab]`
4. 重启新 bash

然后我们输入 `git [tab][tab]` 就可以看到

{% highlight bash %}
wyatt ~$ git 
add                 clean               get-tar-commit-id   notes               send-email 
am                  clone               grep                pull                shortlog 
annotate            commit              gui                 push                show 
apply               config              help                rebase              show-branch 
archive             describe            imap-send           reflog              stage 
bisect              diff                init                relink              stash 
blame               difftool            instaweb            remote              status 
branch              fetch               lg                  repack              submodule 
bundle              filter-branch       log                 replace             svn 
checkout            flow                merge               request-pull        tag 
cherry              format-patch        mergetool           reset               whatchanged 
cherry-pick         fsck                mv                  revert              
citool              gc                  name-rev            rm                  
{% endhighlight %}

是不是很酷? 然后下面是我常用的命令的 bash 文件:

* [git](https://github.com/git/git/blob/master/contrib/completion/git-completion.bash)
* [git-flow](https://github.com/bobthecow/git-flow-completion)
* [rake 第一版](http://turadg.aleahmad.net/2011/02/bash-completion-for-rake-tasks/) [修改版](https://gist.github.com/turadg/840663)
* rvm 直接看[官网](https://rvm.io/workflow/completion/)的就好了 :)
* [rails](https://github.com/jweslley/rails_completion/blob/master/rails.bash)

在为 rake 添加 bash-completion 之后, 感觉超级的好. 在 rails 项目下, 脚本会自动寻找到 tmp/cache 目录, 然后将一个 rake 指令的缓存文件放在这, 避免每次 rake bash completion 太长时间. 而在类似 octopress 没有 tmp/cache 的, 会直接创建在项目根路径下, 这是可能需要在 .gitignore 中添加下 `.rake_t_cache`

{% highlight bash %}
# rake [tab][tab]
wyatt rails_project$ rake 
about                      db:mongoid:remove_indexes  middleware                 spec
assets:clean               db:purge                   notes                      spec:models
assets:precompile          db:reseed                  notes:custom               spec:rcov
db:drop                    db:seed                    rails:template             stats
db:mongoid:create_indexes  db:setup                   rails:update               time:zones:all
db:mongoid:drop            doc:app                    routes                     tmp:clear
db:mongoid:purge           log:clear                  secret                     tmp:create

wyatt rails_project$ ll tmp/cache/
total 8
drwxr-xr-x  4 wyatt  staff   136B Mar  2 21:52 .
drwxr-xr-x  6 wyatt  staff   204B Feb 26 23:36 ..
-rw-r--r--  1 wyatt  staff   2.7K Mar  2 21:51 .rake_t_cache
drwxr-xr-x  4 wyatt  staff   136B Feb 26 23:36 assets
{% endhighlight %}

实在太棒了!




[bc]: http://bash-completion.alioth.debian.org/ 'bash-completion'
