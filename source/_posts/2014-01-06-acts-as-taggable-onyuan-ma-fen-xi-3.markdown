---
layout: post
title: "acts-as-taggable-on源码分析(三)"
date: 2014-01-06 09:38
comments: true
categories: ruby on rails
---

上两节:

[Acts-as-taggable-on源码分析(一)](/blog/2014/01/04/acts-as-taggable-onyuan-ma-fen-xi-1/)

[Acts-as-taggable-on源码分析(二)](/blog/2014/01/04/acts-as-taggable-onyuan-ma-fen-xi-2/)

这节我们来使用acts_as_taggable这个方法

``` ruby
class Topic < ActiveRecord::Base
  acts_as_taggable # Alias for acts_as_taggable_on :tags
end
```

<!-- more -->

``` ruby
t = Topic.first
t.tag_list = 'ruby, rails'
t.save
```

{% img /images/acts_as_taggable_on/acts_as_taggable.png %}

先来看看acts_as_taggable到底做了什么

它定义在taggable.rb文件中

``` ruby
def acts_as_taggable
  acts_as_taggable_on :tags
end
```

继续看acts_as_taggable_on是什么

``` ruby
def acts_as_taggable_on(*tag_types)
  taggable_on(false, tag_types)
end
```

来到taggable_on

``` ruby
def taggable_on(preserve_tag_order, *tag_types)
  # 把所有种类的标签打成symbol
  tag_types = tag_types.to_a.flatten.compact.map(&:to_sym)

  # 如果已经用过taggable_on方法,也就是打过标签了,taggable_on会动态修改taggable?方法的定义,而它的值将由false被修改成true
  if taggable?
    # 数组加法
    self.tag_types = (self.tag_types + tag_types).uniq
    self.preserve_tag_order = preserve_tag_order
  else
    # 只属于类
    class_attribute :tag_types
    self.tag_types = tag_types
    class_attribute :preserve_tag_order
    self.preserve_tag_order = preserve_tag_order

    class_eval do
      has_many :taggings, :as => :taggable, :dependent => :destroy, :class_name => "ActsAsTaggableOn::Tagging"
      has_many :base_tags, :through => :taggings, :source => :tag, :class_name => "ActsAsTaggableOn::Tag"

      # 动态更改了taggable?方法的定义
      def self.taggable?
        true
      end

      include ActsAsTaggableOn::Utils
    end
  end

  # each of these add context-specific methods and must be
  # called on each call of taggable_on
  include ActsAsTaggableOn::Taggable::Core
  include ActsAsTaggableOn::Taggable::Collection
  include ActsAsTaggableOn::Taggable::Cache
  include ActsAsTaggableOn::Taggable::Ownership
  include ActsAsTaggableOn::Taggable::Related
  include ActsAsTaggableOn::Taggable::Dirty
end
```

通过分析acts_as_taggable的方法定义,并没有找到实际的关于保存之后修改tags表和taggings表的语句。

按照我们的猜测如果要实现上述类似的功能,必须要定义一个callback,可以叫before_save或before_save

到底是before_save还是after_save呢

是after_save,为什么,因为你看上面那个图就知道,修改那两个表都是在更变topics表的updated_at之后才发生的,所以是after_save

我们来找这个方法,只能在剩下的几个include语句里下功夫了

我们只能一个一个翻,好运的是include ActsAsTaggableOn::Taggable::Core引用的lib/acts_as_taggable/core.rb文件就定义了这样的after_save

``` ruby
after_save :save_tags

def save_tags
  tagging_contexts.each do |context|
    # 如果当前记录(topic)没有类似tag_list这样的方法就退出
    next unless tag_list_cache_set_on(context)
    # 返回当前记录相关tags的tag_list变量的值
    tag_list = tag_list_cache_on(context).uniq

    # 处理tags表,通过tag_list看是否有存在相同的tag,有的话就返回,没有就创建一个
    tags = ActsAsTaggableOn::Tag.find_or_create_all_with_like_by_name(tag_list)

    # 查当前记录已经存在数据中的tag
    current_tags = tags_on(context)

    # Tag maintenance based on whether preserving the created order of tags
    if self.class.preserve_tag_order?
      old_tags, new_tags = current_tags - tags, tags - current_tags

      shared_tags = current_tags & tags

      if shared_tags.any? && tags[0...shared_tags.size] != shared_tags
        index = shared_tags.each_with_index { |_, i| break i unless shared_tags[i] == tags[i] }

        # Update arrays of tag objects
        old_tags |= current_tags[index...current_tags.size]
        new_tags |= current_tags[index...current_tags.size] & shared_tags

        # Order the array of tag objects to match the tag list
        new_tags = tags.map do |t| 
          new_tags.find { |n| n.name.downcase == t.name.downcase }
        end.compact
      end
    else
      # current_tags是当前记录在数据库有的tag, tags是新的tag
      # 所以old_tags是老的tag
      # new_tags是新的
      # 大家可以动手找几个数组试验一下结果
      old_tags = current_tags - tags
      new_tags = tags - current_tags
    end

    # 查taggings表,准备删除老的taggings记录
    if old_tags.present?
      old_taggings = taggings.where(:tagger_type => nil, :tagger_id => nil, :context => context.to_s, :tag_id => old_tags)
    end

    # 删除老的taggings记录
    if old_taggings.present?
      ActsAsTaggableOn::Tagging.destroy_all "#{ActsAsTaggableOn::Tagging.primary_key}".to_sym => old_taggings.map(&:id)
    end

    # 创建新的taggings记录
    new_tags.each do |tag|
      taggings.create!(:tag_id => tag.id, :context => context.to_s, :taggable => self)
    end
  end

  true
end
```

