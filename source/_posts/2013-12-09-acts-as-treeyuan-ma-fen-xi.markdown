---
layout: post
title: "acts_as_tree源码分析"
date: 2013-12-09 14:30
comments: true
categories: ruby on rails
---

+ [acts_as_tree](https://github.com/amerine/acts_as_tree)

### 这个gem是干什么的?原理是什么?

这个gem是用来实现树型结构的,例如一个分类(category),下面还有子分类,这个时候就可以用它了,还有,你回复了一个贴子,别人再来回复你，这个时候也可以用它。一个菜单之下还有子菜单，子菜单之下又可以有子菜单,子菜单之下有菜单项，它可以实现一种树型结构的菜单。它的原理也很简单,在模型中是实现了一个自关联,假如在一个comment(评论)model中,定义一个children(孩子),再定义一个parent(父), :class_name都是'Comment'。那他们怎么实现关联的,也就是说孩子要知道父亲是谁,父亲要知道它有哪些孩子,它在数据存储中是通过一个字段来关联的,在这个gem中这个字段叫**parent_id**假如我们创建一个根(root), `root = Comment.create`, 这个时候**root***的parent_id是null,它是一个根,`children = root.children.create`,这个**children**是**root**的一个孩子,那**children**的**parent_id**就是**root**的**id**

```
  菜单
   |_ 系统设置
   |    |_ 模版设置
   |    |_ 表格设置
   |_ 人员管理
        |_ 教职工管理
        |_ 学员管理
```

children和parent的关系是这样实现的

``` ruby
belongs_to :parent...
has_many   :children...
```

数据表是这样存储的

<!-- more -->

{% img /images/acts_as_tree_table.png %}

### 分析

这个gem的版本是1.4.0

它的核心文件只有一个**lib/acts_as_tree.rb**

``` ruby lib/acts_as_tree.rb

require 'acts_as_tree/version'

module ActsAsTree

  if defined? Rails::Railtie
    require 'acts_as_tree/railtie'
  elsif defined? Rails::Initializer
    raise "acts_as_tree 1.0 is not compatible with Rails 2.3 or older"
  end

  def self.included(base)
    base.extend(ClassMethods)
  end

  # Specify this +acts_as+ extension if you want to model a tree structure
  # by providing a parent association and a children association. This
  # requires that you have a foreign key column, which by default is called
  # +parent_id+.
  #
  #   class Category < ActiveRecord::Base
  #     include ActsAsTree
  #
  #     acts_as_tree :order => "name"
  #   end
  #
  #   Example:
  #   root
  #    \_ child1
  #         \_ subchild1
  #         \_ subchild2
  #
  #   root      = Category.create("name" => "root")
  #   child1    = root.children.create("name" => "child1")
  #   subchild1 = child1.children.create("name" => "subchild1")
  #
  #   root.parent   # => nil
  #   child1.parent # => root
  #   root.children # => [child1]
  #   root.children.first.children.first # => subchild1
  #
  # In addition to the parent and children associations, the following
  # instance methods are added to the class after calling
  # <tt>acts_as_tree</tt>:
  # * <tt>siblings</tt> - Returns all the children of the parent, excluding
  #                       the current node (<tt>[subchild2]</tt> when called
  #                       on <tt>subchild1</tt>)
  # * <tt>self_and_siblings</tt> - Returns all the children of the parent,
  #                                including the current node (<tt>[subchild1, subchild2]</tt>
  #                                when called on <tt>subchild1</tt>)
  # * <tt>ancestors</tt> - Returns all the ancestors of the current node
  #                        (<tt>[child1, root]</tt> when called on <tt>subchild2</tt>)
  # * <tt>root</tt> - Returns the root of the current node (<tt>root</tt>
  #                   when called on <tt>subchild2</tt>)
  module ClassMethods
    # Configuration options are:
    #
    # * <tt>foreign_key</tt> - specifies the column name to use for tracking
    #                          of the tree (default: +parent_id+)
    # * <tt>order</tt> - makes it possible to sort the children according to
    #                    this SQL snippet.
    # * <tt>counter_cache</tt> - keeps a count in a +children_count+ column
    #                            if set to +true+ (default: +false+).

    # 是通过这个方法来明确rails model association的(has_many belongs_to),而且这个方法会导入InstanceMethods model
    def acts_as_tree(options = {})

      # 默认的配置选项, dependent当你删除节点时，连同子节点全部删除
      configuration = {
        foreign_key:   "parent_id",
        order:         nil,
        counter_cache: nil,
        dependent:     :destroy
      }

      # 替换原来的方法或增加新的option
      configuration.update(options) if options.is_a?(Hash)

      # 定义一个自关联的parent, 如果设count_cache: true那就得在数据表中加入comments_count(comments是你的表名), foreign_key是parent_id，主要是通过它来关联parent和children
      belongs_to :parent, class_name:    name,
        foreign_key:   configuration[:foreign_key],
        counter_cache: configuration[:counter_cache],
        inverse_of:    :children

      # 跟上述差不多,更多的可以看rails guides associations
      if ActiveRecord::VERSION::MAJOR >= 4
        has_many :children, lambda { order configuration[:order] },
          class_name:  name,
          foreign_key: configuration[:foreign_key],
          dependent:   configuration[:dependent],
          inverse_of:  :parent
      else
        has_many :children, class_name:  name,
          foreign_key: configuration[:foreign_key],
          order:       configuration[:order],
          dependent:   configuration[:dependent],
          inverse_of:  :parent
      end

      # 下面都是类的实例方法,相当于class << self
      class_eval <<-EOV
        # 包含实例方法
        include ActsAsTree::InstanceMethods

        # update之后更新counter_cache, 关于create之后更新counter_cache是自动进行的
        after_update :update_parents_counter_cache

        # 以一个数组形式返回所有的根,可通过Comment.roots这样调用
        def self.roots
          # %Q的用法参考这里http://simpleror.wordpress.com/2009/03/15/q-q-w-w-x-r-s/
          # fetch是一个Hash的方法,对传过来的值作为key来取值,没有这个key就用第二个参数来代替
          # 在老版本的acts_as_tree中用#{configuration[:order].nil? ? "nil" : %Q{"#{configuration[:order]}"}},明显用fetch简洁好多
          order_option = %Q{#{configuration.fetch :order, "nil"}}
          # nil查询在老版本的acts_as_tree中用is NULL
          where(:#{configuration[:foreign_key]} => nil).order(order_option)
        end

        # 返回数据中的根
        def self.root
          order_option = %Q{#{configuration.fetch :order, "nil"}}
          self.roots.first
        end
      EOV
    end

  end

  # extend Presentation,这个module下定义的都是类方法,用Comment.tree_view来调用
  module Presentation
    # show records in a tree view
    # Example:
    # root
    #  |_ child1
    #  |    |_ subchild1
    #  |    |_ subchild2
    #  |_ child2
    #       |_ subchild3
    #       |_ subchild4
    #
    # 看上面的例子，用一种良好阅读的方式返回树的形状
    # 树的遍历要用到递归
    def tree_view(label_method = :to_s,  node = nil, level = -1)
      if node.nil?
        puts "root"
        # roots是个类方法,返回所有包含根节点的数组
        nodes = roots
      else
        label = "|_ #{node.send(label_method)}"
        if level == 0
          puts " #{label}"
        else
          puts " |#{"    "*level}#{label}"
        end
        nodes = node.children
      end
      # 1.首先默认传入node=nil, level=-1,这个时候会执行node.nil?，在第一行输出root,接着返回包含根节点的数组
      # 2.遍历由根节点组成的数组,执行tree_view递归函数,而这个时候,node不等于nil,level=0,执行level == 0下面一句输出第一个根节点(类似|_root)
      # 3.接着把这个根节点的孩子传给nodes去执行第二步, 这个node是不会等于nil的,level也将大于1,将会执行puts " |#{"    "*level}#{label}"这一行,在第二层树中level等于1,在输出时前面会空出一份空白,假如第二层树中有孩子,还会继续遍历,直到没有孩子
      nodes.each do |child|
        tree_view(label_method, child, level+1)
      end
    end

  end

  # 下面定义都是实例方法
  module InstanceMethods
    # Returns list of ancestors, starting from parent until root.
    #
    #   subchild1.ancestors # => [child1, root]
    # 返回祖先链
    def ancestors
      node, nodes = self, []
      nodes << node = node.parent while node.parent
      nodes
    end

    # Returns the root node of the tree.
    # 返回根节点
    def root
      node = self
      node = node.parent while node.parent
      node
    end

    # Returns all siblings of the current node.
    #
    #   subchild1.siblings # => [subchild2]
    # 返回所有兄弟节点
    def siblings
      # 减掉自己就是兄弟节点
      self_and_siblings - [self]
    end

    # Returns all siblings and a reference to the current node.
    #
    #   subchild1.self_and_siblings # => [subchild1, subchild2]
    # 返回自身加上兄弟节点
    def self_and_siblings
      # 如果不是根节点就把根节点的所有孩子都返回
      # 否则的话就把所有根节点返回
      # class能引用到自身的类
      parent ? parent.children : self.class.roots
    end

    # Returns children (without subchildren) and current node itself.
    #
    #   root.self_and_children # => [root, child1]
    # 返回自身和孩子
    def self_and_children
      [self] + self.children
    end

    # Returns ancestors and current node itself.
    #
    #   subchild1.self_and_ancestors # => [subchild1, child1, root]
    # 返回自己和祖先
    def self_and_ancestors
      [self] + self.ancestors
    end

    # Returns true if node has no parent, false otherwise
    #
    #   subchild1.root? # => false
    #   root.root?      # => true
    # 判断是否是根节点
    def root?
      parent.nil?
    end

    # Returns true if node has no children, false otherwise
    #
    #   subchild1.leaf? # => true
    #   child1.leaf?    # => false
    # 没有孩子了就是叶子节点
    def leaf?
      children.count == 0
    end

    private

    # 为使这个方法生效必须有:counter_cache => :children_count和children_count这个字段,不然你自己可以重写这个方法, :counter_cache是可以自定义的,默认是children的表名加cache，例如comments_cache
    def update_parents_counter_cache
      # 首先用respond_to?判断是否有这个方法(:children_count)
      # 通过Active Model Dirty来判断是否改变了parent_id
      # 关于Active Model Dirty可以看http://api.rubyonrails.org/classes/ActiveModel/Dirty.html
      if self.respond_to?(:children_count) && parent_id_changed?
        # decrement_counter是给:children_count减少1, parent_id_was是旧的值
        self.class.decrement_counter(:children_count, parent_id_was)
        # increment_counter是给:children_count增加1, parent_id是新的值
        # 具体的decrement_counter和increment_counter可以看http://api.rubyonrails.org/classes/ActiveRecord/CounterCache/ClassMethods.html
        self.class.increment_counter(:children_count, parent_id)
      end
    end
  end
end

# Deprecating the following code in the future.
require 'acts_as_tree/active_record/acts/tree'

```

### 总结

通过这个gem我们可以学一些查询方法,例如关于根节点，祖先节点的查找啊,还可以学习自关联的写法,一些递归写法等
