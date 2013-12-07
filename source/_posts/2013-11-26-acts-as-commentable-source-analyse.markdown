---
layout: post
title: "Acts As Commentable source analyse"
date: 2013-11-26 11:49
comments: true
categories: ruby on rails
---

+ [acts_as_commentable](https://github.com/jackdempsey/acts_as_commentable)

### 这个gem是做什么的?原理是什么?

这个gem是用来实现评论,它跟一般的评论没什么区别,只是这个gem把一些代码进行了封装,这是我们可以学习的地方.

它使用也很简单,只要在model里写上`acts_as_commentable`,就可以使用评论了,其实`acts_as_commentable`在实现上就等于
`has_many :comments, :as => :commentable`, comment这个model是一个简单的多态,它在数据表中的存储是这样的。

{% img /images/acts_as_comment_table.png %}

其实最主要的就是两个字段commentable_id, commentable_type,一个简单的多态

由于`has_many :comments`那就可以进行很多操作，例如创建一条评论

``` ruby
Post.first.comments.create(:title => "First comment.", :comment => "This is the first comment.")
```

经过这样的操作之后,在数据库中就会产生如下如果

{% img /images/acts_as_comment_table_result.png %}

而这个存储方式是由rails自动处理的,更多的rails关系模型可查看[rails guides](http://guides.rubyonrails.org/association_basics.html)



#### 分析

源码目录下如下

{% img /images/acts_as_commentable_file_tree.png %}

这一步会生成一个rails migration文件和一个叫comment.rb的rails model文件

原理很简单,会在数据库中生成一个叫comments的表
它的字段包括title, comment, commentable_type, commentable_id, user_id等

而生成这两个模版文件的源码如下:

``` ruby lib/generators/comment/comment_generator.rb
require 'rails/generators/migration'

class CommentGenerator < Rails::Generators::Base
  include Rails::Generators::Migration

  def self.source_root
    @_acts_as_commentable_source_root ||= File.expand_path("../templates", __FILE__)
  end

  def self.next_migration_number(path)
    Time.now.utc.strftime("%Y%m%d%H%M%S")
  end

  def create_model_file
    template "comment.rb", "app/models/comment.rb"
    migration_template "create_comments.rb", "db/migrate/create_comments.rb"
  end
end
```

这是使用rails generation功能实现的

<!-- more -->

还可以查看其他gem的实现方式

[acts_as_messageable](https://github.com/LTe/acts-as-messageable/blob/master/lib/generators/acts-as-messageable/migration/migration_generator.rb)

更详细的可参考[rails generation](http://guides.rubyonrails.org/generators.html)

### 如何使用

``` ruby
class Post < ActiveRecord::Base
  acts_as_commentable
end

commentable = Post.create
commentable.comments.create(:title => "First comment.", :comment => "This is the first comment.")
```

#### 分析

如果不使用gem来写评论，一般我们会在model上写has_many comments这样的东西，而acts_as_commentable这个gem它也是要实现这个的，只是它进行包装

关键在`acts_as_commentable`这一行

``` ruby lib/commentable_method.rb
require 'active_record'

# ActsAsCommentable
module Juixe
  module Acts #:nodoc:
    module Commentable #:nodoc:

      def self.included(base)
        # active_record类会extend ClassMethods模块
        base.extend ClassMethods  
      end

      module ClassMethods
        # 由active_record继承的model都有acts_as_commentable这个方法
        def acts_as_commentable(options={})
          # 多态
          has_many :comments, {:as => :commentable, :dependent => :destroy}.merge(options)
          # 具体model实例方法
          include Juixe::Acts::Commentable::InstanceMethods
          # 具体model类的单例方法
          extend Juixe::Acts::Commentable::SingletonMethods
        end
      end
      ...

ActiveRecord::Base.send(:include, Juixe::Acts::Commentable)
```

有这一句`ActiveRecord::Base.send(:include, Juixe::Acts::Commentable)`只要任何继承ActiveRecord::Base的model都可以使用`acts_as_commentable` 它是一个classMethods 

现在我们来分析源码commentable_methods.rb

``` ruby
has_many :comments, {:as => :commentable, :dependent => :destroy}.merge(options)
include Juixe::Acts::Commentable::InstanceMethods
extend Juixe::Acts::Commentable::SingletonMethods
```

第一行是一个activemodel relationship表示model使用多态的comment
`Juixe::Acts::Commentable::SingletonMethods`是类的单例方法,下面的方法可以用Post.find_comments_by_user来调用,例如
Book.find_comments_for(Book.last)

``` ruby
# This module contains class methods
module SingletonMethods
  # Helper method to lookup for comments for a given object.
  # This method is equivalent to obj.comments.
  # 找一个对象的所有comments
  def find_comments_for(obj)
    # 返回类似Topic, Article这样的字符串
    commentable = self.base_class.name
    # find_comments_for_commentable是Comment.rb通过include ActsAsCommentable::Comment引入进来的,它是一个类方法
    Comment.find_comments_for_commentable(commentable, obj.id)
  end

  # Helper class method to lookup comments for
  # the mixin commentable type written by a given user.  
  # This method is NOT equivalent to Comment.find_comments_for_user
  # 通过创建者来找comments
  def find_comments_by_user(user)
    commentable = self.base_class.name
    Comment.where(["user_id = ? and commentable_type = ?", user.id, commentable]).order("created_at DESC")
  end
end
```

Juixe::Acts::Commentable::InstanceMethods是实例方法
可以通过类似这样的方式来调用实例方法
`Book.last.add_comment Comment.create(comment: "second comment")`

接下来我们来看看comment.rb这边

它总要来个belongs_to吧

``` ruby comment.rb
class Comment < ActiveRecord::Base

  # 引入一些Comment可以使用的方法
  include ActsAsCommentable::Comment

  belongs_to :commentable, :polymorphic => true

  default_scope :order => 'created_at ASC'

  # NOTE: install the acts_as_votable plugin if you
  # want user to vote on the quality of comments.
  #acts_as_voteable

  # NOTE: Comments belong to a user
  belongs_to :user
end
```

`belongs_to :commentable, :polymorphic => true`这就是原理
然而`include ActsAsCommentable::Comment`这个会做什么呢

接上来往下看

``` ruby lib/comment_method.rb
module ActsAsCommentable
  # including this module into your Comment model will give you finders and named scopes
  # useful for working with Comments.
  # The named scopes are:
  #   in_order: Returns comments in the order they were created (created_at ASC).
  #   recent: Returns comments by how recently they were created (created_at DESC).
  #   limit(N): Return no more than N comments.
  module Comment

    def self.included(comment_model)
      # 引入的是类方法
      comment_model.extend Finders
      # 引入一些scope
      comment_model.scope :in_order, comment_model.order('created_at ASC')
      comment_model.scope :recent,   comment_model.order('created_at DESC')
    end

    # 下面都是Comment model能使用的类方法
    module Finders
      # Helper class method to lookup all comments assigned
      # to all commentable types for a given user.
      # 通过user来找所有comments
      def find_comments_by_user(user)
        where(["user_id = ?", user.id]).order("created_at DESC")
      end

      # Helper class method to look up all comments for 
      # commentable class name and commentable id.
      # 一个简单的where语句
      def find_comments_for_commentable(commentable_str, commentable_id)
        where(["commentable_type = ? and commentable_id = ?", commentable_str, commentable_id]).order("created_at DESC")
      end

      # Helper class method to look up a commentable object
      # given the commentable class name and id 
      # 找到使用评论的对象
      def find_commentable(commentable_str, commentable_id)
        # 返回类自身
        model = commentable_str.constantize
        model.respond_to?(:find_comments_for) ? model.find(commentable_id) : nil
      end
    end
  end
end
```

可以用Comment.find_comments_by_user的方式来调用

### 总结

它的源码并不复杂，我们通过它并不是要写出跟它一样或类似的gem来，只是可以让我们明白一个道理,代码的重用与组织, 就像在Post.rb里写上acts_as_commentable就可以创建评论了,很方便,还有,通过学习这个gem我们可以学习它是如何设计数据库的

类似的gem还有

- [acts_as_follower](https://github.com/tcocca/acts_as_follower)
- [acts-as-messageable](https://github.com/LTe/acts-as-messageable)
- [acts-as-taggable-on](https://github.com/mbleigh/acts-as-taggable-on)
- [acts_as_paranoid](https://github.com/goncalossilva/acts_as_paranoid)
- [ancestry](https://github.com/stefankroes/ancestry)
- [awesome_nested_set](https://github.com/collectiveidea/awesome_nested_set)

关于acts_as_commentable更详细的用法可查看
[acts-as-commentable-plugin](http://juixe.com/techknow/index.php/2006/06/18/acts-as-commentable-plugin/)
