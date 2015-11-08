---
layout: post
title: "rspec 中的 should_receive"
date: 2013-02-27 23:11
comments: true
categories: 
---
第一次在 Rspec 中使用 method mock 测试, 所以就碰到了坑. 前段时候学习了 [Testing with Rspec](http://www.codeschool.com/courses/testing-with-rspec) 对 Rspec 入门. 现在真正使用起来, 还是会碰到很多小细节的问题, 例如今天碰到的这个: should_receive 所检查的对象.

## 两个概念
在课程的 mocking and stubbing 章节中有说明:

* Stub: For replacing a method with code that returns a specified result.
* Mock: A stub with an expectations that the method gets called.

在我自己的理解:

* Stub: 是用来替换掉原来的方法, 并且返回一个指定的值. 他注重的是在测试某一个方法内部调用其他方法的时候, 能够省去考虑内部某一方法的实现细节, 转而将这个原始方法使用另外一个 stub 来替换掉他并且给与指定的值, 以测试当前需要测试的这个方法.
* Mock: 首先一个 Mock 其实本身就是一个 stub, 不过还为其增加了对方法调用的期望测试. 他补充了普通 stub 会遗漏的一个点, **方法是否会被执行**, 就好比当前测试的方法内部有一个 if 语句满足才会调用内部另外一个方法, 而测试需要确保这个方法是被调用了(如果带上没有返回值更好理解), 那么 stub 则无法确保这个测试, 而 Mock 则可以.


## 例子
这里有一个使用 Mock 的例子
{% highlight ruby %}
it 'should not add to versions' do
  version = FactoryGirl.build(:version, created_at: Time.now - 20.hours, updated_at: Time.now - 20.hours)
  @listing.should_receive(:latest_version)

  expect {
    @listing.add_to_versions(version.attributes)
  }.to_not change { Version.count }
end
{% endhighlight %}

在这段代码中, 我希望测试一个名为 add_to_versions 的方法, 在这个 spec 中我希望测试的点有:

1. 这个 spec 中的 version 传入 add_to_versions 经过计算后, 会舍弃掉这个 version
2. 因为判断方法成功的标准和没有调用方法一样(Version.count 不变), 所以在 add_to_versions 的方法过程中, 我还需要判断其成功调用了 latest_version 确保是执行了对 version 的检查.

### Mock
按照这样的目的, 所以我对第二点的测试需要使用 mock 方法, 我期望在测试 add_to_versions 方法调用后 Version 的总数量不会改变, 但是需要确认调用过 latest_version 方法进行过判断. 所以会拥有
{% highlight ruby %}
@listing.should_receive(:latest_version)
{% endhighlight %}


加入 @listing 的 latest_version 没有被调用会抛出异常的(默认期望调用一次). 
对于默认情况的 mock 方法, 其实看看 rspec 对于 should_receive 的实现就能知道了(代码好绕 @,@), 他利用 `alias_method` 将原始方法改名藏起来了

	
**instance_method_stasher.rb**
{% highlight ruby %} 
def stash
  return if !method_defined_directly_on_klass? || @method_is_stashed

  @klass.__send__(:alias_method, stashed_method_name, @method)
  @method_is_stashed = true
end

def stashed_method_name
  "obfuscated_by_rspec_mocks__#{@method}"
end
{% endhighlight %}


最后这个 mock 方法就是一个 Rspec 的 MessageExpectation 对象, 并且被一个 MethodDouble 对象包含着, 同时 MethodDouble 又被一个 Proxy 包含着.

**method_double.rb**
{% highlight ruby %} 
def add_expectation(error_generator, expectation_ordering, expected_from, opts, &implementation)
  configure_method
  expectation = MessageExpectation.new(error_generator, expectation_ordering,
                                       expected_from, self, 1, opts, &implementation)
  expectations << expectation
  expectation
end

# 最后用来调用测试的入口
def verify
  expectations.each {|e| e.verify_messages_received}
end
{% endhighlight %}


也就是说, 从调用 `@listing.should_receive(:latest_version)` 后 Rspec 为我们做了:

1. 为当前对象添加了一个 Rspec Proxy 代理 [methods.rb]
2. 为当前对象与指定的方法包装在一个 MethodDouble 对象中 [proxy.rb]
3. 根据后续的 and_return, at_least 等等为 MethodDouble 初始化一个 MessageExpectation (一个方法) 对象并增加你期望的方法的行为 [method_double.rb, message_expection.rb, instance_method_stasher.rb]

如果再 `should_receive(:method_name)` 那 Rspec 会重用 Proxy 与 MethodDouble, 但会拥有新的 MessageExpectation.


### And_call_original
当我写完这个测试, 看着自己的 `@listing.latest_version` 的实现的时候发现, 如果我仅仅为 `latest_version` 增加一个 `should_receive` 那这个方法会拥有默认返回值为 `nil`, 那放到 `add_to_versions` 方法中, 那测试的不就不是我想要的逻辑了吗? 因为 `latest_version` 方法的返回值被我固定了啊? 可我期望的是能够正常执行 `latest_version` 找到最新的那个版本. 所以在 Rspec 官方找到了 [Calling the original method](l1) , 同时也将测试代码进行了调整

{% highlight ruby %}
# 将这个方法放到一个 context 中
context '#add_to_versions' do
  # 对所需要的数据进行初始化
  before do
    @listing.save
    3.times do |i|
      offset = 24 - (i * 5)
      FactoryGirl.create(:version, listing: @listing, created_at: Time.now - offset.hours, updated_at: Time.now - offset.hours)
    end
  end
  
  # 最后来测试
  it 'should not add to versions' do
    version = FactoryGirl.build(:version, created_at: Time.now - 20.hours, updated_at: Time.now - 20.hours)

    # 确保执行了 latest_version message
    @listing.should_receive(:latest_version).at_least(:once).and_call_original
    expect {
      @listing.add_to_versions(version.attributes)
    }.to_not change { Version.count }
  end
end
{% endhighlight %}


看到调用了 `and_call_original` 我脑袋里面在想, 这个是怎么弄的? 一个标示符?然后带着疑问打开了源代码看到了

**message_expectation.rb**
{% highlight ruby %} 
def and_call_original
  if @method_double.object.is_a?(RSpec::Mocks::TestDouble)
    @error_generator.raise_only_valid_on_a_partial_mock(:and_call_original)
  else
    @implementation = @method_double.original_method
  end
end
{% endhighlight %}

**method_double.rb**
{% highlight ruby %} 
def original_method
  #here
  if @method_stasher.method_is_stashed?
    ::RSpec::Mocks.method_handle_for(@object, @method_stasher.stashed_method_name)
  elsif meth = original_unrecorded_any_instance_method
    meth
  else
    begin
      original_method_from_ancestor(object_singleton_class.ancestors)
    rescue NameError
      raise unless @object.respond_to?(:superclass)
      original_method_from_superclass
    end
  end
rescue NameError
  Proc.new do |*args, &block|
    @object.__send__(:method_missing, @method_name, *args, &block)
  end
end
{% endhighlight %}

这段代码比较多, 主要作用就是去寻找, 应该很多时候都会是进入 method is stashed 中的判断语句. 因为 `add_expectation` 中调用了 `configure_method` 同时这个方法就对需要测试的方法的原始方法进行了 stash.

这个测试方法写到这里, 也算 ok 完成了, 哎, 谁叫自己刚刚接触 Rspec 呢? 一个测试方法写这个长, 看了这么多的源代码还写了这么多字, 感慨, 写出一个好的测试用例也不容易啊.

在刚开始阅读 Rspec 的文档的时候是一头雾水, 不知道从哪个地方开始看起, 只好从 CodeSchool 或者其他的地方了解了基本使用, 再回过头来写测试的时候发现真正的问题的时候才知道该如何去查, **温故而知新** 很有道理.



[l1]: https://www.relishapp.com/rspec/rspec-mocks/v/2-13/docs/message-expectations/calling-the-original-method! "Calling the original method"