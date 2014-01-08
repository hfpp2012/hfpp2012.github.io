---
layout: post
title: "acts-as-taggable-on源码分析(九)"
date: 2014-01-08 16:06
comments: true
categories: ruby on rails
---

+ [Acts-as-taggable-on源码分析(一)](/blog/2014/01/04/acts-as-taggable-onyuan-ma-fen-xi-1/)
+ [Acts-as-taggable-on源码分析(二)](/blog/2014/01/04/acts-as-taggable-onyuan-ma-fen-xi-2/)
+ [Acts-as-taggable-on源码分析(三)](/blog/2014/01/06/acts-as-taggable-onyuan-ma-fen-xi-3/)
+ [Acts-as-taggable-on源码分析(四)](/blog/2014/01/07/acts-as-taggable-onyuan-ma-fen-xi-4/)
+ [Acts-as-taggable-on源码分析(五)](/blog/2014/01/07/acts-as-taggable-onyuan-ma-fen-xi-5/)
+ [Acts-as-taggable-on源码分析(六)](/blog/2014/01/08/acts-as-taggable-onyuan-ma-fen-xi-6/)
+ [Acts-as-taggable-on源码分析(七)](/blog/2014/01/08/acts-as-taggable-onyuan-ma-fen-xi-7/)
+ [Acts-as-taggable-on源码分析(八)](/blog/2014/01/08/acts-as-taggable-onyuan-ma-fen-xi-8/)

这节我们主要来研究cache,主要是cached_tag_list字段的用法

<!-- more -->

每次查topic的tag_list都要从数据库取,有点麻烦,acts_as_taggable_on提供了一个解决方案,在topics表中增加一个字段cached_tag_list,存的是tag_list,例如'ruby, rails',这样就方便多了

```
rails g migration add_cached_tag_list_to_topics cached_tag_list:string

t1.tag_list = 'ruby, rails, php, python'
=> UPDATE `topics` SET `cached_tag_list` = 'ruby, rails, php, python', `updated_at` = '2014-01-08 07:57:40' WHERE `topics`.`id` = 4
```

``` ruby
tag_types.map(&:to_s).each do |tag_type|
  class_eval <<-RUBY, __FILE__, __LINE__ + 1
    # 判断是否拥有cached_tag_list列
    def self.caching_#{tag_type.singularize}_list?
      caching_tag_list_on?("#{tag_type}")
    end
  RUBY
end

def caching_tag_list_on?(context)
  column_names.include?("cached_#{context.to_s.singularize}_list")
end

before_save :save_cached_tag_list
def save_cached_tag_list
  tag_types.map(&:to_s).each do |tag_type|
    # 先判断是否有cached_tag_list列
    if self.class.send("caching_#{tag_type.singularize}_list?")
      # 参数解析,前面已讲过
      if tag_list_cache_set_on(tag_type)
        list = tag_list_cache_on(tag_type).to_a.flatten.compact.join(', ')
        # 存其值
        self["cached_#{tag_type.singularize}_list"] = list
      end
    end
  end

  true
end
```

如果没有cached_tag_list列,这个特性默认是不开启的

``` ruby
# 至少存在一个类似cached_tag_list这样的列就不会退出
return unless base.table_exists? && base.tag_types.any? { |context| base.column_names.include?("cached_#{context.to_s.singularize}_list") }
```


