---
title: "Around Alias"
date: 2019-12-12T22:56:33+08:00
draft: true
---


## Ruby 中的环绕别名 (around Alias) 带来的一系列 Google

先来看一段代码，体验一下什么是环绕别名 (around Alias) :

```ruby
class String
  alias :old_length :length

  def length
    "Length of'#{self}'is: #{old_length}"
  end
end

"abc".length
#=> "Length of'abc'is: 3"
```

后面去 RubyChina 学习发现这是一种 Alias method chain pattern, 在 Rails3 中使用非常广泛，在 Rails5 上面被 prepend 替代了。在 Ruby 2.0 出来之前，Alias method chain pattern 这种做法在 Rails3 ActiveSupport 提供了 `alias_method_chain` 来辅助实现。

归根结底: 通过环绕别名，既修改了自己的逻辑，又不需要调用者更改自己的代码

## 什么是 alias_method_chain

很多人都很迷茫，不知道这个方法怎么用，所以栈爆网有这么一个问题: [Ruby on Rails: alias_method_chain, what exactly does it do?](https://stackoverflow.com/questions/3695839/ruby-on-rails-alias-method-chain-what-exactly-does-it-do)

when would you use alias_method_chain and why?

> the following is largely based on the discussion of `alias_method_chain()` in Metaprogramming Ruby by Paolo Perrotta, which is an excellent book that you should get your hands on.)

这里指的是 《Ruby 元编程》这本书里面提及的 AR 相关部分代码解析，具体可以查书了。（我没看懂那部分... T.T）

看段代码吧:

```
class Klass
  def salute
    puts "Aloha!"
  end
end

Klass.new.salute # => Aloha!
```

现在希望给代码加上一些日志扩展方法功能：

```
class Klass
  def salute_with_log
    puts "Calling method..."
    salute_without_log
    puts "...Method called"
  end

  alias_method :salute_without_log, :salute
  alias_method :salute, :salute_with_log
end

Klass.new.salute
# Prints the following:
# Calling method...
# Aloha!
# ...Method called
```

看完代码，其实就是你定义第一个版本 `foo()`，然后扩展一个 feature 得到 `foo()` 和  `foo_with_feature()` 然后再定义一个没有 feature 原始方法 `foo_without_feature()`，为了避免到处都是 `alias` 的定义，ActiveSupport 提供了一个 `alias_method_chain` 来处理。

这样你只需要写: `alias_method_chain :foo, :feature`

你就得到了 3 个方法: `foo`, `foo_with_feature`, `foo_without_feature`

但是在 Ruby 2.0 中出来新的 feature `prepend` 可以替代这种设计模式 or alias_method_chain

## Ruby Prepend 是干嘛的

这个部分可以参考 RubyChina 另外一个大神写的文章 [理解 Ruby 中的 include 和 prepend](https://ruby-china.org/topics/28712)

从 Ruby 继承链来看就是:

- include 将 module 加入到 类 继承链的上方
- prepend 将 module 加入到 类 继承链的下方

同时想知道怎么样理解和处理 `Prepend` 和 `Alias method chain`  可以参考 NewRelic 工程师团队的博客文章 [ruby-agent-module-prepend-alias-method-chains](https://blog.newrelic.com/engineering/ruby-agent-module-prepend-alias-method-chains/)

对于 Alias method chain 和 prepend 之间使用的坑，NewRelic 工程师团队也给出了建议:

- only Module#prepend
- only alias_method
- have Module#prepend happen after alias_method
