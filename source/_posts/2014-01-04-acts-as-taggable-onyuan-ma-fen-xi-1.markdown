---
layout: post
title: "acts-as-taggable-on源码分析(一)"
date: 2014-01-04 14:19
comments: true
categories: ruby on rails
---

贴标签就要用它了,在很多电商或新闻网站经常能看到这种标签,是用来区别和分类的。

代码结构

<!-- more -->
```
lib
├── acts_as_taggable_on
│   ├── acts_as_taggable_on
│   │   ├── cache.rb   缓存相关
│   │   ├── collection.rb
│   │   ├── compatibility.rb has_many的重新定义
│   │   ├── core.rb 内核
│   │   ├── dirty.rb 脏数据的处理
│   │   ├── ownership.rb
│   │   └── related.rb 找有相关tag对象
│   ├── taggable.rb   可以打标签的对象
│   ├── tagger.rb     tag的拥有者
│   ├── tagging.rb    存标签的关系
│   ├── tag_list.rb   对标签的生成,添加,删除等
│   ├── tag.rb  存标签的内容
│   ├── tags_helper.rb  一个自定义的helper
│   ├── utils.rb  一些数据库的判断,字符的处理等
│   └── version.rb
├── acts-as-taggable-on.rb 主程序

```

数据表存储结构

taggings表

{% img /images/acts_as_taggable_on/taggings.png%}

taggs表

{% img /images/acts_as_taggable_on/tags.png%}

taggings表存的是关联的数据,假如我们给一个topic打了ruby和rails这两个tag,这个表会存topic的id,而ruby和rails的值是存在tags表中的,taggings表通过tag_id关联到tags表中的

### configuration

``` ruby
require "active_record"
require "active_record/version"
# heper要用到
require "action_view"

# 加密用到
require "digest/sha1"

# 把当前目录加到ruby的默认LOAD路径
$LOAD_PATH.unshift(File.dirname(__FILE__))

module ActsAsTaggableOn
  mattr_accessor :delimiter
  @@delimiter = ','

  mattr_accessor :force_lowercase
  @@force_lowercase = false

  mattr_accessor :force_parameterize
  @@force_parameterize = false

  mattr_accessor :strict_case_match
  @@strict_case_match = false

  mattr_accessor :remove_unused_tags
  self.remove_unused_tags = false

  def self.glue
    # 如果是数组取第一个
    delimiter = @@delimiter.kind_of?(Array) ? @@delimiter[0] : @@delimiter
    # 把分隔符加一个空格
    delimiter.ends_with?(" ") ? delimiter : "#{delimiter} "
  end

  def self.setup
    yield self
  end
end


require "acts_as_taggable_on/utils"

require "acts_as_taggable_on/taggable"
require "acts_as_taggable_on/acts_as_taggable_on/compatibility"
require "acts_as_taggable_on/acts_as_taggable_on/core"
require "acts_as_taggable_on/acts_as_taggable_on/collection"
require "acts_as_taggable_on/acts_as_taggable_on/cache"
require "acts_as_taggable_on/acts_as_taggable_on/ownership"
require "acts_as_taggable_on/acts_as_taggable_on/related"
require "acts_as_taggable_on/acts_as_taggable_on/dirty"

require "acts_as_taggable_on/tagger"
require "acts_as_taggable_on/tag"
require "acts_as_taggable_on/tag_list"
require "acts_as_taggable_on/tags_helper"
require "acts_as_taggable_on/tagging"

# 把当前目录从加载目录去掉
$LOAD_PATH.shift


if defined?(ActiveRecord::Base)
  ActiveRecord::Base.extend ActsAsTaggableOn::Compatibility
  ActiveRecord::Base.extend ActsAsTaggableOn::Taggable
  ActiveRecord::Base.send :include, ActsAsTaggableOn::Tagger
end

if defined?(ActionView::Base)
  ActionView::Base.send :include, ActsAsTaggableOn::TagsHelper
end
```

require_relative从相对目录(当前目录)里加载

mattr_accessor属于一个ActsAsTaggableOn module,全部的model包括后面定义的tag.rb, tagging.rb等文件都包括在ActsAsTaggableOn module中,这样有一个好处,避免跟你原有的model同名

ActsAsTaggableOn定义了五个参数

remove_unused_tags: 假如这个tag(ruby, rails)已经没有使用了,就删除它

force_lowercase: 在存储tag的时候全部用小写字母存

force_parameterize: tag被参数化存储

strict_case_match: 查找的时候强制大小写敏感

delimiter: tag的分隔符

看下一节[Acts-as-taggable-on源码分析(二)](/blog/2014/01/04/acts-as-taggable-onyuan-ma-fen-xi-2/)
