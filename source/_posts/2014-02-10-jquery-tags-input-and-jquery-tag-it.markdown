---
layout: post
title: "jquery tags-input &amp; jquery tag-it"
date: 2014-02-10 10:17
comments: true
categories: ruby on rails
---

标签大家肯定很熟悉,在文章中你可能会使用标签,你或许看过土豆视频的标签，或许看过京东商品的标签。这节我们在rails中分别使用tag-it和tags-input这两种关于标签的库,这两种库可以和[Acts-as-taggable-on](http://yinsigan.github.io/blog/2014/01/04/acts-as-taggable-onyuan-ma-fen-xi-1/)结合使用

### 1. [tag-it](https://github.com/aehlke/tag-it)

这个库要利用jquery和jquery ui这两种库

jquery在rails默认是开启的,现在来开启jquery ui这个库

<!-- more -->

#### 1.1 使用```jquery-ui-rails```这个gem

``` ruby Gemfile
group :assets do
  gem 'jquery-ui-rails'
end
```

``` javascript application.js
//= require jquery.ui.all
```

``` css application.css
*= require jquery.ui.all
```


#### 1.2 下载tag-it中tag-it.js和tagit.ui-zendesk.css和jquery.tagit文件

在layouts中引用这些文件

``` ruby application.html.erb
<%= javascript_include_tag "tag-it" %>
<%= stylesheet_link_tag "tagit.ui-zendesk" %>
<%= stylesheet_link_tag    "jquery.tagit", :media => "all" %>
```

#### 1.3 使用这个库

``` ruby
<input type="text" name="" value="" id="myTags"/>

<script type="text/javascript">
    $(document).ready(function() {
      $("#myTags").tagit({
        afterTagAdded: function() {
          // do something special
          $(this).attr('value', $(this).tagit("assignedTags").join(" "));
        },

        afterTagRemoved: function() {
          // do something special
          // $(this).attr('value', $(this).tagit("assignedTags").join(" "));
          $(this).attr('value', $(this).tagit("assignedTags").join(" "));
        },
      });
    });
</script>
```

### 2. [tags-input](https://github.com/xoxco/jQuery-Tags-Input)

这个库比较简单使用

#### 2.1 下载源文件, jquery.tagsinput.js和jquery.tagsinput.css

``` ruby
<%= stylesheet_link_tag "jquery.tagsinput", :media => "all" %>
<%= stylesheet_link_tag "jquery.tagsinput" %>
```

#### 2.2 使用

``` ruby
<input name="tags" id="tags" value="foo,bar,baz" />

$('#tags').tagsInput({
  onChange: function() {
    $(this).attr("value", $(this).val());
  }
});
```