我们把所有相关的方法找出来一个个来看

``` ruby
def tagging_contexts
  custom_contexts + self.class.tag_types.map(&:to_s)
end

attr_writer :custom_contexts

def custom_contexts
  @custom_contexts ||= []
end

# 判断是否定义了类似tag_list这样的实例变量而且它的值不能为nil
def tag_list_cache_set_on(context)
  variable_name = "@#{context.to_s.singularize}_list"
  instance_variable_defined?(variable_name) && !instance_variable_get(variable_name).nil?
end

# 如果类似tag_list这样的实例变量的值存在而不为nil,就返回
def tag_list_cache_on(context)
  variable_name = "@#{context.to_s.singularize}_list"
  return instance_variable_get(variable_name) if instance_variable_defined?(variable_name) && instance_variable_get(variable_name)
  # tags_on(context).map(&name)查的是tags表,返回当前记录(topic)相关context(tags)的base_tags的所有name
  instance_variable_set(variable_name, ActsAsTaggableOn::TagList.new(tags_on(context).map(&:name)))
end

# 主要是一个where语句,关联(join)taggings表的context字段,查tags表
def tags_on(context)
  # topics has_many base_tags,acts_as_taggable定义的
  # 关联taggins表,看它的context是否等于类似tags而且它的tagger_id不能为NULL
  scope = base_tags.where(["#{ActsAsTaggableOn::Tagging.table_name}.context = ? AND #{ActsAsTaggableOn::Tagging.table_name}.tagger_id IS NULL", context.to_s])
  # 如果preserve_tag_order等于true根据taggings表的id来排序
  scope = scope.order("#{ActsAsTaggableOn::Tagging.table_name}.id") if self.class.preserve_tag_order?
  scope
end

def self.find_or_create_all_with_like_by_name(*list)
  list = [list].flatten

  return [] if list.empty?

  # 查tags表是否已经有tag了
  existing_tags = Tag.named_any(list)

  list.map do |tag_name|
    # 把tag_name转成小写的
    comparable_tag_name = comparable_name(tag_name)
    # 发现tags表已经有了就返回
    existing_tag = existing_tags.find { |tag| comparable_name(tag.name) == comparable_tag_name }

    # 没有的就创建一个tag
    existing_tag || Tag.create(:name => tag_name)
  end
end

# 查tags表返回数组
def self.named_any(list)
  if ActsAsTaggableOn.strict_case_match
    where(list.map { |tag| sanitize_sql(["name = #{binary}?", tag.to_s.mb_chars]) }.join(" OR "))
  else
    where(list.map { |tag| sanitize_sql(["lower(name) = ?", tag.to_s.mb_chars.downcase]) }.join(" OR "))
  end
end

def comparable_name(str)
  str.mb_chars.downcase.to_s
end
```

tag_list_cache_on和tag_list_cache_set_on跟tag_list变量有关,跟cache有关,它们都是实例方法,可以用类似t.tag_list_cache_set_on('tags')来调用,可以试下效果

tags_on也是个实例方法,它查找tags表,返回当前记录(topic)的base_tags

关于tag_list和ActsAsTaggableOn::Tagging.table_name后绪会说

总结save_tags的作用,首先是检查tag_list变量的值,先把它的值保存起来,它是的新的tag的列表,然后通过这个tag_list去查数据库中的tags表,如果已经有了相同的name的tag就跳过,如果没有就创建,先把tags表创建好,接着来处理taggings表,首先求当前记录在数据库的tag表相关的记录,通过数组的运算来求出新的tags和老的tags,然后把老的taggings相关的记录删除,再创建新的,这样就完成了。

