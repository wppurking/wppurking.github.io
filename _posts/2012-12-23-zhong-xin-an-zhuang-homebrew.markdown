---
title: "重新安装 Homebrew"
date: 2012-12-23 20:47
comments: true
categories: ruby brew homebrew
---
## 起因
[homebrew (brew)](l1) 是 Mac OS 上一个类似 Ubuntu 上 apt-get 的软件, 从使用 Mac OS 以来, 所有 CLI 中的软件, 都是通过 `brew install` 安装的, 可是在一次 `brew update` 产生了大量大量的 CONFLICT 代码, 突然想起是不是我原来安装一个 app 的时候, 修改了 `/usr/local/Library/Formula` 下的安装脚本, 鬼使神差的我肯定还做了其他动作, T.T 想不起来了.

## 解决
看到那么多的 CONFLICT 我有了重新安装的想法, 去 [brew](l1) 的 github 的 issues 中寻找 [uninstall brew](https://github.com/mxcl/homebrew/issues/search?q=uninstall+brew), oh my god… 我掉进坑里了, 这里找到答案有点困难啊. So turn to google. [Uninstalling Brew to Reinstall Brew](http://zanshin.net/2012/07/31/uninstalling-brew-to-re-install-brew/), 和我要做的是一样的事情.

#### 备份原来利用 brew 安装的应用
准确的说应该是说备份利用 brew 安装的应用的软件名, 因为后面后直接利用 rm 将他们全部删除, 那么重新安装 brew 后再重新编译安装就好了. 通过 brew 安装的软件全部都在 `/usr/local/Cellar` 目录下.

#### 删除老的 brew
然后我就卡擦卡擦:

{% highlight bash %}
# 此刻我已经无法执行 brew 的任何命令了… 因为 brew 的代码里面也有 CONFLICT 
#cd `brew --prefix`
# 不我过知道 brew --prefix 指向的是哪里 :P
cd /usr/local
# 这个目录下包含的是通过 brew 下载源码,编译安装的软件的二进制执行文件
rm -rf Cellar
# 后面发现, 这不就是按照 brew github 中的目录结构去删除的吗 - -||
rm -rf Library .git .gitignore bin/brew README.md share/man/man1/brew
# 在我自己电脑上没有发现有缓存, 所以没有执行这条命令
#rm -rf ~/Library/Caches/Homebrew
{% endhighlight %}


#### 重新安装 brew

{% highlight bash %}
ruby -e "$(curl -fsSkL raw.github.com/mxcl/homebrew/go)"
{% endhighlight %}


#### 顿悟
在等待 homebrew 重新下载安装的过程中, 我打开了上面的[安装脚本](raw.github.com/mxcl/homebrew/go)阅读了起来. 我惊讶的看到了 `git reset --hard`

{% highlight bash %}
ohai "Downloading and Installing Homebrew..."
Dir.chdir HOMEBREW_PREFIX do
  if git
    # we do it in four steps to avoid merge errors when reinstalling
    system git, "init", "-q"
    system git, "remote", "add", "origin", "https://github.com/mxcl/homebrew"

    args = git, "fetch", "origin", "master:refs/remotes/origin/master", "-n"
    args << "--depth=1" if ARGV.include? "--fast"
    system *args
	
    # 这里这里~~~
    system git, "reset", "--hard", "origin/master"
  else
    # -m to stop tar erroring out if it can't modify the mtime for root owned directories
    # pipefail to cause the exit status from curl to propogate if it fails
    # we use -k because OS X curl has a bunch of bad SSL certificates
    # you may want to remove the -k flag from your fork!
    system "/bin/bash -o pipefail -c '/usr/bin/curl -skSfL https://github.com/mxcl/homebrew/tarball/master | /usr/bin/tar xz -m --strip 1'"
  end
end
{% endhighlight %}


我昏, 我好笨哦, 我怎么一开始没有想到在利用 `brew update` 在 `git merge` 发生错误之后将我本地的 brew 代码直接用 `git reset --hard` 成为最新的代码呢? 然后我 `cd /usr/local;git remote -v;ls -h` 然后又打开 github 上的源代码结构看了下. 哇, 这家伙将 brew 的安装设置得好巧妙, 直接将将 brew 安装在了 `/usr/local` 下(至少在我 Mac OS 10.6/10.7 下是的).

## 总结
这个问题绕了一圈最后回到了原点, 我可以直接执行 

{% highlight bash %}
cd /usr/local;git fetch origin;git reset --hard origin/master
{% endhighlight %}


来解决, 而我则绕了一大圈 [备份 -> 删除 brew -> 安装 brew -> 重新安装软件].  
想起一句话 "问题在解决之后, 会感觉这个问题好简单".



[l1]:http://mxcl.github.com/homebrew/