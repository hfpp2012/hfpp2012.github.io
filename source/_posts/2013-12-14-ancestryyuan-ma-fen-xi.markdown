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

这个gem的版本是1.3.0,关于新版的变化可以看文章的最后部分

### 架构

**数据库表**

{% img /images/ancestry/table.png %}

<!-- more -->

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
# 改了ancestry值了哦,这个时候对象变"脏"了
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
  # 求原始数据,was
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
# 改ancestry值,对象变"脏"(Dirty)
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

# 也就是自己ancestry_depth的值
def depth
  ancestor_ids.size
end

def scope_depth depth_options, depth
  # inject方法很灵活
  depth_options.inject(self.base_class) do |scope, option|
    # relative_depth是传进来的值
    scope_name, relative_depth = option
    if [:before_depth, :to_depth, :at_depth, :from_depth, :after_depth].include? scope_name
      # ancestry_depth的值加上传进来的值
      scope.send scope_name, depth + relative_depth
    else
      raise Ancestry::AncestryException.new("Unknown depth option: #{scope_name}.")
    end
  end
end

scope_method = if rails_3 then :scope else :named_scope end
# 很多方法用到这个,在rails 3中会变成send :scope, :ordered_by_ancestry...
# 如果是根会排在前面
send scope_method, :ordered_by_ancestry, :order => "(case when #{table_name}.#{ancestry_column} is null then 0 else 1 end), #{table_name}.#{ancestry_column}"
```

#### 其他几个查询方法

主要是**subtree**和**root**

``` ruby
# Subtree
# 包括自己查后代,使用self.id, child_ancestry是查自己的子,"#{child_ancestry}/"查孙一代起
def subtree_conditions
  ["#{self.base_class.table_name}.#{self.base_class.primary_key} = ? or #{self.base_class.table_name}.#{self.base_class.ancestry_column} like ? or #{self.base_class.table_name}.#{self.base_class.ancestry_column} = ?", self.id, "#{child_ancestry}/%", child_ancestry]
end

def subtree depth_options = {}
  self.base_class.ordered_by_ancestry.scope_depth(depth_options, depth).scoped :conditions => subtree_conditions
end

def subtree_ids depth_options = {}
  subtree(depth_options).all(:select => self.base_class.primary_key).collect(&self.base_class.primary_key.to_sym)
end

# Root
# 返回根的id,在acts_as_tree可是要遍历的,这个就简单多了
# ancestor_ids是空的,那就返回自己的id
def root_id
  if ancestor_ids.empty? then id else ancestor_ids.first end
end

# 如果根就是自己,那就不用查了
def root
  if root_id == id then self else self.base_class.find(root_id) end
end

# 判断是不是nil而已
def is_root?
  read_attribute(self.base_class.ancestry_column).blank?
end
```

#### 将第三代c的父改为根(root),变成了第二代

第三代要变成第二代,把ancestry改为第一代的id,ancestry_depth从2变成1即可

**但是**第三代也有后代,而且可能有很多(第四代,第五代...),他们怎么办

改变前:

{% img /images/ancestry/before_change_parent.png %}

{% img /images/ancestry/change_parent.png %}

改变后:

{% img /images/ancestry/after_change_parent.png %}

先来看看ancestry的变化

他的后代的ancestry的变化是通过**before_save**来实现的

在**before_save**之前c(77)的ancestry最先变化,它由"72/73"变成了"72"

看上面那个命令图,第一个改的后代是id为78(c的子)那个
它由"72/73/77"变成"72/77"

id为79(c的孙)那个Comment的ancestry由"72/73/77/78"变成"72/77/78"

看出一个规律就是把"72/73"变成"72"罢了,其他的不变,因为c的ancestry由"72/73"变成了"72",所以它的后代跟着变

如果上面推测正确,那么...

``` ruby has_ancestry.rb
# Update descendants with new ancestry before save
before_save :update_descendants_with_new_ancestry
```

``` ruby instance_methods.rb
# Descendants
# 查的是所有后代,下面的update_descendants_with_new_ancestry会用到
def descendant_conditions
  ["#{self.base_class.table_name}.#{self.base_class.ancestry_column} like ? or #{self.base_class.table_name}.#{self.base_class.ancestry_column} = ?", "#{child_ancestry}/%", child_ancestry]
end

def descendants depth_options = {}
  self.base_class.ordered_by_ancestry.scope_depth(depth_options, depth).scoped :conditions => descendant_conditions
end

# 在外面包装一个scope
def unscoped_descendants
  # with_exclusive_scope就是with_scope,不明白的可以看http://apidock.com/rails/ActiveRecord/Base/with_scope/class
  self.base_class.send(:with_exclusive_scope) do
    self.base_class.all(:conditions => descendant_conditions)
  end
