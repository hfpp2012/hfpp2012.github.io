---
layout: post
title: "ruby元编程之block"
date: 2013-12-26 17:36
comments: true
categories: ruby rails metaprogramming
---
>> 以最简单的方式来理解block

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

也就是说任何方法你都可以给它传一个block,而且必须是最后一个参数,但是要使用yield去执行这个block

另外,```awesome_print { puts 'I'm xiaozi }```太难看了,那就

<!-- more -->

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

这样也行

``` ruby
def awesome_print
  yield "xiaozi"
end
awesome_print { |name| puts "I'm #{name}" }

# =>
I'm xiaozi
```

很简单吧!

如何判断一个方法传过来block了?

``` ruby
def awesome_print
  yield if block_given?
  'no block'
end
```

明白了吗,就是用block_given?

在这里,```{ ... }```或```do ... end```跟yield就是成对存在的,传了block,执行时就要用yield

## Proc

它和block有什么不同呢

``` ruby
say_hello = Proc.new { |name| "I'm #{name}" }
say_hello.call('xiaozi')

# =>
I'm xiaozi
```

用了关键字Proc.new来创建一个block对象,然后赋给一个变量,最后用call来调用的

原来Proc.new创建的block对象还可以赋给一个变量

在这里,用Proc.new创建的block对象,要由call来执行和调用

## lambda

``` ruby
say_hello = lambda { |name| "I'm #{name}" }
say_hello.class # => Proc
say_hello.call('xiaozi')

# =>
I'm xiaozi
```

跟Proc差不多,lambda也创建一个block对象,而且它的类还是Proc

在这里,用lambda创建的block对象,要由call来执行和调用

## ->

ruby 1.9之后用这个来代替lambda

于是```say_hello = lambda { |name| "I'm #{name}" }```变成了```say_hello = -> { |name| "I'm #{name}" }```

``` ruby
say_hello = -> { |name| "I'm #{name}" }
say_hello.class # => Proc
say_hello.call('xiaozi')

# =>
I'm xiaozi
```

推荐用这种方法

### return yield

yield能作为返回值出现

``` ruby
def awesome_print
  puts yield
end
awesome_print { "I'm xiaozi" }

# =>
I'm xiaozi
```

### 传多个block对象

一个block对象作为参数不够,怎么办

``` ruby
def awesome_print(who, where)
  "#{who.call} live in #{where.call}"
end
who = -> { "xiaozi" }
where = -> {'shenzhen'}
awesome_print(who, where)

# =>
"xiaozi live in shenzhen"
```

列好队,用block对象传过去就行了

## block vs lambda vs Proc

lambda跟Proc在形式上还比较相似,但它们跟```do ... end```或```{ ... }```这种形式相差好多

不信?来看下


### & 区别块和块对象

``` ruby
def awesome_print(&the_proc)
  the_proc
end
my_proc = awesome_print { "I'm xiaozi" }
my_proc.class # => Proc
my_proc.call

# =>
I'm xiaozi
```

&这个符号会不会很熟悉呢,对,C语言就有,意思是取地址

难道这里的block跟地址有关系?

在这里&the_proc表示接收的参数的值要是一个```do ... end```或```{ ... }```这样的块,而不是由Proc.new或lambda创建的块对象

注意区分**块**和**块对象**

很明显```do ... end```或```{ ... }```这样的块跟&是一对的

假如把```{ .. }```换成lambda

``` ruby
def awesome_print(&the_proc)
  the_proc
end
my_proc = lambda { "I'm xiaozi" }
awesome_print(my_proc)

# =>
ArgumentError: wrong number of arguments (1 for 0)
```

出错了吧,看来lambda这样的块对象和&不是一对的

那给lambda前面加个&(类似于C语言的取地址)

``` ruby
def awesome_print(&the_proc)
  the_proc
end
awesome_print(&lambda { puts "I'm xiaozi" }).call
# or
my_proc = lambda {  "I'm xiaozi" }
awesome_print(&my_proc).call

# =>
I'm xiaozi
```

果然把lambda前加个&就行

或者这样也行

``` ruby
# 注意这行&the_proc 变成了 the_proc
def awesome_print(the_proc)
  the_proc
end
my_proc = lambda {  "I'm xiaozi" }
awesome_print(my_proc).call

# =>
I'm xiaozi
```

Proc也是跟lambda一样的,不信?你去试试

