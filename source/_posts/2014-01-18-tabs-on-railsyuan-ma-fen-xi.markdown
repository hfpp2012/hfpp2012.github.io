---
layout: post
title: "tabs_on_rails源码分析"
date: 2014-01-18 14:38
comments: true
categories: ruby on rails
---

+ [tabs_on_rails](https://github.com/weppos/tabs_on_rails)
+ [breadcrumbs_on_rails源码分析](/blog/2014/01/17/breadcrumbs-on-railsyuan-ma-fen-xi/)

这个gem是用来生成tab用的,由于这个gem的源码结构的实现方法跟breadcrumbs_on_rails的有些相似,大家可以看下[breadcrumbs_on_rails源码分析](/blog/2014/01/17/breadcrumbs-on-railsyuan-ma-fen-xi/)这篇blog

是这样来使用的

<!-- more -->

``` ruby home/index.html.erb
<%= tabs_tag do |tab| %>
  <%= tab.say_hello 'say_hello', root_path %>
  <%= tab.home_index 'home_index', home_index_path %>
  <%= tab.home_hello 'home_hello', home_hello_path %>
<% end %>

<p>The name of current tab is <%= current_tab %>.</p>


<%= @a %>
```

``` ruby home_controller.rb
class HomeController < ApplicationController
  def index
    set_tab :home_index
    @a = current_tab?(:home_index)
  end

  def hello
  end
end
```

显示结果是这样的

{% img /images/tabs_on_rails/result.png %}

tabs_tag中定义了三个tab, 分别为say_hello, home_index, home_hello，如果要设置当前tab必须在控制器里设置

源码结构

```
lib
├── tabs_on_rails
│   ├── action_controller.rb  # 定义了controller可以使用的set_tab等方法
│   ├── railtie.rb # 启动文件
│   ├── tabs
│   │   ├── builder.rb  # 生成视图用的
│   │   └── tabs_builder.rb # 继承了builder.rb文件
│   ├── tabs.rb # 生成视图用的主要文件
│   └── version.rb
└── tabs_on_rails.rb # 主程序文件，包含一些require语句
```

先来分析set_tab这个方法,这个方法是属于controller的，这个方法是定义在action_controller.rb这个文件中的。
``` ruby

protected

def set_tab(name, namespace = nil)
  # tab_stack是专门放tab用的
  # 可以不传namespace(命名空间),默认为:default
  tab_stack[namespace || :default] = name
end

module ClassMethods

  def set_tab(*args)
    # extract_options!的意思是看*args最后一个是不是hash，如果是的话就弹(pop)出来
    options = args.extract_options!
    # 把剩下的参数给name, namespace
    name, namespace = args

    # 添加了一个before_filter过滤,执行的还是protected中的set_tab方法
    before_filter(options) do |controller|
      controller.send(:set_tab, name, namespace)
    end
  end
end
```

有两个set_tab的方法,一个是受保护的(protected),一个是类方法,也就是说一个在controller这个类中就可以执行，一个要在controller中的方法里执行。

由代码可知两个set_tab方法最终都是执行protected中的set_tab方法

在来看跟set_tab也是protected的方法

``` ruby
protected

# 返回当前的tab
def current_tab(namespace = nil)
  tab_stack[namespace || :default]
end

# 判断是否是当前的tab
def current_tab?(name, namespace = nil)
  current_tab(namespace).to_s == name.to_s
end

# @tab_stack是一个hash，key默认为:default(命名空间namespace)
def tab_stack
  @tab_stack ||= {}
end
```

我们来试验下tab_stack这个protected方法的效果

``` ruby home_controller.rb
class HomeController < ApplicationController
  def index
    set_tab :home_index
    @a = current_tab?(:home_index)
    @b = tab_stack
  end

end
```

``` ruby
...

<p>The name of current tab is <%= current_tab %>.</p>

<%= @a %>
<%= @b %>
```

将会输出

{% img /images/tabs_on_rails/tab_stack.png %}

而在这几个方法中只有current_tab, current_tab?是helper方法

``` ruby
included do
  extend        ClassMethods
  helper        HelperMethods
  helper_method :current_tab, :current_tab?
end
```

也就是说current_tab, current_tab?可以在view文件中直接使用

HelperMethods模组里定义了tabs_tag方法,这个方法是生成tab用的

``` ruby
def tabs_tag(options = {}, &block)
  Tabs.new(self, { :namespace => :default }.merge(options)).render(&block)
end
```

找到Tabs类,它定义在tabs.rb文件中。

``` ruby
require 'tabs_on_rails/tabs/builder'
require 'tabs_on_rails/tabs/tabs_builder'

...
```

先来看头两行，导入了两个模组,这两个模组是有继承关系的，来看下。

``` ruby lib/tabs/builder.rb
module TabsOnRails
  class Tabs

    class Builder

      # 初始化
      def initialize(context, options = {})
        # 视图上下文
        @context   = context
        # 命名空间
        @namespace = options.delete(:namespace) || :default
        # 其他选项
        @options   = options
      end

      # 判断是否是当前tab
      def current_tab?(tab)
        tab.to_s == @context.current_tab(@namespace).to_s
      end


      # 等子类实现的方法
      def tab_for(*args)
        raise NotImplementedError
      end

      # 等子类实现的方法
      def open_tabs(*args)
      end

      # 等子类实现的方法
      def close_tabs(*args)
      end

    end

  end
end
```

Builder定义了几个方法,大部分(tab_for, open_tabs, close_tabs)都是空的,等待子类来实现。

lib/tabs/tabs_builder.rb中定义的TabsBuilder就是来实现Builder的。

``` ruby lib/tabs/tabs_builder.rb
module TabsOnRails
  class Tabs

    class TabsBuilder < Builder

      # 生成带tab的html就是这个方法,会生成带a标签或span标签(当前tab)
      # tab是标签的名字,对应controller里的
      # name是a标签的内容, url_options是a标签的路径, item_options是其他选项,例如class等
      def tab_for(tab, name, url_options, item_options = {})
        # current_tab?这个方法在Builder实现的
        # 如果是当前tab,把class属性的值以空格分隔成数组,再加上一个"current"属性，再组成class
        item_options[:class] = item_options[:class].to_s.split(" ").push(@options[:active_class] || "current").join(" ") if current_tab?(tab)
        # 如果不是当前tab就用a标签,如果是就用span标签
        content = @context.link_to_unless(current_tab?(tab), name, url_options) do
          @context.content_tag(:span, name)
        end
        # 把a标签的内容放在li里
        @context.content_tag(:li, content, item_options)
     end

      # 外面包一层ul
      def open_tabs(options = {})
        # 产生一个空的ul标签
        @context.tag("ul", options, open = true)
      end

      def close_tabs(options = {})
        "</ul>".html_safe
      end

    end

  end
end
```

content_tag就是生成html用的,第一个参数是标签的名称(li, span, a),第二个参数是内容

看来生成tag的工作主要在TabsBuilder这个模组上了。

``` ruby
<%= tabs_tag do |tab| %>
  <%= tab.say_hello 'say_hello', root_path %>
  <%= tab.home_index 'home_index', home_index_path %>
  <%= tab.home_hello 'home_hello', home_hello_path %>
<% end %>

def tabs_tag(options = {}, &block)
  # 传默认的namespace为:default
  Tabs.new(self, { :namespace => :default }.merge(options)).render(&block)
end
```

来看下tabs.rb这个文件的内容

``` ruby
module TabsOnRails

  class Tabs

    # 指定默认的default_builder,也就是上面讲到的TabsBuilder
    mattr_accessor :default_builder
    @@default_builder = TabsBuilder

    # 初始化
    def initialize(context, options = {})
      # 视图上下文
      @context = context
      # 如果没有指定builder,就用默认的builder
      # 主要传给builder一个视图上下文就ok，builder会去绘制a或span标签的
      @builder = (options.delete(:builder) || self.class.default_builder).new(@context, options)
      @options = options
    end

    # arity是看有没有固定参数http://www.ruby-doc.org/core-2.1.0/Method.html
    %w(open_tabs close_tabs).each do |name|
      # 定义open_tabs和close_tabs方法
      define_method(name) do |*args|                      # def open_tabs(*args)
        # 取builder中的open_tabs和close_tabs方法
        method = @builder.method(name)                    #   method = @builder.method(:open_tabs)
        if method.arity.zero?                             #   if method.arity.zero?
          method.call                                     #     method.call
        else                                              #   else
          method.call(*args)                              #     method.call(*args)
        end                                               #   end
      end                                                 # end
    end

    # 当传tab.home_index就会执行这个方法,绘制a标签或span标签
    def method_missing(*args, &block)
      @builder.tab_for(*args, &block)
    end


    def render(&block)
      # 如果没有传block就报异常
      raise LocalJumpError, "no block given" unless block_given?

      # 把@options.dup复制一份
      options = @options.dup
      # 没有这两个open_tabs和close_tabs参数就为空
      open_tabs_options  = options.delete(:open_tabs)  || {}
      close_tabs_options = options.delete(:close_tabs) || {}

      # 输出html
      "".tap do |html|
        html << open_tabs(open_tabs_options).to_s
        # capture的用法见这里http://api.rubyonrails.org/classes/ActionView/Helpers/CaptureHelper.html#method-i-capture
        html << @context.capture(self, &block)
        html << close_tabs(close_tabs_options).to_s
      end.html_safe
    end

  end

end
```
