---
layout: default
title: "使用 Golang 编写小应用后的感受"
date: 2013-08-10 22:59
comments: true
categories: 
---
## 起因
最近公司里面的人数增多了, 而我们现在为保证公司正常运行的业务所需要的翻墙都在 [wyatt_host](https://github.com/wppurking/wyatt_hosts) 这个当中, 而大家似乎都不太会知道如何去替换掉自己系统中的 hosts 文件, 所以决定写一个小应用 [hosts_replacer](https://github.com/wppurking/hosts_replacer) 来自动更新系统中的 hosts. 

可是问题就在这里, 大家使用的系统都是 Windows 而我所在的开发环境是 Mac OS, 在我的环境下我可以索性写一个 ruby 脚本很快的解决这个问题, 可是当我写好这个 ruby 脚本让他们使用的时候则会碰到需要到 windows 电脑上安装 ruby 解释器的情况, 这真的会是得不偿失啊, 所以将想到了 Golang 这个可以编译成执行执行的二进制文件语言. 这样我只要写一次, 然后编译一个 exe 文件大家只要下载这个 exe 双击一下就 ok 啦~ 

## 感受
在各种 VM 横飞的现在, 一个新的编译语言的出现给我最直接的感觉就是, "部署真是超级简单".  如果用 Java 我要下载 Java JRE, 如果用 Ruby 我要下载 Ruby 解释器, 如果用 Python 我要下载 Python 解释器, 如果用 JavaScript 我要下载 NodeJS 等等, 当然 C 和 C++ 应该也可以, 可是这两门语言的代码我现在不会写…

### 方法的组织
在编写这个小应用的时候, 抛开核心库语言特性等等不说, Golang 中对功能代码的组织方式的确是让我感觉非常新奇的一个地方, 以前在看别人的 Golang 代码的时候发现, 为什么大家都是在一个文件中所有方法都从文件的最左边开始, 没有一点缩进的呢? 因为我们常用的 Class 模型去编写代码至少都会有一个 class 关键字加上一个大括号或者缩进来将方法归到这一个 Class 中. 也正是因为这样我就在想, 我们现在都是通过类, 通过包, 或者命令空间来组织方法, 将方法归类到一起, 而在 Golang 中, 我可以随意的通过目录, 文件来将方法物理的归类到一起, 但是在内存中呢? 我却可以夸文件的调用方法, 就像


{% highlight go %}
// 我可以
strings.Replace("abc", "a", "99", -1)
// 也可以
strings.Split("a,b,c,d", ",")
{% endhighlight %}

但你通过 `godoc -http=:8080` 启动 golang 文档, 并且访问到 [Package files](http://0.0.0.0:8080/pkg/strings/#pkg-examples) 会发现, `Replace` 方法是在 replace.go 文件中, `Split` 方法是在 `strings.go` 文件中. 突然想到 ruby 也可以通过 open class 的形式来达到这样的效果, 只是 Golang 让这种组织方法的方式很自然. 我不知道常规意义上使用 class 来组织方法的方式和 Golang 这种方式谁好, 不过 Golang 的这种物理组织方式与内存中组织方式的混合对我感觉的确挺新鲜的.

### 编译器超严格
我从没看到哪个编译器会对没有使用过的变量或者引入包进行编译器报错的, 也是第一次看到 Golang 的编译器竟然在编译代码的时候告诉我有一个地方存在死循环的问题 O_o

这个严格的编译器导致我在使用 "fmt" 输出信息查看代码运行的时候, 我总得在 import "fmt" 和注释掉 import "fmt" 之间来回切换. 也使得我有时候写完一次代码一编译发现给我一下提示了 5, 6 个错误, 让我慢慢改 - -\|\|

由于这强悍的编译器, 所以在 golang 中类型之间的转换也是相当的严格. 最常见的情况是, 想将某些变量转换为 string 类型输出. 在 ruby 中我们可以 `"abc" + 123.to_s` , 在 Java 中有自动调用 `toString()` 所以可以 `"abc" + 123`, 而在 Golang 中则需要借助 "strconv" 这个 lib 来完成 - -\|\| 代码一下就多了, 

{% highlight go %}
// 伪代码
import "strconv"
i := 123
"abc" + strconv.Iota(i)
{% endhighlight %}

我只感觉在 Golang 中不同类型向 string 的转换相当麻烦, 不过按照类型系统来看的话, int (其他类型) 和 string 是两个完全不同的类型, 不同类戏之间使用 `+` 或者强制转换就算有, 也的确应该需要足够严格才好.


最后我只想说, 这个编译器很尽责.

### 基础工具超实用
对代码格式, 我现在团队里面都是统一使用一个 IDEA 的 settings.jar 配置, 然后统一使用 IDEA 提供的代码格式化来处理. 在 Golang 中, 直接为我们提供了 `gofmt -w .` 然后我当前目录以及子目录中的所有 .go 文件全部按照一个统一格式处理好了. 这个工具很直接, 很简单!

我不知道是不是 *nix 的影响, 现在的依赖管理基本上都会是一个统一类似的概念, 通过某个命令自动到一个中心或者可配置的中心仓库去寻找所需要的库, 例如 java 的 maven, ruby 的 rubygem, python 的 easy_install ,这里 golang 则有一个 `go get`.

Golang 提供的这些工具, 使用起来都是很直接, 很简单, 不用太多想法.



## 最后
总的来说 Golang 还是一门不错的语言啦, 至少对我最大的吸引在于他是编译的, 同时语法真的是相当相当的简单了. 类型系统(12 个类型), 程序语句(if, for, go, select..), Error 处理, 包处理 然后, 然后就差不多了吧 ^_^





