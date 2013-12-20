---
layout: post
title: "acts_as_paranoid源码分析"
date: 2013-12-20 21:57
comments: true
categories: ruby on rails
---

+ [acts_as_paranoid](https://github.com/goncalossilva/acts_as_paranoid)

它的作用就是假删除,在实际中还是会很有用的。有一天,客户说我刚才误删了一个东西要把它找回来,这个时候它就派上用场了。

其实它实际上不删除数据中的数据,只不过是隐藏起来而已,只要让用户看不到,它就等于删除了,实际上,要还原的话修改一下数据库就可以回来了

它实现的原理很简单,只不过是用一个标志来实现隐藏数据,在数据表中加一个字段,把它的值改一下,它就删除了(隐藏),修改回来,它又出现了

什么字段呢,acts_as_paranoid默认情况下用**"deleted_at"**,用一个参数**column**来指定,它可以有三种类型boolean, string, time,这三个参数用**column_type**参数来指定

boolean:布尔型,被删除时值为true
string:字符串型,被删除时值为"deleted",这个值可以用deleted_value参数来指定
time:时间型,被删除时值为当前时间(删除操作的时间)

以上三个类型未删除时值都为NULL(nil)。建议使用time类型

使用举例如下:

``` ruby
  class School < ActiveRecord::Base
    acts_as_paranoid :column_type => :boolean
    或
    acts_as_paranoid :column => "is_deleted", :column_type => :string, :deleted_value => "deleted"
    ...
  end
```


