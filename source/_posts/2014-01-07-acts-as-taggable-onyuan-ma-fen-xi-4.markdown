---
layout: post
title: "acts-as-taggable-on源码分析(四)"
date: 2014-01-07 15:11
comments: true
categories: ruby on rails
---

+ [Acts-as-taggable-on源码分析(一)](/blog/2014/01/04/acts-as-taggable-onyuan-ma-fen-xi-1/)
+ [Acts-as-taggable-on源码分析(二)](/blog/2014/01/04/acts-as-taggable-onyuan-ma-fen-xi-2/)
+ [Acts-as-taggable-on源码分析(三)](/blog/2014/01/06/acts-as-taggable-onyuan-ma-fen-xi-3/)

这一节我们继续来研究tag_list,tag_list.add,tag_list.remove这些方法用法和原理

core.rb文件定义了tag_list,虽然不是直接定义

<!-- more -->

``` ruby
# 当ActsAsTaggableOn::Taggable::Core被included时就会执行这个方法
base.initialize_acts_as_taggable_on_core

def initialize_acts_as_taggable_on_core
  # taggable_mixin是一个动态创建的module
  include taggable_mixin
  tag_types.map(&:to_s).each do |tags_type|
    # 转成单数
    tag_type         = tags_type.to_s.singularize
    # 可能会输出类似:tag_taggings这样的东西
    context_taggings = "#{tag_type}_taggings".to_sym
    # 可能会输出类似:tags这样的东西
    context_tags     = tags_type.to_sym
    # 按照taggings表的id排序
    taggings_order   = (preserve_tag_order? ? "#{ActsAsTaggableOn::Tagging.table_name}.id" : [])

    class_eval do
      # has_many_with_compatibility是has_many修改之后的
      # topic has_many tag_taggings
      has_many_with_compatibility context_taggings, :as => :taggable,
                                  :dependent => :destroy,
                                  :class_name => "ActsAsTaggableOn::Tagging",
                                  :order => taggings_order,
                                  :conditions => ["#{ActsAsTaggableOn::Tagging.table_name}.context = (?)", tags_type],
                                  :include => :tag

      # topic has_many tags
      has_many_with_compatibility context_tags, :through => context_taggings,
                                  :source => :tag,
                                  :class_name => "ActsAsTaggableOn::Tag",
                                  :order => taggings_order

    end

    taggable_mixin.class_eval <<-RUBY, __FILE__, __LINE__ + 1
      def #{tag_type}_list
        tag_list_on('#{tags_type}')
      end

      def #{tag_type}_list=(new_tags)
        set_tag_list_on('#{tags_type}', new_tags)
      end

      def all_#{tags_type}_list
        all_tags_list_on('#{tags_type}')
      end
    RUBY
  end
end

def taggable_mixin
  @taggable_mixin ||= Module.new
end
```

{tag_type}_list会被代替成tag_list

现在要看的是tag_list_on还有set_tag_list_on这两个方法

``` ruby
def tag_list_on(context)
  # 把类似tags,skills这类的东西添加到custom_contexts
  add_custom_context(context)
  tag_list_cache_on(context)
end

# 把类似tags,skills这类的东西添加到custom_contexts
def add_custom_context(value)
  custom_contexts << value.to_s unless custom_contexts.include?(value.to_s) or self.class.tag_types.map(&:to_s).include?(value.to_s)
end

attr_writer :custom_contexts
def custom_contexts
  @custom_contexts ||= []
end

def tag_list_cache_on(context)
  # variable_name可能会等于@tag_list这样的东西
  variable_name = "@#{context.to_s.singularize}_list"
  # 如果已经定义了@tag_list这个变量,并且有值存在,这返回它
  return instance_variable_get(variable_name) if instance_variable_defined?(variable_name) && instance_variable_get(variable_name)
  # 没有的话就给@tag_list设值
  instance_variable_set(variable_name, ActsAsTaggableOn::TagList.new(tags_on(context).map(&:name)))
end
```

