---
layout: default
title: "gokogiri 的使用小试"
date: 2013-08-30 10:39
comments: true
categories: 
---

## 介绍
因为 Golang 的编译拥有方便部署的特性, 一直想将现在系统中一部分独立抓取业务使用 Golang 编写, 这样编译好以后, 直接扔到系统上就可以运行起来很简单, 可是一直没有找到一个合适的 html 的解析库, 直到后面发现 [gokogiri](https://github.com/moovweb/gokogiri) 发现这真是一个很棒的解析库, 依赖 C 库 libxml. 相应的其他的 html parser 工具还有 [goquery](https://github.com/PuerkitoBio/goquery) 其依赖 golang 的 go.net/html 库, 因为库成熟度的原因我还是选择了 gokogiri.

## Gokogiri 基本使用
比较悲剧的是, gokogiri 项目中介绍使用的文档一点都没有, 很是烦恼所以只能慢慢的阅读他的代码与相关的测试来查看这个库该如何使用. 下面就通过 "抓取 www.google.com 上的所有 a 标签" 为例子来介绍一下使用.

### 首先设置好自己的 GOPATH
{% highlight bash %}
cd ~
mkdir goo
cd goo
GOPATH=~/goo
{% endhighlight %}

### 然后我们需要添加依赖:  `go get github.com/moovweb/gokogiri`


### 代码实现
**main.go**
{% highlight go %}
package main

import (
	"github.com/moovweb/gokogiri"
	"io/ioutil"
	"log"
	"net/http"
)

func main() {
	resp, err := http.Get("http://www.google.com")
	defer resp.Body.Close()
	handlErr(err)

	html, err := ioutil.ReadAll(resp.Body)
	handlErr(err)

	// 解析 html 成为 gokogiri 的 Document 对象
	doc, err := gokogiri.ParseHtml(html)
	handlErr(err)
	defer doc.Free()

    // Search 是 Document 的解析 API, 这里只能填写 XPath
	nodeArr, err := doc.Search("//a")
	handlErr(err)

	// 返回的是 []Node
	for _, node := range nodeArr {
		log.Println(node)
	}

}

// 我发现每写一行代码都要处理下 error ...
func handlErr(err error) {
	if err != nil {
		log.Fatalln(err)
	}
}
{% endhighlight %}

通过 `go run main.go` 即可.

## Gokogiri 通过 CSS
现在 jQuery 在前端非常的流行, 其所带来的 CSS 类似的选择器也大受 Html 解析库的欢迎, 例如 Java 中的 Jsoup, Python 中的 pyquery, Golang 中的 goquery, Ruby 中的 nokogiri 等等, 所以很多时候我也想通过 CSS 的选择器语法来搜索元素. 好的是, gokogiri 提供了通过 CSS 来解析文档的特性, 不好的是这个特性并没有集成进行 `Document.Search()` 这个 API 中. 我们将上面的例子修改为通过 CSS 解析.

### 添加 gokogiri/css 所依赖的库:
因为 gokogiri 的 css 解析库依赖 [rubex](github.com/moovweb/rubex) 并且其还依赖一些另外的库, 所以需要额外安装, 如其 github 仓库所介绍的:

Mac OS: `brew install oniguruma` 
Ubuntu: `sudo apt-get install libonig2`

因为我是 Mac OS 所以就执行了第一行, 如果是 windows 的话, "呵呵"

安装好以后就可以添加依赖了 `go get github.com/moovweb/rubex` 


### 调整代码

**main.go**
{% highlight go %}
package main

import (
	"github.com/moovweb/gokogiri"
	// 区别之一: 引入 gokogiri/css 库
	"github.com/moovweb/gokogiri/css"
	"io/ioutil"
	"log"
	"net/http"
)

func main() {
	resp, err := http.Get("http://www.google.com")
	defer resp.Body.Close()
	handlErr(err)

	html, err := ioutil.ReadAll(resp.Body)
	handlErr(err)

	// 解析 html 成为 gokogiri 的 Document 对象
	doc, err := gokogiri.ParseHtml(html)
	handlErr(err)
	defer doc.Free()

	// 区别之二: 使用 css 库的 Conver 函数
	nodeArr, err := doc.Search(css.Convert("a", css.GLOBAL))
	handlErr(err)

	for _, node := range nodeArr {
		log.Println(node)
	}

}

// 我发现每写一行代码都要处理下 error ...
func handlErr(err error) {
	if err != nil {
		log.Fatalln(err)
	}
}
{% endhighlight %}

通过查看 gokogiri 的 css 包的源码与其测试可以看出, gokorigi 用来支持 CSS 选择器的方式是将 CSS 选择器语法转换成为了 XPath , 而在 `Document.Search()` 没有将 css 的语法集成进来很大的原因我测应该是对 css 语法转换为 XPath 语法所额外需要的依赖包的问题.


