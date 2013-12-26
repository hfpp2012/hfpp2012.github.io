---
layout: post
title: "ruby元编程之block"
date: 2013-12-26 17:36
comments: true
categories: ruby rails metaprogramming
---
>> 最简单的方式来用block

## block

我们先看最简单的例子

``` ruby
def awesome_print
  yield
end
awesome_print { puts "I'm xiaozi" }

# =>
I'm xiaozi
```

也就是说任何方法你都可以给它传一个block,但是要使用yield去执行这个block

另外,```awesome_print { puts 'I'm xiaozi }```太难看了,那就

``` ruby
awesome_print do
  puts "I'm xiaozi"
end
```

漂亮了吧

太简单可不行,总得传参数

``` ruby
def awesome_print name
  yield name
end
awesome_print('xiaozi') do |name|
  puts "I'm #{name}"
end

# =>
I'm xiaozi
```

总结一下yield,它就是块的占位符,awesome_print把参数值'xiaozi'传给```do ... end```块中的name参数,yield占的坑让给了```do ... end```块,并执行,name参数的值'xiaozi'传给```yield name```中的name参数

很简单吧!

上文说了任何方法都可以传入一个block,那怎么判断传block了没,因为有时候要根据有没有传block来定制一些业务

``` ruby
def awesome_print
  yield if block_given?
  'no block'
end
```

明白了吗,就是用block_given?

## proc

它和block有什么不同呢

``` ruby
say_hello = Proc.new { |name| "I'm #{name}" }
say_hello.call('xiaozi')

# =>
I'm xiaozi
```

用了关键字Proc.new来创建一个block赋给一个变量,再用call来调用的

原来block还可以取个名字的哦

## lambda

``` ruby
say_hello = lambda { |name| "I'm #{name}" }
say_hello.class # => Proc
say_hello.call('xiaozi')

# =>
I'm xiaozi
```

## &

``` ruby
def awesome_print(&the_proc)
  the_proc
end

p = awesome_print { |name| "I'm #{name}" }
p.class # => Proc
p.call('xiaozi')

# =>
I'm xiaozi
```





