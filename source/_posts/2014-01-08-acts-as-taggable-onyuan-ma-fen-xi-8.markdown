---
layout: post
title: "acts-as-taggable-on源码分析(八)"
date: 2014-01-08 14:39
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

这节我们主要来研究Collection,主要是tag_counts的用法

<!-- more -->

```
Topic.tag_counts
=> SELECT tags.*, taggings.tags_count AS count FROM `tags` JOIN (SELECT taggings.tag_id, COUNT(taggings.tag_id) AS tags_count FROM `taggings` INNER JOIN topics ON topics.id = taggings.taggable_id WHERE (taggings.taggable_type = 'Topic' AND taggings.context = 'tags') AND (taggings.taggable_id IN(4,6,7)) GROUP BY taggings.tag_id HAVING COUNT(taggings.tag_id) > 0) AS taggings ON taggings.tag_id = tags.id
```

tag_counts方法定义在lib/acts-as-taggable-on/collection.rb文件中

``` ruby
tag_types.map(&:to_s).each do |tag_type|
  class_eval <<-RUBY, __FILE__, __LINE__ + 1
    def self.#{tag_type.singularize}_counts(options={})
      tag_counts_on('#{tag_type}', options)
    end

    def #{tag_type.singularize}_counts(options = {})
      tag_counts_on('#{tag_type}', options)
    end

    def top_#{tag_type}(limit = 10)
      tag_counts_on('#{tag_type}', :order => 'count desc', :limit => limit.to_i)
    end

    def self.top_#{tag_type}(limit = 10)
      tag_counts_on('#{tag_type}', :order => 'count desc', :limit => limit.to_i)
    end
  RUBY
end
```

可见,tag_counts即可以是实例方法,也可以是类方法(带self)

``` ruby
def tag_counts_on(context, options = {})
  all_tag_counts(options.merge({:on => context.to_s}))
end

def all_tag_counts(options = {})
  # 验证参数
  options.assert_valid_keys :start_at, :end_at, :conditions, :at_least, :at_most, :order, :limit, :on, :id

  scope = {}

  # 如果有条件就加入
  options[:conditions] = sanitize_sql(options[:conditions]) if options[:conditions]

  # 创建的开始时间
  start_at_conditions = sanitize_sql(["#{ActsAsTaggableOn::Tagging.table_name}.created_at >= ?", options.delete(:start_at)]) if options[:start_at]
  # 创建的结束时间
  end_at_conditions   = sanitize_sql(["#{ActsAsTaggableOn::Tagging.table_name}.created_at <= ?", options.delete(:end_at)])   if options[:end_at]

  # taggable和context的查询条件
  taggable_conditions  = sanitize_sql(["#{ActsAsTaggableOn::Tagging.table_name}.taggable_type = ?", base_class.name])
  taggable_conditions << sanitize_sql([" AND #{ActsAsTaggableOn::Tagging.table_name}.taggable_id = ?", options.delete(:id)])  if options[:id]
  taggable_conditions << sanitize_sql([" AND #{ActsAsTaggableOn::Tagging.table_name}.context = ?", options.delete(:on).to_s]) if options[:on]

  # 查taggings表的条件
  tagging_conditions = [
    taggable_conditions,
    scope[:conditions],
    start_at_conditions,
    end_at_conditions
  ].compact.reverse

  # 查tags表的条件
  tag_conditions = [
    options[:conditions]
  ].compact.reverse

  # 这个table_name是topics
  taggable_join = "INNER JOIN #{table_name} ON #{table_name}.#{primary_key} = #{ActsAsTaggableOn::Tagging.table_name}.taggable_id"
  taggable_join << " AND #{table_name}.#{inheritance_column} = '#{name}'" unless descends_from_active_record? # Current model is STI descendant, so add type checking to the join condition

  tagging_joins = [
    taggable_join,
    scope[:joins]
  ].compact

  tag_joins = [
  ].compact

  # 用select选择列
  tagging_scope = ActsAsTaggableOn::Tagging.select("#{ActsAsTaggableOn::Tagging.table_name}.tag_id, COUNT(#{ActsAsTaggableOn::Tagging.table_name}.tag_id) AS tags_count")
  tag_scope = ActsAsTaggableOn::Tag.select("#{ActsAsTaggableOn::Tag.table_name}.*, #{ActsAsTaggableOn::Tagging.table_name}.tags_count AS count").order(options[:order]).limit(options[:limit])

  # 上面的select加join操作
  tagging_joins.each      { |join|      tagging_scope = tagging_scope.joins(join)      }
  # 上面的select加where条件操作
  tagging_conditions.each { |condition| tagging_scope = tagging_scope.where(condition) }

  tag_joins.each          { |join|      tag_scope     = tag_scope.joins(join)          }
  tag_conditions.each     { |condition| tag_scope     = tag_scope.where(condition)     }

  # 最少几个tag
  at_least  = sanitize_sql(["COUNT(#{ActsAsTaggableOn::Tagging.table_name}.tag_id) >= ?", options.delete(:at_least)]) if options[:at_least]
  # 最多几个tag
  at_most   = sanitize_sql(["COUNT(#{ActsAsTaggableOn::Tagging.table_name}.tag_id) <= ?", options.delete(:at_most)]) if options[:at_most]
  having    = ["COUNT(#{ActsAsTaggableOn::Tagging.table_name}.tag_id) > 0", at_least, at_most].compact.join(' AND ')

  # 分组的列
  group_columns = "#{ActsAsTaggableOn::Tagging.table_name}.tag_id"

  scoped_select = "#{table_name}.#{primary_key}"
  select_query = "#{select(scoped_select).to_sql}"

  # 查找topics表中的所有id
  res = ActiveRecord::Base.connection.select_all(select_query).map { |item| item.values }.flatten.compact.join(",")
  res = "NULL" if res.blank?

  # 用where in语句来找
  tagging_scope = tagging_scope.where("#{ActsAsTaggableOn::Tagging.table_name}.taggable_id IN(#{res})")
  # 分组
  tagging_scope = tagging_scope.group(group_columns).having(having)

  # 关联taggings和tags表,查找tags表
  tag_scope = tag_scope.joins("JOIN (#{tagging_scope.to_sql}) AS #{ActsAsTaggableOn::Tagging.table_name} ON #{ActsAsTaggableOn::Tagging.table_name}.tag_id = #{ActsAsTaggableOn::Tag.table_name}.id")
  tag_scope
end
```

如果是实例的tag_counts方法呢

``` ruby
def tag_counts_on(context, options={})
  self.class.tag_counts_on(context, options.merge(:id => id))
end
```

多传了一个id的参数

``` ruby
def all_tag_counts(options = {})
  ...
  # 这一行或被执行在查数据时就会查特定的topic了
  taggable_conditions << sanitize_sql([" AND #{ActsAsTaggableOn::Tagging.table_name}.taggable_id = ?", options.delete(:id)])  if options[:id]
  ...
end
```

collection.rb文件中还有all_tags的方法,跟all_tag_counts差不多,可以自己研究一下
