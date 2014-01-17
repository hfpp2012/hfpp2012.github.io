---
layout: post
title: "breadcrumbs_on_rails源码分析"
date: 2014-01-17 09:28
comments: true
categories: ruby on rails
---

+ [breadcrumbs_on_rails](https://github.com/weppos/breadcrumbs_on_rails)

这个gem是用来生成面包屑的

是这样来使用的

``` ruby config/routes.rb
G::Application.routes.draw do

  get "say/hello"

  get "home/index"

  get "home/hello"

  root :to => "say#hello"

end
```

<!-- more -->

``` ruby
# say_controller.rb
class SayController < ApplicationController
  add_breadcrumb "root_path", :root_path
  def hello
    add_breadcrumb "home_hello_path", home_hello_path
  end
end

# home_controller.rb
class HomeController < ApplicationController
  add_breadcrumb "root_path", :root_path
  def index
    add_breadcrumb "home_hello", :home_hello_path
  end

  def hello
    add_breadcrumb "home_index", :home_index_path
  end
end

```

app/views/home/index.html.erb app/views/home/hello.html.erb app/views/say/hello.html.erb三个文件都添加这一行

``` ruby
<%= render_breadcrumbs :separator => ' / ' %>
```

效果是这样的

{% img /images/breadcrumbs_on_rails/root_path.png %}

{% img /images/breadcrumbs_on_rails/home_index_path.png %}

{% img /images/breadcrumbs_on_rails/home_hello_path.png %}

源码文件

```
lib
├── breadcrumbs_on_rails
│   ├── action_controller.rb  定义add_breadcrumb, render_breadcrumbs方法
│   ├── breadcrumbs.rb  渲染视图
│   ├── railtie.rb 加载controller时导入action_controller这个文件的内容
│   └── version.rb
└── breadcrumbs_on_rails.rb
```

从breadcrumbs_on_rails的使用中可以知道,定义了add_breadcrumb,和render_breadcrumbs这样的helper方法

``` ruby
helper          HelperMethods
helper_method   :add_breadcrumb, :breadcrumbs
```

render_breadcrumbs方法正是定义在HelperMethods这个module中

``` ruby
protected
def add_breadcrumb(name, path = nil, options = {})
  self.breadcrumbs << Breadcrumbs::Element.new(name, path, options)
end

def breadcrumbs
  @breadcrumbs ||= []
end

module ClassMethods
  def add_breadcrumb(name, path = nil, filter_options = {})
    # This isn't really nice here
    # 这个方法先不要管,没有用到第三个参数
    if eval = Utils.convert_to_set_of_strings(filter_options.delete(:eval), %w(name path))
      name = Utils.instance_proc(name) if eval.include?("name")
      path = Utils.instance_proc(path) if eval.include?("path")
    end

    # 重点在这一行,也就是说还是会执行protected中的add_breadcrumb方法
    before_filter(filter_options) do |controller|
      controller.send(:add_breadcrumb, name, path)
    end
  end
end
```

有两个add_breadcrumb方法,一个是controller中的类方法,一个是protected的

来看下Breadcrumbs::Element.new(name, path, options)

``` ruby
class Element

  attr_accessor :name
  attr_accessor :path
  attr_accessor :options

  # name和options分别对应Breadcrumbs的名称和路径
  # 可以不传path
  def initialize(name, path = nil, options = {})
    self.name     = name
    self.path     = path
    self.options  = options
  end
end
```

也就是说add_breadcrumb主要是是生成一个Element,主要的处理工作在render_breadcrumbs

render_breadcrumbs应该会生成a标签的,来看下代码

``` ruby
def render_breadcrumbs(options = {}, &block)
  # 默认情况下我们没有传:builder这个参数,先不管
  # 所有新添加的Breadcrumb都会放到breadcrumbs中
  builder = (options.delete(:builder) || Breadcrumbs::SimpleBuilder).new(self, breadcrumbs, options)
  content = builder.render.html_safe
  if block_given?
    capture(content, &block)
  else
    content
  end
end

class Builder

  # context 为[ActionView::Base]视图上下文
  # elements会存add_breadcrumb添加的breadcrumb
  def initialize(context, elements, options = {})
    @context  = context
    @elements = elements
    @options  = options
  end

  # 由子类来重载
  def render
    raise NotImplementedError
  end


  protected

    def compute_name(element)
      case name = element.name
      when Symbol
        @context.send(name)
      when Proc
        name.call(@context)
      else
        # 如果存的是字符串就原样输出
        name.to_s
      end
    end

    def compute_path(element)
      case path = element.path
      when Symbol
        @context.send(path)
      when Proc
        path.call(@context)
      else
        @context.url_for(path)
      end
    end

end

class SimpleBuilder < Builder

  def render
    @elements.collect do |element|
      render_element(element)
    end.join(@options[:separator] || " &raquo; ")
  end

  def render_element(element)
    # 如果没存path的情况
    if element.path == nil
      content = compute_name(element)
    else
      # 关键是这一行生成a标签的
      content = @context.link_to_unless_current(compute_name(element), compute_path(element), element.options)
    end
    if @options[:tag]
      @context.content_tag(@options[:tag], content)
    else
      content
    end
  end

end
```

render_breadcrumbs是通过SimpleBuilder的render方法来渲染视图,而SimpleBuilder是继承自Builder的

@context.link_to_unless_current是通过这个来生成标签的
