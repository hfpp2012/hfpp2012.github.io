---
layout: post
title: "acts-as-taggable-on源码分析(七)"
date: 2014-01-08 10:48
comments: true
categories: ruby on rails
---

+ [Acts-as-taggable-on源码分析(一)](/blog/2014/01/04/acts-as-taggable-onyuan-ma-fen-xi-1/)
+ [Acts-as-taggable-on源码分析(二)](/blog/2014/01/04/acts-as-taggable-onyuan-ma-fen-xi-2/)
+ [Acts-as-taggable-on源码分析(三)](/blog/2014/01/06/acts-as-taggable-onyuan-ma-fen-xi-3/)
+ [Acts-as-taggable-on源码分析(四)](/blog/2014/01/07/acts-as-taggable-onyuan-ma-fen-xi-4/)
+ [Acts-as-taggable-on源码分析(五)](/blog/2014/01/07/acts-as-taggable-onyuan-ma-fen-xi-5/)
+ [Acts-as-taggable-on源码分析(六)](/blog/2014/01/08/acts-as-taggable-onyuan-ma-fen-xi-6/)

这节我们来讲解Tag Ownership,也即是acts_as_tagger的原理

<!-- more -->

``` ruby
class Topic < ActiveRecord::Base
  acts_as_taggable
end

class User < ActiveRecord::Base
  acts_as_tagger
end
```

来看下acts_as_tagger实现的源码

``` ruby
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
```

所以可以user.owned_taggings, user.owned_tags

```
u = User.first
u.tag(t1, :with => "user1", :on => :tags)

=>
INSERT INTO `taggings` (`context`, `created_at`, `tag_id`, `taggable_id`, `taggable_type`, `tagger_id`, `tagger_type`) VALUES ('tags', '2014-01-08 03:04:29', 19, 4, 'Topic', 1, 'User')
```

``` ruby
def tag(taggable, opts={})
  # 即将有相同的key也不替换
  opts.reverse_merge!(:force => true)
  # 是否要跳过save
  skip_save = opts.delete(:skip_save)
  # 如果taggable没有使用acts_as_taggable就退出
  return false unless taggable.respond_to?(:is_taggable?) && taggable.is_taggable?

  # 必须传:on参数
  raise "You need to specify a tag context using :on"                unless opts.has_key?(:on)
  # 必须传:with参数
  raise "You need to specify some tags using :with"                  unless opts.has_key?(:with)
  # context参数没传对
  raise "No context :#{opts[:on]} defined in #{taggable.class.to_s}" unless (opts[:force] || taggable.tag_types.include?(opts[:on]))

  taggable.set_owner_tag_list_on(self, opts[:on].to_s, opts[:with])
  taggable.save unless skip_save
end
```

set_owner_tag_list_on方法定义在lib/acts_as_taggable_on/ownership.rb文件中

``` ruby
def set_owner_tag_list_on(owner, context, new_list)
  # 把context添加@custom_context变量中
  add_custom_context(context)

  # 取变量的值,初始可能为空
  cache = cached_owned_tag_list_on(context)

  # 现在的值不为空,很关键的语句
  cache[owner] = ActsAsTaggableOn::TagList.from(new_list)
end

def cached_owned_tag_list_on(context)
  variable_name = "@owned_#{context}_list"
  # 如果有定义variable_name,就把值给cache,没有的话就设为{}
  cache = (instance_variable_defined?(variable_name) && instance_variable_get(variable_name)) || instance_variable_set(variable_name, {})
end
```

怎么在taggings表中的tagger_type存User的呢

玄机在taggable.save unless skip_save

taggable保存之后

