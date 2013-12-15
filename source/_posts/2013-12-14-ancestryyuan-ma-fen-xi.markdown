---
layout: post
title: "ancestry源码分析"
date: 2013-12-14 21:12
comments: true
categories: ruby on rails
---

+ [ancestry](https://github.com/stefankroes/ancestry)

### 这个gem是做什么的,原理是什么

这个gem跟[acts_as_tree](/blog/2013/12/09/acts-as-treeyuan-ma-fen-xi/)的作用是一样的,不明白的同学可以看这篇[blog](/blog/2013/12/09/acts-as-treeyuan-ma-fen-xi/).而且在这篇blog我们不直接来分析代码,而是通过使用ancestry的功能来分析代码,并作些比较.由于个人感觉这个gem的代码有点老式,会加上我自己认为可以改进的代码.


### 架构

**数据库表**

{% img /images/ancestry/table.png %}

{% img /images/ancestry/table_content.png %}

在acts_as_tree中只要一个字段来存父与子的关系,然后用`has_many :children`和`belongs_to :parent`,而ancestry也是用一个字段(ancestry),不过它的存取方式跟acts_as_tree是不一样的,它不止存着父的id，还存父的父,也就是说存了祖先的id(62/63),而ancestry_depth存的是层次,类似于祖谱中的辈分,祖先是0(第一代),它的儿子是1(第二代)

在上面的图中,id为62的comment就是root(祖先),它的ancestry是**NULL**,它的ancestry_depth是0

id为64的comment的父为63,63的父为62,所以64的ancestry为62/63

假如在id为64的comment下创建一个子,那这个子的ancestry就为62/63/64

要找64的父也很简单,只要查找一下ancestry这个字段的值,最后一个数字就是了
要找64的祖先也很简单,ancestry的第一个数字就是

ancestry这个gem就是以这种方式来找子和父和存储整个关系的

这种存储方式较act_as_tree的parent_id有更好的先进性

+ **查询速度快**

只存父的id(parent_id)要找祖先的话很麻烦的,只能一层一层往上找,很耗时间

+ **查询和操作功能多**

acts_as_tree只支持几个较简单的查询,查父:parent,查子:children,查祖先:ancestors,查根等

而ancestry支持的查询就很多了,除了上面的,还能查path,children_ids之类的

两者的区别的原因在于实现的方式不一样,acts_as_tree就是通过实例方法来实现的,而ancestry是通过scope
来实现(比较灵活),下面会讲到


**源码文件**

{% img /images/ancestry/source_tree.png %}

只有五个文件

+ **ancestry.rb**: 主程序文件,一些require语句
+ **class_methods.rb**: 类方法的集全
+ **instance_methods.rb**: 实例方法的集合
+ **exceptions.rb**: 自定义的例外(关于报错的)
+ **has_ancestry.rb**: 关于has_ancestry这个方法的源码

PS:我觉得instance_methods.rb和class_methods.rb还有has_ancestry.rb可以合并在一个文件

然后

``` ruby

module Ancestry
  extend ActiveSupport::Concern

  included do
  end

  module ClassMethods
  # class_methods.rb文件的内容可放于此
  end

  module InstanceMethods
  # instance_methods.rb文件的内容可放于此
  end
end

```

或者

``` ruby
module Ancestry
  def self.included(base)
    base.extend ClassMethods
  end

  module ClassMethods
    def has_ancestry
      ...
      include Ancestry::InstanceMethods
    end

  end

  module InstanceMethods
  end
end
```

而has_ancestry的是这样的

``` ruby
  class << ActiveRecord::Base # 继承到ActiveRecord::Base都有has_ancestry这个方法哦
    def has_ancestry options={}
      # Include instance methods
      include Ancestry::InstanceMethods

      # Include dynamic class methods
      extend Ancestry::ClassMethods
      .......
    end
  end
```

### 根据功能来解析代码


#### 创建根(root)

你懂的`root = Comment.create`


{% img /images/ancestry/root.png %}

#### 在根(root)下创建个孩子

`first_children = root.children.create`

{% img /images/ancestry/first_children.png %}

在acts_as_tree也可以通过这样的方式来创建子
但在ancestry中是没有`has_many children`的
猜想一下源码肯定会有一个实例方法叫children(`root.children`)
即使有这个方法也不能create啊
ancestry用了一个方式来实现,正是scope,可以说很多的查询都是基于它

你肯定用过类似`Topic.where(:title => 'xx').order("created_at DESC").limit(4)`这样的方法
就是用的scope

先来看看实例方法**children**

``` ruby instance_methods.rb
def children
  self.base_class.scoped :conditions => child_conditions
end
```

base_class是什么东西呢

``` ruby has_ancestry.rb
class << ActiveRecord::Base
  ...
  # Save self as base class (for STI)
  def ancestry options={}
    # :base_class是个类变量(相当于java中的static变量)
    cattr_accessor :base_class
    # self就是Comment
    self.base_class = self
  end
  ...
end
```

原来就是引用**Comment**自己嘛

scoped是什么呢,看这里[scoped](http://apidock.com/rails/ActiveRecord/NamedScope/ClassMethods/scoped)

child_conditions呢

``` ruby instance_methods.rb
# Children
def child_conditions
  {self.base_class.ancestry_column => child_ancestry}
end
```

ancestry_column默认是ancestry,就是数据库那个列名

``` ruby has_ancestry.rb
# Create ancestry column accessor and set to option or default
cattr_accessor :ancestry_column
self.ancestry_column = options[:ancestry_column] || :ancestry
```

重头戏在**child_ancestry**

``` ruby instance_methods.rb
# The ancestry value for this record's children
def child_ancestry
  # New records cannot have children
  raise Ancestry::AncestryException.new('No child ancestry for new record. Save record before performing tree operations.') if new_record?

  # 这段代码比较长,最佳实践不推荐这样写,应该拆成多行
  if self.send("#{self.base_class.ancestry_column}_was").blank? then id.to_s else "#{self.send "#{self.base_class.ancestry_column}_was"}/#{id}" end
end
```

`"#{self.base_class.ancestry_column}_was"`这是什么东西来的,看这里[Active Model Dirty](http://api.rubyonrails.org/classes/ActiveModel/Dirty.html)

**root**的**ancestry**(root.base_class.ancestry_column)就是**nil**,所以`self.base_class_column => child_ancestry`就是**root**的**id.to_s("72")**

如果不是root呢,那就是`"#{self.send "#{self.base_class.ancestry_column}_was"}/#{id}"`,例如如果是first_children,那就是"72/73"

经过**child_conditions**,ancestry的值被改变了,我们通过scoped链过来的build方法看它的改变,所以

{% img /images/ancestry/root_ancestry_changes.png %}
{% img /images/ancestry/first_child_ancestry_changes.png %}

这个时候save一下就会将数据保存到数据库中了