总结一下,block跟&是一对的,block对象(lambda或Proc)取&才跟&是一对的,block和block对象的区别在于&,block对象取&之后才在形式上跟block一样,
&是把block转成了block对象

### & yield call相互转化

上面说过块block给方法后用yield来执行和调用,但是如果block传给&的参数变量,又可以用call来执行,到底怎么回事

``` ruby
# yield & block
def awesome_print
  yield
end
awesome_print { "I'm xiaozi" }

def awesome_print(code)
  code.call
end
awesome_print lambda { "I'm xiaozi" }
```

这两种方法上面都说过了,但是`lambda { "I'm xiaozi" }`不给call执行,要给yield执行,可以吗

``` ruby
def awesome_print
  yield
end

awesome_print(&lambda { "I'm xiaozi"} )
```

这样都行!它们有一个相同的联系点?

### Symbol’s To Proc

你肯定用过类似这样的方法吧```(1..10).inject(&:+)```或者```[:a, :b, :c].map(&:to_s)```等

``` ruby
[:a, :b, :c].map(&:to_s)

# =>
["a", "b", "c"]
```

好神奇,怎么做到的?

&:to_s能够作为参数值传递,其中:to_s是一个symbol,&:to_s会将to_s转成一个proc对象,结果相当于:to_s.to_proc

任何symbol都有to_proc方法

``` ruby
class Symbol
  def to_proc
    proc { |obj, *args| obj.send(self, *args) }
  end
end
```

self就是symbol本身,也就是方法(method)

第一个参数obj是接收symbol方法的对象
第二个参数*args是传递给symbol方法的参数列表

怎么用它呢

``` ruby
class Topic
  def title
    "I'm xiaozi"
  end

  def get_title
    yield self
  end
end
t = Topic.new
t.get_title(&:title)

# =>
"I'm xiaozi"
```

如果要使用第二个参数呢

``` ruby
class Topic
  def title(name)
    "I'm #{name}"
  end

  def get_title(name)
    yield self, name
  end
end
t = Topic.new
t.get_title('xiaozi', &:title)

# =>
"I'm xiaozi"
```

其实很简单,就是把借get_title的手去执行t的title方法

来验证下to_proc方法

``` ruby
class Topic
  def title
    "I'm xiaozi"
  end

  def get_title
    yield self
  end
end
t = Topic.new
t.get_title( & proc{ |obj, *args| obj.send(:title, *args) } )
# 再简单一点
t.get_title( & proc{ |obj| obj.send(:title) } )
# 还可以这样,因为&proc的形式跟{}是一样的
t.get_title{ |obj| obj.send(:title) }
# 干脆这样
t.get_title{ |obj| obj.title }

# =>
"I'm xiaozi"
```

现在明白了吧,其实(1..10).inject(&:+)也是这个道理啦

``` ruby
(1..10).inject(&:+)
(1..10).inject(&:+.to_proc)

# can be expressed as
(1..10).inject( &proc{ |obj, *args| obj.send(:+, *args) } ) 

# which & will covert into
(1..10).inject( |obj, *args| obj.send(:+, *args) } ) 

# which we can convert to
(1..10).inject do |obj, *args| 
  obj.send(:+, *args) 
end

# and then we can just rename our parameters to something more friendly
(1..10).inject do |result, element| 
  result.send(:+, element) 
end

# => 55
```

### Proc vs lambda的不同之处

#### return语句

当使用return时,Proc会中途退出,而lambda不会

``` ruby
def awesome_print
  my_proc = lambda { return "I'm xiaozi" }
  result = my_proc.call
  return result + ' hello'  # unreachable code!
end
awesome_print
# =>
=> "I'm xiaozihello"

def awesome_print
  my_proc = Proc.new { return "I'm xiaozi" }
  result = my_proc.call
  return result + ' hello'  # unreachable code!
end
awesome_print
# =>
I'm xiaozi
```

#### 参数检查

lambda会检查参数列表,而Proc不会检查

``` ruby
def awesome_print(statu)
  who = 'xiaozi'
  where = 'shenzhen'
  statu.call
end

awesome_print(Proc.new{|who, where, other| "#{who} live in #{where} #{other.class}"})
# =>
" live in  NilClass"

awesome_print(lambda{|who, where, other| "#{who} live in #{where} #{other.class}"})
# =>
ArgumentError: wrong number of arguments (0 for 3)
```
