---
layout: post
title: "acts_as_follower源码分析"
date: 2013-12-03 16:41
comments: true
categories: ruby on rails
---

+ [acts_as_follower](https://github.com/tcocca/acts_as_follower)

### 这个gem是做什么的?原理是什么?

这个gem主要是用来实现类似twitter那种关注，新浪那种收听的功能
主要是两个model的关联,例如一个用户订阅了一本书,我们可以这样实现`User.first.follow(Book.first)`

它在实现是就是利用两个多态，一个叫followable(被follow者), 一个叫followable(跟随者),把每个对象的类名字符串和id存进数据库实现关联,其他代码就实现了关于两者的查询代码, 下面是两者的关系

``` ruby
has_many :followings, :as => :followable, :dependent => :destroy, :class_name => 'Follow'
has_many :follows, :as => :follower, :dependent => :destroy
```

``` ruby follow.rb
belongs_to :followable, :polymorphic => true
belongs_to :follower,   :polymorphic => true
```

-----------------------------------------

### 来看一下它的数据结构

目录树结构

{% img /images/acts_as_follower_file_tree.png %}

结构库表结构

{% img /images/acts_as_follower_table_tree.png %}

主要有五个字段,分别是**blocked**, **followable_id**, **followable_type**, **follower_id**, **follower_type**, 主要是分为两个多态(**follower**, **followerable**), 例如一个用户订阅了一本书, **followerable**可以存**Book**和它的**id**, 而**follower**这个多态存**User**和它的**id**


-------------------

### 接下来就来分晰啦

#### 源码目录结构功能介绍

<!-- more -->
+ lib/acts_as_follower.rb: 主文件,主要定义了ActsAsFollower这个model(每个gem都会做差不多的事),还包括一些autoload语句，做的事主要是require你的所用到的rb文件啦

``` ruby lib/acts_as_follower
require "acts_as_follower/version"

module ActsAsFollower
  autoload :Follower,     'acts_as_follower/follower'
  autoload :Followable,   'acts_as_follower/followable'
  autoload :FollowerLib,  'acts_as_follower/follower_lib'
  autoload :FollowScopes, 'acts_as_follower/follow_scopes'

  require 'acts_as_follower/railtie' if defined?(Rails) && Rails::VERSION::MAJOR >= 3
end
```

+ lib/acts_as_follower/railtie.rb: 这个文件的作用就大了,在model(继承自**ActiveRecord::Base**)能使用**acts_as_followable**,**acts_as_follower**这个多亏了这个文件,其实这个文件使用了一个叫railtie的rails部件,它官方的定义是这样的:**Railtie is the core of the Rails framework and provides several hooks to extend Rails and/or modify the initialization process**, 它能修改一些启动信息,在加载ActiveRecord部件时执行`include ActsAsFollower::Follower`这样的语句,以后每个继承自ActiveRecord::Base的class都能使用ActsAsFollower::Follower下面定义的方法(而acts_as_follower就是在这个文件里定义的)，其实可以用类似这样的写法`ActiveRecord::Base.send(:include, Juixe::Acts::Commentable)`来实现相同的目的,关于rails railtie的详细信息可查看[rails railtie](http://api.rubyonrails.org/classes/Rails/Railtie.html)和[Rails3: Railtie 和 Plugins 系統](http://ihower.tw/blog/archives/4873)

``` ruby lib/acts_as_follower/railtie.rb
require 'acts_as_follower'
require 'rails'

module ActsAsFollower
  class Railtie < Rails::Railtie

    initializer "acts_as_follower.active_record" do |app|
      ActiveSupport.on_load :active_record do
        include ActsAsFollower::Follower
        include ActsAsFollower::Followable
      end
    end

  end
end
```

+ lib/acts_as_follower/followable.rb: 这个文件主要服务于被follow的对象,它定义了好多用于被**follow**的实例方法,包括**followers_by_type**, **followers_count**,它是这样来调用的,例如`Book.first.followers_count`, `Book.first.followers_by_type('User')`

``` ruby lib/acts_as_follower/followable.rb
# 被follow
module ActsAsFollower #:nodoc:

  # 由上面可知ActiveRecord会include下面的Followable
  module Followable

    # 这种写法很常见
    def self.included(base)
      base.extend ClassMethods
    end

    # ClassMethods下面定义的是实例方法,ActiveRecord::Base可直接用，也就是说可以直接在继承自ActiveRecord::Base的model下用
    module ClassMethods
      def acts_as_followable
        has_many :followings, :as => :followable, :dependent => :destroy, :class_name => 'Follow'
        # 下面的实例方法
        include ActsAsFollower::Followable::InstanceMethods
        # 这个下文有说
        include ActsAsFollower::FollowerLib
      end
    end

    # 下面的都是实例方法可用于被follow的实例来调用,下面的self都是被follow对象的实例
    module InstanceMethods

      # Returns the number of followers a record has.
      # 返回followins的数量,而unblocked在lib/acts_as_followable/follow_scope.rb定义,只是一个简单的where scope
      def followers_count
        self.followings.unblocked.count
      end

      # Returns the followers by a given type
      # 下面主要是一个查询语句,constantize的定义见http://api.rubyonrails.org/classes/ActiveSupport/Inflector.html
      # 而parent_class_name的定义在lib/acts_as_followable/follower_lib.rb,很简单,主要是返回class名字
      def followers_by_type(follower_type, options={})
        # parent_class_name是把一个实例转成类的字符串,例如'Topic', 'Article'等
        follows = follower_type.constantize.
          joins(:follows).
          where('follows.blocked'         => false,
                'follows.followable_id'   => self.id,
                'follows.followable_type' => parent_class_name(self),
                'follows.follower_type'   => follower_type)
        if options.has_key?(:limit)
          follows = follows.limit(options[:limit])
        end
        if options.has_key?(:includes)
          follows = follows.includes(options[:includes])
        end
        follows
      end

      # 返回所有follower的数量
      def followers_by_type_count(follower_type)
        self.followings.unblocked.for_follower_type(follower_type).count
      end

      # Allows magic names on followers_by_type
      # e.g. user_followers == followers_by_type('User')
      # Allows magic names on followers_by_type_count
      # e.g. count_user_followers == followers_by_type_count('User')
      # 实现更加灵活的查询
      def method_missing(m, *args)
        if m.to_s[/count_(.+)_followers/]
          followers_by_type_count($1.singularize.classify)
        elsif m.to_s[/(.+)_followers/]
          followers_by_type($1.singularize.classify)
        else
          super
        end
      end

      # 返回follow的数量
      def blocked_followers_count
        self.followings.blocked.count
      end

      # Returns the followings records scoped
      def followers_scoped
        # 因为follow belongs_to follower
        self.followings.includes(:follower)
      end

      # apply_options_to_scope是定义在lib/acts_as_followable/follower_lib.rb的一个方法,只要是过滤参数,把参数提取加到where scope中
      # 返回所有followers
      def followers(options={})
        followers_scope = followers_scoped.unblocked
        followers_scope = apply_options_to_scope(followers_scope, options)
        # 取出所有follower
        followers_scope.to_a.collect{|f| f.follower}
      end

      def blocks(options={})
        blocked_followers_scope = followers_scoped.blocked
        blocked_followers_scope = apply_options_to_scope(blocked_followers_scope, options)
        blocked_followers_scope.to_a.collect{|f| f.follower}
      end

      # Returns true if the current instance is followed by the passed record
      # Returns false if the current instance is blocked by the passed record or no follow is found
      # 查看是否被follow
      def followed_by?(follower)
        self.followings.unblocked.for_follower(follower).first.present?
      end

      # 如果查到记录就变成true，没有就创建那条记录
      def block(follower)
        get_follow_for(follower) ? block_existing_follow(follower) : block_future_follow(follower)
      end

      # 把following的记录删除
      def unblock(follower)
        get_follow_for(follower).try(:delete)
      end

      # 返回是否被follower, for_follower是个where scope
      def get_follow_for(follower)
        self.followings.for_follower(follower).first
      end

      private

      # 创建following记录
      def block_future_follow(follower)
        Follow.create(:followable => self, :follower => follower, :blocked => true)
      end

      # block!会把blocked属性变成true
      def block_existing_follow(follower)
        get_follow_for(follower).block!
      end

    end

  end
end

```

+ lib/acts_as_follower/follower.rb: 这是一个关于**follower**(叫做跟随者), 跟**followable.rb**的作用差不多，只是是反过来的

``` ruby lib/acts_as_follower/follower.rb
module ActsAsFollower #:nodoc:
  module Follower

    # active_record会include
    def self.included(base)
      base.extend ClassMethods
    end

    module ClassMethods
      # 在model里就可以用acts_as_follower
      def acts_as_follower
        has_many :follows, :as => :follower, :dependent => :destroy
        # 具体类加载实例方法
        include ActsAsFollower::Follower::InstanceMethods
        # 具体类加载FollowerLib库方法
        include ActsAsFollower::FollowerLib
      end
    end

    module InstanceMethods

      # Returns true if this instance is following the object passed as an argument.
      # 是否following某个followable
      def following?(followable)
        0 < Follow.unblocked.for_follower(self).for_followable(followable).count
      end

      # Returns the number of objects this instance is following.
      # 在数据表中查看这个follower总共跟随了多少followable
      def follow_count
        Follow.unblocked.for_follower(self).count
      end

      # Creates a new follow record for this instance to follow the passed object.
      # Does not allow duplicate records to be created.
      # follow某followable
      def follow(followable)
        if self != followable
          self.follows.find_or_create_by(followable_id: followable.id, followable_type: parent_class_name(followable))
        end
      end

      # Deletes the follow record if it exists.
      def stop_following(followable)
        # get_follow是用followable来找到那条follow记录然后把它删除掉
        if follow = get_follow(followable)
          follow.destroy
        end
      end

      # returns the follows records to the current instance
      def follows_scoped
        self.follows.unblocked.includes(:followable)
      end

      # Returns the follow records related to this instance by type.
      def follows_by_type(followable_type, options={})
        follows_scope  = follows_scoped.for_followable_type(followable_type)
        follows_scope = apply_options_to_scope(follows_scope, options)
      end

      # Returns the follow records related to this instance with the followable included.
      def all_follows(options={})
        follows_scope = follows_scoped
        follows_scope = apply_options_to_scope(follows_scope, options)
      end

      # Returns the actual records which this instance is following.
      def all_following(options={})
        all_follows(options).collect{ |f| f.followable }
      end

      # Returns the actual records of a particular type which this record is following.
      def following_by_type(followable_type, options={})
        followables = followable_type.constantize.
          joins(:followings).
          where('follows.blocked'         => false,
                'follows.follower_id'     => self.id,
                'follows.follower_type'   => parent_class_name(self),
                'follows.followable_type' => followable_type)
        if options.has_key?(:limit)
          followables = followables.limit(options[:limit])
        end
        if options.has_key?(:includes)
          followables = followables.includes(options[:includes])
        end
        followables
      end

      def following_by_type_count(followable_type)
        follows.unblocked.for_followable_type(followable_type).count
      end

      # Allows magic names on following_by_type
      # e.g. following_users == following_by_type('User')
      # Allows magic names on following_by_type_count
      # e.g. following_users_count == following_by_type_count('User')
      def method_missing(m, *args)
        if m.to_s[/following_(.+)_count/]
          following_by_type_count($1.singularize.classify)
        elsif m.to_s[/following_(.+)/]
          following_by_type($1.singularize.classify)
        else
          super
        end
      end

      # Returns a follow record for the current instance and followable object.
      def get_follow(followable)
        self.follows.unblocked.for_followable(followable).first
      end

    end

  end
end

```
+ lib/acts_as_follower/follow_scopes.rb: 这里封装的都是一些查询语句,在**follow.rb**文件里有一句`extend ActsAsFollower::FollowScopes`可以在实例上使用这些方法

``` ruby lib/acts_as_follower/follow_scopes.rb
module ActsAsFollower #:nodoc:
  module FollowScopes

    def for_follower(follower)
      where(:follower_id => follower.id,
            :follower_type => parent_class_name(follower))
    end

    def for_followable(followable)
      where(:followable_id => followable.id, :followable_type => parent_class_name(followable))
    end

    def for_follower_type(follower_type)
      where(:follower_type => follower_type)
    end

    def for_followable_type(followable_type)
      where(:followable_type => followable_type)
    end

    def recent(from)
      where(["created_at > ?", (from || 2.weeks.ago).to_s(:db)])
    end

    def descending
      order("follows.created_at DESC")
    end

    def unblocked
      where(:blocked => false)
    end

    def blocked
      where(:blocked => true)
    end

  end
end
```

+ lib/acts_as_follower/follower_lib.rb: 这里封装了两个方法,上文有讲过,**follow.rb**文件里有一句`extend ActsAsFollower::FollowerLib`

``` ruby lib/acts_as_follower/follower_lib.rb
module ActsAsFollower
  module FollowerLib

    private

    # Retrieves the parent class name if using STI.
    def parent_class_name(obj)
      if obj.class.superclass != ActiveRecord::Base
        return obj.class.superclass.name
      end
      return obj.class.name
    end

    def apply_options_to_scope(scope, options = {})
      if options.has_key?(:limit)
        scope = scope.limit(options[:limit])
      end
      if options.has_key?(:includes)
        scope = scope.includes(options[:includes])
      end
      if options.has_key?(:joins)
        scope = scope.joins(options[:joins])
      end
      if options.has_key?(:where)
        scope = scope.order(options[:where])
      end
      if options.has_key?(:order)
        scope = scope.order(options[:order])
      end
      scope
    end
  end
end

```

+ lib/acts_as_follower/generators/templates/follow.rb: 这个文件是用来生成模版的,生成的是一个model文件,放在app/models下,它是一个中间表,是所有**follower**和**followable**的链接点

``` ruby app/models/follow.rb
class Follow < ActiveRecord::Base

  extend ActsAsFollower::FollowerLib
  extend ActsAsFollower::FollowScopes

  # NOTE: Follows belong to the "followable" interface, and also to followers
  belongs_to :followable, :polymorphic => true
  belongs_to :follower,   :polymorphic => true

  def block!
    self.update_attribute(:blocked, true)
  end

end
```

### 总结

学会组织代码的能力和一些rails的内部的东西,学习这个gem查询一些数据的方法,还有双重多态的写法
