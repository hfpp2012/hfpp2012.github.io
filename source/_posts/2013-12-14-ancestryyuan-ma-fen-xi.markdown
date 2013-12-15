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

看一下几个关于children的方法

``` ruby instance_methods.rb
# 查找所有孩子的id
def child_ids
  # 在rails 3或4中可以用children.pluck(:id)
  children.all(:select => self.base_class.primary_key).map(&self.base_class.primary_key.to_sym)
end

# 判断是否有孩子
def has_children?
  self.children.exists?({})
end

# 是否叶子节点(没有孩子)
def is_childless?
  !has_children?
end
```

#### 在first_children创建一个siblings(兄弟)节点

{% img /images/ancestry/siblings_changes.png %}

``` ruby instance_methods.rb
# Siblings
# 变成跟自己(first_children)一样的ancestry
def sibling_conditions
  {self.base_class.ancestry_column => read_attribute(self.base_class.ancestry_column)}
end

# 返回所有的兄弟节点,是包括自己的(first_children),因为是根据ancestry(父)来查的
def siblings
  self.base_class.scoped :conditions => sibling_conditions
end

# 返回所有的兄弟节点的id
def sibling_ids
   siblings.all(:select => self.base_class.primary_key).collect(&self.base_class.primary_key.to_sym)
end

def has_siblings?
  self.siblings.count > 1
end

def is_only_child?
  !has_siblings?
end
```

如果不让siblings包含自己,只要`first_children.siblings - [first_children]`就行了

#### 指定父(parent)节点创建一个Comment

在comment.rb的白名单上加 ```attr_accessible :parent```

{% img /images/ancestry/create_with_parent.png %}

既然可以指定**parent**,那一定有`parent=()`这个方法

``` ruby instance_methods.rb
# Parent
# 找到父那个对象再去找父的child_ancestry,即假如父为根就是它的id...
def parent= parent
  write_attribute(self.base_class.ancestry_column, if parent.blank? then nil else parent.child_ancestry end)
end

# 还是会执行paren= parent方法
def parent_id= parent_id
  self.parent = if parent_id.blank? then nil else self.base_class.find(parent_id) end
end

# 返回祖先链的最后一个id,即父的id,ancestors下面会讲到
def parent_id
  if ancestor_ids.empty? then nil else ancestor_ids.last end
end

# 用id找到那个parent,找不到就是nil
def parent
  if parent_id.blank? then nil else self.base_class.find(parent_id) end
end
```

#### 查找刚创建的第三代的祖先链

{% img /images/ancestry/ancestor_search.png %}

``` ruby instance_methods.rb
# Ancestors
# 读取ancestry的值split后组成数组,取变量的值没有查数据库
def ancestor_ids
  read_attribute(self.base_class.ancestry_column).to_s.split('/').map { |id| cast_primary_key(id) }
end

# 把ancestor_ids数组传给id
def ancestor_conditions
  {self.base_class.primary_key => ancestor_ids}
end

def ancestors depth_options = {}
  self.base_class.scope_depth(depth_options, depth).ordered_by_ancestry.scoped :conditions => ancestor_conditions
end

def path_ids
  ancestor_ids + [id]
end

def path_conditions
  {self.base_class.primary_key => path_ids}
end

def path depth_options = {}
  self.base_class.scope_depth(depth_options, depth).ordered_by_ancestry.scoped :conditions => path_conditions
end
```

**scope_depth**是个类方法,看它之前先来看看架构那部分那里,表据表除了有个**ancestry**字段,还有**ancestry_depth**这个字段

这个**ancestry_depth**就是类变量**depth_cache_column**定义的

``` ruby has_ancestry.rb
# Create ancestry column accessor and set to option or default
if options[:cache_depth]
  # Create accessor for column name and set to option or default
  self.cattr_accessor :depth_cache_column
  self.depth_cache_column = options[:depth_cache_column] || :ancestry_depth

  # Cache depth in depth cache column before save
  # 验证depth_cache_column必须是整数且大于或等于0
  before_validation :cache_depth

  # Validate depth column
  validates_numericality_of depth_cache_column, :greater_than_or_equal_to => 0, :only_integer => true, :allow_nil => false
end

# Create named scopes for depth
# 下面就是一个条件,跟ancestry_depth比,例如before_depth就是'"ancestry_depth < ?", 3'之类的
{:before_depth => '<', :to_depth => '<=', :at_depth => '=', :from_depth => '>=', :after_depth => '>'}.each do |scope_name, operator|
  # send scope_method就是定义一个scope
  send scope_method, scope_name, lambda { |depth|
    raise Ancestry::AncestryException.new("Named scope '#{scope_name}' is only available when depth caching is enabled.") unless options[:cache_depth]
    {:conditions => ["#{depth_cache_column} #{operator} ?", depth]}
  }
end
```
{% img /images/ancestry/scope_depth.png %}

为什么数字会改变呢,原来那个**before_depth**不是基于**root**(根),而是基于**c**(孙),**c**有两个前辈1+1=2,所以数据库查的是2+2=4

``` ruby class_methods.rb解读scope_depth
def ancestors depth_options = {}
  self.base_class.scope_depth(depth_options, depth).ordered_by_ancestry.scoped :conditions => ancestor_conditions
end

def depth
  ancestor_ids.size
end

def scope_depth depth_options, depth
  depth_options.inject(self.base_class) do |scope, option|
    scope_name, relative_depth = option
    if [:before_depth, :to_depth, :at_depth, :from_depth, :after_depth].include? scope_name
      scope.send scope_name, depth + relative_depth
    else
      raise Ancestry::AncestryException.new("Unknown depth option: #{scope_name}.")
    end
  end
end
```