end

# Update descendants with new ancestry
def update_descendants_with_new_ancestry
  # Skip this if callbacks are disabled
  # ancestry_callbacks_disabled?是一个开关,为了让下面的更改有序的进行而不发生数据更改的碰撞,可以看下面相关的代码
  unless ancestry_callbacks_disabled?
    # If node is valid, not a new record and ancestry was updated ...
    # ancestry_column肯定先改变啦,那就那个c的值啦,!new_reocrd?也肯定不是新数据啦,必须要是有效的valid?(验证通过)
    if changed.include?(self.base_class.ancestry_column.to_s) && !new_record? && valid?
      # ... for each descendant ...
      # 会去循环每个后代哦
      unscoped_descendants.each do |descendant|
        # ... replace old ancestry with new ancestry
        # 进行有without_ancestry_callbacks的代码,里面包装的是一个原子操作,防止数据被错误修改
        descendant.without_ancestry_callbacks do
          # 更改ancestry的值哦
          descendant.update_attribute(
            # 就是ancestry那个列嘛
            self.base_class.ancestry_column,
            # 读取每个后代ancestry的值,gsub进行全局替换
            descendant.read_attribute(descendant.class.ancestry_column).gsub(
              # child_ancestry是没变之前的ancestry的值,例如上例中的c的ancestry值"72/73"
              /^#{self.child_ancestry}/,
              # self就是c,替换成c现有的值,也就是"72"
              if read_attribute(self.class.ancestry_column).blank? then id.to_s else "#{read_attribute self.class.ancestry_column }/#{id}" end
            )
          )
        end
      end
    end
  end
end

# 开关控制
# Callback disabling
def without_ancestry_callbacks
  @disable_ancestry_callbacks = true
  yield
  @disable_ancestry_callbacks = false
end

def ancestry_callbacks_disabled?
  !!@disable_ancestry_callbacks
end
```

代码果然如我们预期的效果一样

接下来我们来看看ancestry_depth的变化

说到这个必须谈下`has_ancestry :cache_depth => true`

`:cache_depth => true`开启了之后就可以在**ancestry_depth**(默认)存树的深度,root为0,第一代为1

``` ruby has_ancestry.rb
# Create ancestry column accessor and set to option or default
if options[:cache_depth]
  ...
  # Cache depth in depth cache column before save
  # 在验证之前会执行这个的
  before_validation :cache_depth
  ...
end
```

:cache_depth到底做了什么事儿

``` ruby
# 求深度,例如,ancestry为'72/73'的深度就是'2'(第三代)
def depth
  ancestor_ids.size
end

# write_attribute是save(false)的,更改ancestry_depth的值
def cache_depth
  write_attribute self.base_class.depth_cache_column, depth
end
```

在改descendant时根本没有改它的**ancestry_depth**,所以descendant的ancestry_depth没有即时更新,跟我们预期的结果不一样

只要重载**update_descendants_with_new_ancestry**方法

加上下面的代码就可以解决那个问题了

``` ruby
def update_descendants_with_new_ancestry
  ...
  descendant.update_attribute(
    self.base_class.ancestry_column,
    descendant.read_attribute(descendant.class.ancestry_column).gsub(
      /^#{self.child_ancestry}/,
      if read_attribute(self.class.ancestry_column).blank? then id.to_s else "#{read_attribute self.class.ancestry_column }/#{id}" end
    )
  )
  # 加入下面的代码哦
  descendant.reload
  descendant.update_attribute(
    self.base_class.depth_cache_column,
    descendant.depth
  )
  ...
