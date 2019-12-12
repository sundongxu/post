---
title: "Best Practice about ROR"
date: 2019-12-12T22:17:42+08:00
categories:
    - Ruby
tags: 
    - Ruby
    - Rails
    - BestPractice
---

[TOC]

## About ruby

### 带 block 遍历数组应该使用 each 而不是 for
before:

这里需要遍历数组并且打印

```
for elem in [1, 2, 3, 4, 5]
    puts elem
end
```

Ruby 里面使用 each 带一个 block是一个更有效率的做法
after:

```
[1, 2, 3, 4, 5].each do |elem|
	puts elem
end
```

还可以变得更简单一点:

```
[1, 2, 3, 4, 5].each { |elem| puts elem }
```

### 收集结果(遍历数组元素的处理结果)使用 map 而不是 each + <<
这里希望收集数组中特定元素的大写形式

before:

```
result = []
['biology', 'english', 'Math'].each do |major|
  next if major.size < 5
  result << major.capitalize
end

puts result
```

使用 `map` 会过滤和收集每次循环阶段 block 处理的结果，组织后返回

after:

```
result = ['biology', 'english', 'Math'].map do |major|
  next if major.size < 5
  major.capitalize
end

puts result
```

对于数组每个元素希望过滤部分元素可以使用 `reject` 配合 block，这样可以写成一行

```
result = ['biology', 'english', 'Math'].reject{ |m| m.size < 5 }
                                       .map { |m| m.capitalize }

puts result
```


### 复杂条件判断使用 case/when 而不是 if/elsif/else

before:

```
major = gets.chomp

if major == "Biology"
	puts "Mmm the study of life itself!"
elsif major == "Computer Science"
	puts "I'm a computer!"
elsif major == "English"
	puts "Sweet! I'm great with numbers!"
elsif major == "Math"
else
	puts "That's a cool major!"
end
```


使用 `case/when` 之后，判断的条件更加清晰，更好阅读
after:

```
major = gets.chomp

case major
when "Biology"
	puts "Mmm the study of life itself!"
when "Computer Science"
	puts "I'm a computer!"
when "English"
	puts "No way! What's your favorite book?"
when "Math"
	puts "Sweet! I'm great with numbers!"
else
	puts "That's a cool major!"
end
```

### 使用 !! 判断值是否存在(不是 Nil)

这里有个类 (Person) 希望提供一个方法判断名字是否为 nil

```
class Person
  def initialize(name=nil)
    	@name = name
  end

  def has_name?
	  !!@name
  end
end

Person.has_name?            #=> false
Person('lion').has_name?    #=> true
```

虽然 `!!` 有点难以理解，这里解释一下。`!` 是对值求反，`nil` 在 `true/false` 中会被处理为 `false`，这样 `!nil` => `true`，再求反一次，就变成了 `false`

```
@name       #=> nil
!@name      #=> true
!!@name     #=> false
```

```
@name       #=> lion
!@name      #=> false
!!@name     #=> true
```

### Hash 中的 string key 最好使用 symbol 而不是 string

`symbol` 是 ruby 的一种类型，类似于 `string`。当使用场景里面需要存储不需要变更的字符串时候，基本都推荐使用 `symbol`，因为从内存角度来看更省内存和高效

可以看个例子:
```
str1 = 'a'
str2 = 'b'
str1.object_id == str2.object_id    #=> false

sym1 = :a
sym2 = :a
sym1.object_id == sym2.object_id    #=> true
```

在 hash 中使用的例子:

before:

```
our_garden = { "roses": 9, "rhododendrons": 2, 
               "poppies": 12, "geraniums": 6 ,
               "sneezeworts": 5 }
```

after:

```
our_garden = { roses: 9, rhododendrons: 2, 
               poppies: 12, geraniums: 6 ,
               sneezeworts: 5 }
```

### 使用 unless 而不是 !if

在一些函数的调用结果希望进行取反之后做处理，建议使用 `unless` 而不是 `!if`

下面例子希望在没有 name 情况下，不要打印任何信息，使用 `unless` 之后代码变得更好阅读

```
class Person
  def initialize(name=nil)
    	@name = name
  end

  def has_name?
	  !!@name
  end
  
  def name
    @name
  end
end

def echo_name(person = nil)
    return if !person.has_name?

    puts person.name
end
```

after:

```
def echo_name(person = nil)
    return unless person.has_name?

    puts person.name
end
```

### 返回 bool 的函数名需要加上 ?

before:

```
def exist
    false
end
```

after:

```
def exist?
    false
end
```

## About Rails

### Prevent SQL Injection

很多场景里面需要对数据库查询条件进行拼接，这里有注入的风险

before:

```
User.where("name = #{params[:name]}")
```

after:

```
User.where('name = ?', params[:name])
```

or

```
User.where(name: params[:name])
```

### default_scope is evil

建议不要在 model 里面加 `default_scope`, 先看代码吧

```
class Post
  default_scope where(published: true).order(created_at: :desc)
end
```

然后你会发现很多情况下都不会按照你想要的实现:

默认情况下按照 created_at 排序返回

```
> Post.limit(10)
  Post Load (3.3ms)  SELECT `posts`.* FROM `posts` WHERE `posts`.`published` = 1 ORDER BY created_at desc LIMIT 10
```

想要只用 updated_at 排序，然后就不对劲了:

```
> Post.order("updated_at desc").limit(10)
  Post Load (17.3ms)  SELECT `posts`.* FROM `posts` WHERE `posts`.`published` = 1 ORDER BY created_at desc, updated_at desc LIMIT 10
```

如果想要做到只用 updated_at 排序就得这么做:

```
> Post.unscoped.order("updated_at desc").limit(10)
  Post Load (1.9ms)  SELECT `posts`.* FROM `posts` ORDER BY updated_at desc LIMIT 10
```

而且在 model 的 initialization 也会被影响, 这往往不是你想要的:

```
> Post.new
=> #<Post id: nil, title: nil, created_at: nil, updated_at: nil, user_id: nil, published: true>
```

所以，使用 `scope` 情况中，不要使用 `default_scope` (不是特别必要)，而是定义 `scope` 并且手动调用

### model.save! 或者 model.save 并且确认返回值

在调用 model.save 时，如果数据无效，并不会被保存，这时候需要手动确认 save 的返回值

```
post = Posts.new do |p|
  p.title = 'example'
  p.body = nil
end

raise 'post is invalid' unless post.save
```

`Post.body` 在数据库中是不能为 `NULL` 的，但是 `save` 可以正常执行但是没有保存，需要手动确认，然后确认结果又不是很直观。

这个时候使用 `save!`，就会直接知道问题，达到目的

```
post = Posts.new do |p|
  p.title = 'example'
  p.body = nil
end

post.save!
```
