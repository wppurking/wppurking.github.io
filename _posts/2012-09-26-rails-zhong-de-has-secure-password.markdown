---
layout: post
title: "Rails3 中的 has_secure_password"
date: 2012-09-26 21:26
comments: true
categories: 
---

在阅读完 [Ruby on Rails Tutorial][link1] 并且跟着书上的模拟 twitter 的例子完成编写之后, 虽然不能说什么都没有收获, 但对 Rails 的了解还仅仅停留在一个大体上的认知阶段, 如果真要我动手去做什么东西, 还是会不知道怎么动手. 所以自己就按照这个 twitter 的例子上所拥有的功能按照自己的想法来一一实现一次~ 然后将其中碰到的一个一个问题记录下来, 而 has_secure_password 就是其中的第一篇~

在 [Sample App][link2] 中, 虽然提到了 [device](https://github.com/plataformatec/devise) 这个 gem 来提供用户注册登陆这一块的功能, 但其还是选择自己来实现, 而实现的方式就是使用了 Rails3 中内置的 ClassMethods.has_secure_password 与自己来实现 session 登陆.

利用 [Dash](http://kapeli.com/dash/) 查看 Rails3 的文档, 阅读到 has_secure_password 方法, 然后我就知道了:

* 利用 BCrypt (bcrypt-ruby ~> 3.0.0) 生成了一个 `password_digest` 值
* 多了一个实例方法 `authenticate`
* 多了两个可读写字段  `password` 与 `password_confirmation`
* 自动增加了 `password` 的 presence 与 confirmation 检查

开始我挺糊涂的, 怎么也不知道这些是如何办到的, 所以我就利用 RubyMine 打开了对应的源代码(开源好啊 T.T), 看完源代码后发现其实我还是不知道这些方法是怎么添加到 Model 上去的, 不过还是理解了这些额外功能的出处, 源代码文件为: secure_password.rb 主要内容为:

* 引用了 bcrypt gem, 用来对 password 进行解密与加密用
* 利用 ruby 的 attr_reader 添加了 :password 字段的读取
* 利用 rails 的 Validate 框架添加了

{% highlight ruby %}
validates_confirmation_of :password
validates_presence_of :password_digest
{% endhighlight %}


* 添加了 module InstanceMethodOnActivation, 也就是引入了 authenticate 与 password= 两个方法, 一个用来登陆, 一个用来对接收的密码进行处理

看完源代码后 !! 果然, 果然没有 password 的 presence 检查, 我说怎么在 `rails c` 中, 怎么不初始化 password 不会报告为空错误, 原来这里根本没有添加嘛, 这也难过在 [Sample App][link2] 中需要自己添加对 password 与 password_confirmation 的检查代码了. 不过添加对 `password` 的存储也没有必要, 因为在 rails 中存储的是通过单向加密处理后的 `password_digest` 而 `password` 和 `password_confirmation` 都是中间临时使用的值.

看完 module Instancemethodonactivation 中的方法, 就彻底明白 `has_secure_password` 存在什么魔法了.

{% highlight ruby %}
module InstanceMethodsOnActivation
  # Returns self if the password is correct, otherwise false.
  def authenticate(unencrypted_password)
    if BCrypt::Password.new(password_digest) == unencrypted_password
      self
    else
      false
    end
  end
    
  # Encrypts the password into the password_digest attribute.
  def password=(unencrypted_password)
    @password = unencrypted_password
    unless unencrypted_password.blank?
      self.password_digest = BCrypt::Password.create(unencrypted_password)
    end
  end
end
{% endhighlight %}


看到这两个方法就差不多知道做了什么了, 设置 `password` 的时候, 利用 BCrypt 对 `password` 加密成为 `password_digest` 用来存储到数据库中, 验证密码的时候, 将用户输入的密码通过 `def password=` 方法进行加密, 和存储的 `password_digest` 进行比较验证.

刚刚看到 `BCrypt::Password.new(password_digest) == unencrypted_password` 可能会有点疑惑, 为什么是拿加密的 `password_digest` 和 没有加密的 `unencrypted_password` 进行比较啊? 而且也没有对 `unencrypted_password` 做任何操作啊? 为什么说是比较的加密后的内容呢? 这是因为在 `BCrypt` 库中, 其重写了 `BCrypt::Password` 类的 `==` 方法

{% highlight ruby %}
# Initializes a BCrypt::Password instance with the data from a stored hash.
def initialize(raw_hash)
  if valid_hash?(raw_hash)
    self.replace(raw_hash)
    @version, @cost, @salt, @checksum = split_hash(self)
  else
    raise Errors::InvalidHash.new("invalid hash")
  end
end


# Compares a potential secret against the hash. Returns true if the secret is the original secret, false otherwise.
def ==(secret)
  super(BCrypt::Engine.hash_secret(secret, @salt))
end
alias_method :is_password?, :==
{% endhighlight %}


虽然代码上写的是 `BCrypt::Password.new(password_digest) == unencrypted_password`, 实际上比较的是经过加密后的内容.


恩, 到这里这个 `has_secure_password` 算是搞明白鸟, 收工.



[link1]: http://ruby.railstutorial.org/ "Ruby on Rails Tutorial"
[link2]: https://github.com/railstutorial/sample_app "Sample App"