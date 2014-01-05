---
layout: post
title: "acts-as-taggable-on源码分析(二)"
date: 2014-01-04 23:15
comments: true
categories: ruby on rails
---

继上节[Acts-as-taggable-on源码分析(一)](/blog/2014/01/04/acts-as-taggable-onyuan-ma-fen-xi-1/)我们继续来讲解

这节我们来分析各个模型之间的关系

acts-as-taggable-on主要涉及两个表,一个叫tags,它只有一个字段name,存的是tag的内容,还有一个表叫taggings

<!-- more -->
除了这两个主要表,我们还要给商品或新闻,或文章打tag,所以还需要这张表,这一部分的信息存在tagggins表中的taggable_id和taggable_type中,除此之外还可以指定是哪个用户打的tag,也就是tag的拥有者,我们用一个users表,它的信息存在taggings表中的tagger_type和tagger_id中

下面是它们的模型关系图

{% img /images/acts_as_taggable_on/model.png%}

### (一)tag.rb

``` ruby
module ActsAsTaggableOn
  class Tag < ::ActiveRecord::Base
    include ActsAsTaggableOn::Utils
    has_many :taggings, :dependent => :destroy, :class_name => 'ActsAsTaggableOn::Tagging'
  end
end
```

tag是has_many :taggings的

ActsAsTaggableOn::Utils的内容是

``` ruby
module ActsAsTaggableOn
  module Utils
    def self.included(base)

      base.send :include, ActsAsTaggableOn::Utils::OverallMethods
      base.extend ActsAsTaggableOn::Utils::OverallMethods
    end

    module OverallMethods
      # 判断是否使用PostgreSQL
      def using_postgresql?
        ::ActiveRecord::Base.connection && ::ActiveRecord::Base.connection.adapter_name == 'PostgreSQL'
      end

      # 判断是否使用SQLite
      def using_sqlite?
        ::ActiveRecord::Base.connection && ::ActiveRecord::Base.connection.adapter_name == 'SQLite'
      end

      # 生成六位数的随机乱码
      def sha_prefix(string)
        Digest::SHA1.hexdigest("#{string}#{rand}")[0..6]
      end

      private
      def like_operator
        using_postgresql? ? 'ILIKE' : 'LIKE'
      end

      # escape _ and % characters in strings, since these are wildcards in SQL.
      # 跳脱!%_,在这三种字符前面加上!
       def escape_like(str)
         str.gsub(/[!%_]/){ |x| '!' + x }
       end
    end

  end
end
```

### (二)taggable.rb

``` ruby
module ActsAsTaggableOn
  module Taggable
    # 默认为false
    def taggable?
      false
    end

    def acts_as_taggable
      acts_as_taggable_on :tags
    end

    def acts_as_taggable_on(*tag_types)
      taggable_on(false, tag_types)
    end

    private

      def taggable_on(preserve_tag_order, *tag_types)
        ...
        class_eval do
          has_many :taggings, :as => :taggable, :dependent => :destroy, :class_name => "ActsAsTaggableOn::Tagging"
          has_many :base_tags, :through => :taggings, :source => :tag, :class_name => "ActsAsTaggableOn::Tag"

          def self.taggable?
            true
          end

          include ActsAsTaggableOn::Utils
        end
        ...
        # called on each call of taggable_on
        include ActsAsTaggableOn::Taggable::Core
        include ActsAsTaggableOn::Taggable::Collection
        include ActsAsTaggableOn::Taggable::Cache
        include ActsAsTaggableOn::Taggable::Ownership
        include ActsAsTaggableOn::Taggable::Related
        include ActsAsTaggableOn::Taggable::Dirty
      end

  end
end
```

taggable.rb主要是定义了acts_as_taggable这样的方法,这是主要的方法,在任何需要打tag的model都要用这个方法,这个方法还导入了很多modules

关键在这两行

``` ruby
class_eval do
  has_many :taggings, :as => :taggable, :dependent => :destroy, :class_name => "ActsAsTaggableOn::Tagging"
  has_many :base_tags, :through => :taggings, :source => :tag, :class_name => "ActsAsTaggableOn::Tag"
end
```

### (三)tagger.rb

``` ruby
...
def acts_as_tagger(opts={})
  class_eval do
    has_many_with_compatibility :owned_taggings, 
      opts.merge(
        :as => :tagger, 
        :dependent => :destroy,
        :class_name => "ActsAsTaggableOn::Tagging"
      )

    has_many_with_compatibility :owned_tags, 
                                :through => :owned_taggings, 
                                :source => :tag, 
                                :class_name => "ActsAsTaggableOn::Tag", 
                                :uniq => true
  end

  ...
end
...
```

跟taggable.rb的关系有些类似,都是通过多态来实现的

### (四)tagging.rb

``` ruby
module ActsAsTaggableOn
  class Tagging < ::ActiveRecord::Base #:nodoc:
    ...
    belongs_to :tag, :class_name => 'ActsAsTaggableOn::Tag'
    belongs_to :taggable, :polymorphic => true
    belongs_to :tagger,   :polymorphic => true
    ...
  end
end
```

很清晰的关系
