---
layout: post
title: "Golang 的中的 windows.go darwin.go"
date: 2013-09-07 10:42
comments: true
categories: 
---
## 起因
前几天无意中逛了一下 [foreman 作者的 github](https://github.com/ddollar?tab=repositories) 发现他用 Golang 重新实现了一次 foreman, 然后我就很感兴趣的打开看了看. 现在看起来这里面的代码还是挺少的, 只有 13 个文件, 大概 600+ 行的代码 (cat *.go | wc -l) , 所以就每个文件都打开详细看了看. 然后我这一看就有了这篇文章. 

## 疑问
在 [forego](https://github.com/ddollar/forego) 中有这么几个文件: [process.go](https://github.com/ddollar/forego/blob/2cdf2cd7cb9bbd97f6f5591659493c026130c59e/process.go)  [darwin.go](https://github.com/ddollar/forego/blob/2cdf2cd7cb9bbd97f6f5591659493c026130c59e/darwin.go) [linux.go](https://github.com/ddollar/forego/blob/2cdf2cd7cb9bbd97f6f5591659493c026130c59e/linux.go) [windows.go](https://github.com/ddollar/forego/blob/2cdf2cd7cb9bbd97f6f5591659493c026130c59e/windows.go) 其中在 `process.go` 中定义了 `type Process struct` 用来抽象具体运行一个应用的进程, 然后在另外三个文件中都有三个相同的方法

**相同方法**
{% highlight go %}
func (p *Process) Start() {
// do something
}

func (p *Process) Signal(signal syscall.Signal) {
// do something
}

func ShutdownProcesses(of *OutletFactory) {
// do something
}
{% endhighlight %}

问题就来了, 所有的文件都是放在相同的 `package main` 中, 难道 Golang 允许 `Process struct` 拥有三个相同的方法存在吗? 虽然我知道 Golang 能够通过 package 让代码在逻辑上处于一个整体, 而在物理上分割为多个文件, 可当不同文件中的代码被编译器处理到一起的时候难道相同方法也会合并吗?

## 寻找
我开始在想, 是不是他将三个平台的文件写到一个项目中, 需要针对某一个平台编译的时候, 将另外两个文件先暂时挪开, 而为了方便管理将他们放在同一个 repo 中呢? 所以我把代码 clone 下来然后将依赖弄好进行编译, 我在 Mac OS 上进行编辑, 我没有处理 `windows.go` `linux.go` 这两个文件执行一下 `go build` 结果是成功, 并且程序可以正常运行.

我猜测是不是文件名的特性, 所以我将 `darwin.go` 改名为 `darwin_bak.go` 结果也是成功.

接下来, 我就来验证 Golang 是不是真的可以存在多个相同方法, 看是不是会让我大吃一惊.然后就有了下面代码

**a.go**
{% highlight go %}
package main

import (
"log"
)

func (p *P) Start() {
  log.Println("a.Start()")
}
{% endhighlight %}

**b.go**
{% highlight go %}
package main

import (
"log"
)

func (p *P) Start() {
  log.Println("b.Start()")
}
{% endhighlight %}


**main.go**
{% highlight go %}
package main

import (
"log"
)

type  P struct {
  Name string
}

func NewP() (p *P) {
  p = new(P)
  p.Name = "wyatt"
  return
}


func main() {
  p := NewP()

  log.Println(p)
  p.Start()
}
{% endhighlight %}

结果运行 go build 的结果是

{% highlight bash %}
./b.go:7: (*P).Start redeclared in this block
	previous declaration at ./a.go:7
{% endhighlight %}


送了一口气, 还好和想象的一样. 把我难倒了, 我开始上网 Google. 在尝试了一大堆的关键字 [windows.go](https://www.google.com/#hl=en&newwindow=1&q=windows.go&qscrl=1)  [golang windows.go filename](https://www.google.com/#hl=en&newwindow=1&q=golang+windows.go+filename&qscrl=1) [golang darwin.go](https://www.google.com/#hl=en&newwindow=1&q=golang+darwin.go&qscrl=1) 最后终于在 [go build 的文档中找到了](http://golang.org/pkg/go/build/)

## 结论

> If a file's name, after stripping the extension and a possible _test suffix, matches any of the following patterns:
> 
> *_GOOS
> 
> *_GOARCH
> 
> *_GOOS_GOARCH
>
>  (example: source_windows_amd64.go) or the literals
>
> GOOS
> 
> GOARCH
> 
> (example: windows.go) where GOOS and GOARCH represent any known operating system and architecture values respectively, then the file is considered to have an implicit build constraint requiring those terms.
 
和 `go test` 使用文件名做区分类似, 这个使用文件名区分用途的规则也被用倒了 `go build` 上, `GOOS` 用于代表需要目标操作系统, 例如 windows, darwin, linux. `GOARCH` 用于代表硬件环境, 例如 386, amd64, arm. 而当文件名是以 `GOOS` 和 `GOARCH` 命名的时候, `go build` 只会有针对性的加载对应平台的那份代码进行编译.

原来如此, 就是 `go build` 有这样的特性, 我又找了几个其他的 Golang 的开源项目, 除了 [forego](https://github.com/ddollar/forego) 外还有 [fsnotify](https://github.com/howeyc/fsnotify) 也是, 看来这个特性是自己孤陋寡闻了.