end
```

#### 删除c(有后代)

删除一个Comment很简单,只要把那条记录删除就行了,但是删除后它的后代怎么办

ancestry是通过:orphan_strategy这个参数来控制,而has_ancestry.rb有这句`before_destroy :apply_orphan_strategy`

:orphan_strategy只有三个值: :rootify, :destroy, :restrict

先看下:rootify

改变前:

{% img /images/ancestry/destroy_rootify.png %}

改变后:

{% img /images/ancestry/after_destroy_rootify.png %}

c被删除后,它的子(81)没有了父,就把ancestry设为NULL,它自己的子(82)的ancestry还是指向自己(81),依此类推...

其实做的就是把id为82的comment的ancestry:"72/73/80/81"的"72/73/80"(81的ancestry,也就是c.child_ancestry)删除掉

:destroy就不做这些改变了,直接删除数据

:restrict,只要有后代就抛出异常

来看下代码

``` ruby
# Apply orphan strategy
def apply_orphan_strategy
  # Skip this if callbacks are disabled
  unless ancestry_callbacks_disabled?
    # If this isn't a new record ...
    unless new_record?
      # ... make all children root if orphan strategy is rootify
      if self.base_class.orphan_strategy == :rootify
        unscoped_descendants.each do |descendant|
          descendant.without_ancestry_callbacks do
            # descendant.ancestry == child_ancestry就是c的子,直接设为NULL
            # 从c的"孙"开始,把ancestry的值前面一部分替换掉,也就是子的ancestry,所有孙之后的后代的这部分都替换成空
            descendant.update_attribute descendant.class.ancestry_column, (if descendant.ancestry == child_ancestry then nil else descendant.ancestry.gsub(/^#{child_ancestry}\//, '') end)
          end
        end
      # ... destroy all descendants if orphan strategy is destroy
      elsif self.base_class.orphan_strategy == :destroy
        unscoped_descendants.each do |descendant|
          descendant.without_ancestry_callbacks do
            # 直接删除后代
            descendant.destroy
          end
        end
      # ... throw an exception if it has children and orphan strategy is restrict
      elsif self.base_class.orphan_strategy == :restrict
        # 抛出异常
        raise Ancestry::AncestryException.new('Cannot delete record because it has descendants.') unless is_childless?
      end
    end
  end
end
```

### 总结

1. 通过这个gem我们可以学习比较先进的数据库设计的理念,就是减少数据库的查询次数
2. 学习scope的灵活用法
3. 高效查询算法的设计

----------------------------------------------------------------------------

### 关于新版本的改变

+ 去掉了primary_key_format这个参数(没什么用处)

+ 用上了reorder这个scope

``` ruby
-  send scope_method, :ordered_by_ancestry, :order => "(case when #{table_name}.#{ancestry_column} is null then 0 else 1 end), #{table_name}.#{ancestry_column}"
-  send scope_method, :ordered_by_ancestry_and, lambda { |order| {:order => "(case when #{table_name}.#{ancestry_column} is null then 0 else 1 end), #{table_name}.#{ancestry_column}, #{order}"} }
+  send scope_method, :ordered_by_ancestry, reorder("(case when #{table_name}.#{ancestry_column} is null then 0 else 1 end), #{table_name}.#{ancestry_column}")
+  send scope_method, :ordered_by_ancestry_and, lambda { |order| reorder("(case when #{table_name}.#{ancestry_column} is null then 0 else 1 end), #{table_name}.#{ancestry_column}, #{order}") }
```

+ 不用with_exclusive_scope,用unscoped代替

+ `send scope`用`scope`代替

+ /\A[0-9]+(\/[0-9]+)*\Z/这个用常量替换

``` ruby
-  validates_format_of ancestry_column, :with => /\A[0-9]+(\/[0-9]+)*\Z/, :allow_nil => true
+  validates_format_of ancestry_column, :with => Ancestry::ANCESTRY_PATTERN, :allow_nil => true
```

+ 更改before_save那个方法

``` ruby
-  if changed.include?(self.base_class.ancestry_column.to_s) && !new_record? && valid?
+  if changed.include?(self.base_class.ancestry_column.to_s) && !new_record? && sane_ancestry?
def sane_ancestry?
  ancestry.nil? || (ancestry.to_s =~ Ancestry::ANCESTRY_PATTERN && !ancestor_ids.include?(self.id))
end
```

+ find用unscoped包住

``` ruby
-  self.parent = if parent_id.blank? then nil else self.base_class.find(parent_id) end
+  self.parent = if parent_id.blank? then nil else unscoped_find(parent_id) end
def unscoped_find id
  self.base_class.unscoped { self.base_class.find(id) }
end
```

+ conditions用where代替

``` ruby
-  self.base_class.scoped :conditions => child_conditions
+  self.base_class.where child_conditions
```

+ 用上了select方法

``` ruby
-  children.all(:select => self.base_class.primary_key).map(&self.base_class.primary_key.to_sym)
+  children.select(self.base_class.primary_key).map(&self.base_class.primary_key.to_sym)
```

+ nil?代替blank?

``` ruby
-  write_attribute(self.base_class.ancestry_column, if parent.blank? then nil else parent.child_ancestry end)
+  write_attribute(self.base_class.ancestry_column, if parent.nil? then nil else parent.child_ancestry end)
```

+ 用in查询

``` ruby
def ancestor_conditions
  t = get_arel_table
  t[get_primary_key_column].in(ancestor_ids)
end

def get_arel_table
  self.ancestry_base_class.arel_table
end
```


