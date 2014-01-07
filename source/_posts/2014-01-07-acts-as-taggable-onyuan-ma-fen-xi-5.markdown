---
layout: post
title: "acts-as-taggable-on源码分析(五)"
date: 2014-01-07 20:32
comments: true
categories: ruby on rails
---

+ [Acts-as-taggable-on源码分析(一)](/blog/2014/01/04/acts-as-taggable-onyuan-ma-fen-xi-1/)
+ [Acts-as-taggable-on源码分析(二)](/blog/2014/01/04/acts-as-taggable-onyuan-ma-fen-xi-2/)
+ [Acts-as-taggable-on源码分析(三)](/blog/2014/01/06/acts-as-taggable-onyuan-ma-fen-xi-3/)
+ [Acts-as-taggable-on源码分析(四)](/blog/2014/01/07/acts-as-taggable-onyuan-ma-fen-xi-4/)

这节我们来研究Finding Tagged Objects,也就是tagged_with这个方法的原理和使用方法

<!-- more -->

{% img /images/acts_as_taggable_on/tagged_with.png %}

我们先看tagged_with('ruby, rails')这种最简单的形式

``` ruby
def tagged_with(tags, options = {})
  # 返回tag_list的数组
  tag_list = ActsAsTaggableOn::TagList.from(tags)
  # 返回空的数组
  empty_result = where("1 = 0")

  # 如果tag_list为空就返回空的数组
  return empty_result if tag_list.empty?

  joins = []
  conditions = []
  having = []
  select_clause = []

  context = options.delete(:on)
  owned_by = options.delete(:owned_by)
  # 返回表名
  # Guesses the table name, but does not decorate it with prefix and suffix information.
  alias_base_name = undecorated_table_name.gsub('.','_')
  quote = ActsAsTaggableOn::Tag.using_postgresql? ? '"' : ''

  if options.delete(:exclude)
    ...
  elsif options.delete(:any)
    ...
  else
    # 查tags表找到匹配的tag
    tags = ActsAsTaggableOn::Tag.named_any(tag_list)

    # 当没有完全匹配时就返回空的数组
    # 也就是说tag_list中的tag必须存于数据库tags表中的
    return empty_result unless tags.length == tag_list.length

    tags.each do |tag|
      # 加上表别名
      taggings_alias = adjust_taggings_alias("#{alias_base_name[0..11]}_taggings_#{sha_prefix(tag.name)}")
      # joins taggings表
      tagging_join  = "JOIN #{ActsAsTaggableOn::Tagging.table_name} #{taggings_alias}" +
                      "  ON #{taggings_alias}.taggable_id = #{quote}#{table_name}#{quote}.#{primary_key}" +
                      " AND #{taggings_alias}.taggable_type = #{quote_value(base_class.name)}" +
                      " AND #{taggings_alias}.tag_id = #{tag.id}"

      tagging_join << " AND " + sanitize_sql(["#{taggings_alias}.context = ?", context.to_s]) if context

      # 查看是否有拥有者,即tagger
      if owned_by
          tagging_join << " AND " +
            sanitize_sql([
              "#{taggings_alias}.tagger_id = ? AND #{taggings_alias}.tagger_type = ?",
              owned_by.id,
              owned_by.class.base_class.to_s
            ])
      end

      joins << tagging_join
    end
  end

  taggings_alias, tags_alias = adjust_taggings_alias("#{alias_base_name}_taggings_group"), "#{alias_base_name}_tags_group"

  if options.delete(:match_all)
    ...
  end

  # 组合sql语句
  select(select_clause) \
    .joins(joins.join(" ")) \
    .where(conditions.join(" AND ")) \
    .group(group) \
    .having(having) \
    .order(options[:order]) \
    .readonly(false)
end
```

tagged_with方法主要是通过joins taggings表来查数据,通过taggable_type来关联

现在来看下tagged_with('ruby, rails', :any => true)

```
[35] pry(main)> Topic.tagged_with('ruby, rails', :any => true)
  ActsAsTaggableOn::Tag Load (4.0ms)  SELECT `tags`.* FROM `tags` WHERE (lower(name) = 'ruby' OR lower(name) = 'rails')
  Topic Load (38.0ms)  SELECT DISTINCT topics.* FROM `topics` JOIN taggings topic_taggings_047dc82 ON topic_taggings_047dc82.taggable_id = topics.id AND topic_taggings_047dc82.taggable_type = 'Topic' WHERE (topic_taggings_047dc82.tag_id = 16 OR topic_taggings_047dc82.tag_id = 17)
=> [#<Topic id: 4, title: nil, created_at: "2014-01-06 01:48:27", updated_at: "2014-01-06 01:48:49">]
```

``` ruby
# get tags, drop out if nothing returned (we need at least one)
tags = if options.delete(:wild)
  ActsAsTaggableOn::Tag.named_like_any(tag_list)
else
  # named_any是通过where or条件来查找表的
  ActsAsTaggableOn::Tag.named_any(tag_list)
end

# 如果一个都没查到就返回空数组
return empty_result unless tags.length > 0

# 没有传:on参数的话,taggings_context就为空字符串
taggings_context = context ? "_#{context}" : ''

# 给tagings表加上表别名
taggings_alias   = adjust_taggings_alias(
  "#{alias_base_name[0..4]}#{taggings_context[0..6]}_taggings_#{sha_prefix(tags.map(&:name).join('_'))}"
)

# joins taggings表的条件
tagging_join  = "JOIN #{ActsAsTaggableOn::Tagging.table_name} #{taggings_alias}" +
                "  ON #{taggings_alias}.taggable_id = #{quote}#{table_name}#{quote}.#{primary_key}" +
                " AND #{taggings_alias}.taggable_type = #{quote_value(base_class.name)}"
tagging_join << " AND " + sanitize_sql(["#{taggings_alias}.context = ?", context.to_s]) if context

# joins taggings表之后给个条件,只要taggings表的tag_id匹配的就行
conditions << tags.map { |t| "#{taggings_alias}.tag_id = #{t.id}" }.join(" OR ")
select_clause = "DISTINCT #{table_name}.*" unless context and tag_types.one?

if owned_by
    tagging_join << " AND " +
        sanitize_sql([
            "#{taggings_alias}.tagger_id = ? AND #{taggings_alias}.tagger_type = ?",
            owned_by.id,
            owned_by.class.base_class.to_s
        ])
end

joins << tagging_join
```
tagged_with('ruby, rails', :any => true)的原理是这样的,先通过named_any查tags，传进来的tag_list只有有的就返回,然后去joins taggings表,joins的条件为taggings.taggable_id = topics.id, taggings.taggable_type = 'Topic',然后给这个joins传一个条件,就是taggings.tag_id是named_any查了之后存在的
