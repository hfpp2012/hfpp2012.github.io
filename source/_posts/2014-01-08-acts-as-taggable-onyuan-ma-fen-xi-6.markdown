---
layout: post
title: "acts-as-taggable-on源码分析(六)"
date: 2014-01-08 09:59
comments: true
categories: ruby on rails
---

+ [Acts-as-taggable-on源码分析(一)](/blog/2014/01/04/acts-as-taggable-onyuan-ma-fen-xi-1/)
+ [Acts-as-taggable-on源码分析(二)](/blog/2014/01/04/acts-as-taggable-onyuan-ma-fen-xi-2/)
+ [Acts-as-taggable-on源码分析(三)](/blog/2014/01/06/acts-as-taggable-onyuan-ma-fen-xi-3/)
+ [Acts-as-taggable-on源码分析(四)](/blog/2014/01/07/acts-as-taggable-onyuan-ma-fen-xi-4/)
+ [Acts-as-taggable-on源码分析(五)](/blog/2014/01/07/acts-as-taggable-onyuan-ma-fen-xi-5/)

这节我们来研究Relationships,即是find_related_tags这个方法的原理和用法

<!-- more -->

```
t1.tag_list => ["ruby", "rails"]
t2.tag_list => ["python"]
t3.tag_list => ["ruby", "rails", "python"]

t1.find_related_tag => t3
t2.find_related_tag => t3
t3.find_related_tag => t1, t2
```

def find_related_#{tag_type}(options = {})这一行就是find_related_tags的定义之处

``` ruby
...
tag_types.map(&:to_s).each do |tag_type|
  class_eval <<-RUBY, __FILE__, __LINE__ + 1
    def find_related_#{tag_type}(options = {})
      related_tags_for('#{tag_type}', self.class, options)
    end
    alias_method :find_related_on_#{tag_type}, :find_related_#{tag_type}
    ...
  RUBY
end
...
```

找到related_tags_for方法

``` ruby
def related_tags_for(context, klass, options = {})
  tags_to_ignore = Array.wrap(options.delete(:ignore)).map(&:to_s) || []
  tags_to_find = tags_on(context).collect { |t| t.name }.reject { |t| tags_to_ignore.include? t }

  # 查topics, tags, taggings表,找taggs.name存在于tags_to_find的然后用group by topics.id
  klass.select("#{klass.table_name}.*, COUNT(#{ActsAsTaggableOn::Tag.table_name}.#{ActsAsTaggableOn::Tag.primary_key}) AS count") \
       .from("#{klass.table_name}, #{ActsAsTaggableOn::Tag.table_name}, #{ActsAsTaggableOn::Tagging.table_name}") \
       .where(["#{exclude_self(klass, id)} #{klass.table_name}.#{klass.primary_key} = #{ActsAsTaggableOn::Tagging.table_name}.taggable_id AND #{ActsAsTaggableOn::Tagging.table_name}.taggable_type = '#{klass.base_class.to_s}' AND #{ActsAsTaggableOn::Tagging.table_name}.tag_id = #{ActsAsTaggableOn::Tag.table_name}.#{ActsAsTaggableOn::Tag.primary_key} AND #{ActsAsTaggableOn::Tag.table_name}.name IN (?)", tags_to_find]) \
       .group(group_columns(klass)) \
       .order("count DESC")
end

# 不包含自己
def exclude_self(klass, id)
  if [self.class.base_class, self.class].include? klass
    "#{klass.table_name}.#{klass.primary_key} != #{id} AND"
  else
    nil
  end
end
```

find_related_tags的主要作用就是通过tags.name in ...来查找相关的记录的

其他的几个方法例如find_matching_contexts_for, find_related_tags_for, find_matching_contexts_for可以自己研究