``` ruby
after_save :save_owned_tags

def save_owned_tags
  tagging_contexts.each do |context|
    # onwer是个key,tag_list是值
    cached_owned_tag_list_on(context).each do |owner, tag_list|

      # 查找已经存在的tag或者创建新的,返回的是当前的tag
      tags = ActsAsTaggableOn::Tag.find_or_create_all_with_like_by_name(tag_list.uniq)

      # 关联tagger查tags表,返回where查询
      owned_tags = owner_tags_on(owner, context)

      # Tag maintenance based on whether preserving the created order of tags
      if self.class.preserve_tag_order?
        old_tags, new_tags = owned_tags - tags, tags - owned_tags

        shared_tags = owned_tags & tags

        if shared_tags.any? && tags[0...shared_tags.size] != shared_tags
          index = shared_tags.each_with_index { |_, i| break i unless shared_tags[i] == tags[i] }

          # Update arrays of tag objects
          old_tags |= owned_tags.from(index)
          new_tags |= owned_tags.from(index) & shared_tags

          # Order the array of tag objects to match the tag list
          new_tags = tags.map { |t| new_tags.find { |n| n.name.downcase == t.name.downcase } }.compact
        end
      else
        # 找到老的tags
        old_tags = owned_tags - tags
        # 找到新的tags
        new_tags = tags - owned_tags
      end

      # 从tags中找到老的,准备关联查找taggings,并删除
      if old_tags.present?
        old_taggings = ActsAsTaggableOn::Tagging.where(:taggable_id => id, :taggable_type => self.class.base_class.to_s,
                                                       :tagger_type => owner.class.base_class.to_s, :tagger_id => owner.id,
                                                       :tag_id => old_tags, :context => context)
      end

      # 删除老的taggings
      if old_taggings.present?
        ActsAsTaggableOn::Tagging.destroy_all(:id => old_taggings.map(&:id))
      end

      # 创建新的taggings包括tagger
      new_tags.each do |tag|
        taggings.create!(:tag_id => tag.id, :context => context.to_s, :tagger => owner, :taggable => self)
      end
    end
  end

  true
end

def owner_tags_on(owner, context)
  # 查拥有者
  # 因为base_tags是通过joins来的,所以加个条件taggings.context = ?...
  if owner.nil?
    scope = base_tags.where([%(#{ActsAsTaggableOn::Tagging.table_name}.context = ?), context.to_s])
  else
    scope = base_tags.where([%(#{ActsAsTaggableOn::Tagging.table_name}.context = ? AND
                               #{ActsAsTaggableOn::Tagging.table_name}.tagger_id = ? AND
                               #{ActsAsTaggableOn::Tagging.table_name}.tagger_type = ?), context.to_s, owner.id, owner.class.base_class.to_s])
  end

  # 排序
  if self.class.preserve_tag_order?
    scope.order("#{ActsAsTaggableOn::Tagging.table_name}.id")
  else
    scope
  end
end
```

save_owned_tags的方法跟以前的core.rb中的save_tags有些相似,都是通过找taggs表,如果有新的就创建新的tag,然后关联taggings表,删除老的,创建新的,只是不同的是,save_owned_tags多加了tagger的保存,另外,save_owned_tags是在tagging_contexts不为空的情况下才有数据库的操作,也就是说发生在tag方法上,所以它跟core.rb中的save_tags不冲突。

现在来看下tags_from这个方法

```
t1.tags_from(u) => ["user1"]
```

``` ruby
# 重写acts_as_taggable_on
def acts_as_taggable_on(*args)
  initialize_acts_as_taggable_on_ownership
  super(*args)
end

def initialize_acts_as_taggable_on_ownership
  tag_types.map(&:to_s).each do |tag_type|
    class_eval <<-RUBY, __FILE__, __LINE__ + 1
      # 定义tags_from方法
      def #{tag_type}_from(owner)
        owner_tag_list_on(owner, '#{tag_type}')
      end
    RUBY
  end
end

def owner_tag_list_on(owner, context)
  add_custom_context(context)

  # 从变量取值,没有就为空
  cache = cached_owned_tag_list_on(context)

  # 查topics的base_tags,通过owner和context来关联
  # 返回的是数组,包含tag的name
  cache[owner] ||= ActsAsTaggableOn::TagList.new(*owner_tags_on(owner, context).map(&:name))
end
```

t1.reload

``` ruby
def reload(*args)
  # 把所有cache变量都清空
  self.class.tag_types.each do |context|
    instance_variable_set("@owned_#{context}_list", nil)
  end

  super(*args)
end
```