来看ActsAsTaggableOn::TagList这个模块

``` ruby
require 'active_support/core_ext/module/delegation'

module ActsAsTaggableOn
  class TagList < Array
    attr_accessor :owner

    # 初始化方法,ActsAsTaggableOn::TagList.new执行的
    def initialize(*args)
      add(*args)
    end

    # 从字符中近回一个新的数组,例如"rails, ruby" => ['rails', 'ruby']
    def self.from(string)
      # ActsAsTaggableOn.glue可能会是', ',字符串没有join方法
      string = string.join(ActsAsTaggableOn.glue) if string.respond_to?(:join)

      # tap会处理tag_list然后把它返回
      new.tap do |tag_list|
        # 复制一份
        string = string.to_s.dup

        # 可能会是","
        # 这一段代码的意思就是把类似"rails, ruby"用正则分析之后再用split变成数组给add方法处理
        d = ActsAsTaggableOn.delimiter
        d = d.join("|") if d.kind_of?(Array) 
        string.gsub!(/(\A|#{d})\s*"(.*?)"\s*(#{d}\s*|\z)/) { tag_list << $2; $3 }
        string.gsub!(/(\A|#{d})\s*'(.*?)'\s*(#{d}\s*|\z)/) { tag_list << $2; $3 }

        tag_list.add(string.split(Regexp.new d))
      end
    end

    # 添加tag
    def add(*names)
      # 这个方法在后面定义,这个主要是对复杂的参数进行处理和解析,如果是一般简单的参数直接返回
      extract_and_apply_options!(names)
      # 加上原先的
      concat(names)
      clean!
      self
    end

    # 删除tag
    def remove(*names)
      extract_and_apply_options!(names)
      # name存在才删除
      delete_if { |name| names.include?(name) }
      self
    end

    # 转成字符串输出
    def to_s
      # 锁住就复制一份,否则返回自身
      tags = frozen? ? self.dup : self
      # 去掉空白、重复、为空的情况
      tags.send(:clean!)

      tags.map do |name|
        d = ActsAsTaggableOn.delimiter
        d = Regexp.new d.join("|") if d.kind_of? Array
        name.index(d) ? "\"#{name}\"" : name
      end.join(ActsAsTaggableOn.glue)
    end

    private

    # 去掉空白、重复、为空的情况
    def clean!
      # 去掉空的
      reject!(&:blank?)
      # 去掉左右边界的留空
      map!(&:strip)
      # 如果要小写就转成小写字母
      map!{ |tag| tag.mb_chars.downcase.to_s } if ActsAsTaggableOn.force_lowercase
      map!(&:parameterize) if ActsAsTaggableOn.force_parameterize

      # 保证唯一
      uniq!
    end

    def extract_and_apply_options!(args)
      # 判断最后一个参数是不是数组,如果是的话就弹出来,否则返回一个空的hash
      # 在处理这样的参数传递会用到tag_list.add("Fun, Happy", :parse => true)
      options = args.last.is_a?(Hash) ? args.pop : {}
      # 如果key不是:parse会抛出异常
      options.assert_valid_keys :parse

      # 如果parse等于true,相当于传了"Fun, Happy", :parse => true这样的参数
      # 这个时候会把"Fun, Happy"送给from处理,也就是解析字符串以数组返回
      if options[:parse]
        args.map! { |a| self.class.from(a) }
      end

      # 不存在parse参数,如果是这样的参数"Fun", "Happy",直接返回就行了
      args.flatten!
    end
  end
end
```

ActsAsTaggableOn::Tagging本身继承自Array,from方法是解决字符串参数转成数组,add是添加tag，以数组形式返回,remove是移除tag,clean会清除数组中一些空的,重复的元素,to_s是把数组以字符串的形式返回

tag_list这个方法有一个cache的效果,具体体现在它会通过@tag_list来存储值,没有的话才找数据库的

