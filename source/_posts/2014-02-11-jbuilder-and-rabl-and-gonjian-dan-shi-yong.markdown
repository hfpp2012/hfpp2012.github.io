---
layout: post
title: "jbuilder &amp; rabl &amp; gon简单使用"
date: 2014-02-11 14:56
comments: true
categories: ruby on rails
---

这节介绍几个和json处理有关的gem(大部分的内容跟railscasts的差不多,在这里只是进行汇总和总结,方便查看)

用rails在默认情况下就能产生json

是这样的

``` ruby app/controllers/articles_controller.rb
# GET /articles/1
# GET /articles/1.json
def show
  @article = Article.find(params[:id])
  respond_to do |format|
    format.html # new.html.erb
    format.json { render json: @article }
  end
end
```

<!-- more -->

看到那个/articles/1.json没有,默认情况下你在浏览器访问/articles/1.json就能返回json了,是这个format.json { render json: @article }

@article拥有as_json这样的方法可以格式化json的输出,具体的使用可以查看[rails as_json api](http://api.rubyonrails.org/classes/ActiveModel/Serializers/JSON.html )

这种方法还是有局限性,不能灵活自定义json的输出,例如不能改写字段的名字

先删除掉respond_to那段代码

下面我们来介绍两个可以产生json的gem:[jbuilder](https://github.com/rails/jbuilder)和[rabl](https://github.com/nesquena/rabl)

### 1. [jbuilder](https://github.com/rails/jbuilder)

#### 1.1 这个gem在rails中是被注释掉的,开启它就可以用了

``` ruby Gemfile
gem 'jbuilder'
```

#### 1.2 新增jbuilder的模板文件

##### 1.2.1 最简单的方式

``` ruby app/views/articles/show.json.jbuilder
json.id @article.id
json.title @article.title
```

输出一个json,键为id和title(json之后的词)

##### 1.2.2 不写键名,默认用字段名

``` ruby app/views/articles/show.json.jbuilder
json.extract! @article, :id, :title, :updated_at
```

输出的结果为

``` ruby
{"id":1,"title":"dddddd","updated_at":"2014-02-09T01:58:47Z"}
```

##### 1.2.3 extract!的简写方式

``` ruby app/views/articles/show.json.jbuilder
json.(@article, :id, :title, :updated_at)
```

##### 1.2.4 自己加一个key

``` ruby
json.edit_url edit_article_url(@article) if current_user.admin?
```

##### 1.2.5 关联一个对象

如果article属于user

``` ruby
json.user @article.user, :id, :name
```

输出为

``` ruby
{"id":1,"title":"dddddd","updated_at":"2014-02-09T01:58:47Z","user":{"id":1,"name":"xxx"}}
```

jbuilder还有许多其他功能,例如partial,array等

### 2. [rabl](https://github.com/nesquena/rabl)

rabl相比于jbuilder,个人觉得更具有可读性

#### 2.1 安装

``` ruby Gemfile
gem 'rabl'
```

#### 2.2 使用

它的模版更简单

##### 2.2.1 最简单的方式

``` ruby
object @article
attributes :id, :title
```

##### 2.2.2 添加一个key

``` ruby
node(:edit_url) { "..." }
```

#### 2.2.3 关联

``` ruby
child :user do
  attributes :id, :name
end

child :comments do
  attributes :id, :content
end
```

#### 2.2.4 partial

``` ruby index.json.rabl
collection @articles
extends "articles/show"
```

#### 2.2.5 去掉Root Nodes

也就是去掉那个"article"

``` ruby config/initializers/rabl_config.rb
Rabl.configure do |config|
  config.include_json_root = false
end
```

#### 2.2.6 Embedding JSON in a Web Page

``` ruby
<div id="articles" data-articles="<%= render(template: "articles/index.json.rabl") %>" >
```

rabl github上的readme介绍了更好的用法,可以自定于key的名字,还有很多的配置选项

### 3. [gon](https://github.com/gazay/gon)

之所以要介绍这个gem,并不是因为它是产生json的,而它能把前面两个gem结合起来

它的作用是传递变量给javascript(Passing Data to JavaScript)

#### 3.1 安装

``` ruby Gemfile
gem 'gon'
```

``` ruby app/views/layouts/application.html.erb
<head>
  <title>Store</title>
  <%= include_gon %>
  <%= stylesheet_link_tag    "application", media: "all" %>
  <%= javascript_include_tag "application" %>
  <%= csrf_meta_tag %>
</head>
```

#### 3.2 简单使用

``` ruby app/controllers/articles_controlle.rb
def index
  @articles = Article.all
  gon.articles = Article.all
end
```

``` ruby app/assets/javascripts/articles.js.coffee
Jquery ->
  console.log gon.articles
```

#### 3.3 和rabl结合

``` ruby app/controllers/articles_controlle.rb
def index
  @articles = Article.all
  gon.rabl "app/views/articles/index.json.rabl", as: "articles"
end
```

``` ruby app/assets/javascripts/articles.js.coffee
alert gon.articles if gon
```
