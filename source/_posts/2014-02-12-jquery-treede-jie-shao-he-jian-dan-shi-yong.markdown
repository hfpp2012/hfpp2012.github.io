---
layout: post
title: "jquery tree的介绍和简单使用"
date: 2014-02-12 11:24
comments: true
categories: jquery plugins
---

这节来介绍树(tree)形菜单的使用

首先通过介绍[jquery treeview](https://github.com/jzaefferer/jquery-treeview)的使用来了解树形菜单,最后会介绍几种jquery tree的特性

<!-- more -->

### 1. jquery treeview

这个tree支持引导线的显示,支持图片,支持ajax,给我感觉中规中距

#### 1.1 安装

下载所有的图片文件,还有jquery.treeview.js和jquery.treeview.css

``` ruby
<%= javascript_link_tag "jquery.treeview" %>
<%= javascript_include_tag "jquery.treeview" %>
```

Note: 如果发现图片没有找到,应该根据你图片的存储位置,更改jquery.treeview.css中的background-image中的路径,直到正确为止.

#### 1.2 添加html

html为ul li结构,只要按照类似的结构姐织代码就可以,如果有使用ancestry这种gem,可以用它来生成html代码

``` html
<ul id="browser" class="filetree treeview-famfamfam">
  <li><span class="folder">Folder 1</span>
  <ul>
    <li><span class="folder">Item 1.1</span>
    <ul>
      <li><span class="file">Item 1.1.1</span></li>
    </ul>
    </li>
    <li><span class="folder">Folder 2</span>
    <ul>
      <li><span class="folder">Subfolder 2.1</span>
      <ul id="folder21">
        <li><span class="file">File 2.1.1</span></li>
        <li><span class="file">File 2.1.2</span></li>
      </ul>
      </li>
      <li><span class="folder">Subfolder 2.2</span>
      <ul>
        <li><span class="file">File 2.2.1</span></li>
        <li><span class="file">File 2.2.2</span></li>
      </ul>
      </li>
    </ul>
    </li>
    <li class="closed"><span class="folder">Folder 3 (closed at start)</span>
    <ul>
      <li><span class="file">File 3.1</span></li>
    </ul>
    </li>
    <li><span class="file">File 4</span></li>
  </ul>
  </li>
</ul>
```

#### 1.3 添加javascript

只要一个函数

``` javascript
$("#browser").treeview({
    toggle: function() {
        console.log("%s was toggled.", $(this).find(">span").text());
    }
});
```

#### 1.4 有图片的

``` html
<ul id="browser" class="filetree">
  <li><img src="folder.gif" /> 123</span>
<ul>
  <li>blabla <img src="file.gif" /></li>
</ul>
</li>
<li><img src="folder.gif" />
<ul>
  <li><img src="folder.gif" />
  <ul id="folder21">
    <li><img src="file.gif" /> more text</li>
    <li>and here, too<img src="file.gif" /></li>
  </ul>
  </li>
  <li><img src="file.gif" /></li>
</ul>
</li>
<li class="closed">this is closed! <img src="folder.gif" />
<ul>
  <li><img src="file.gif" /></li>
</ul>
</li>
<li><img src="file.gif" /></li>
</ul>
```

### 2. 几种jquery tree

#### 2.1 [jtree](http://jtree.tk/index)

+ 简单使用
+ 支持拖拽
+ 插入新元素
+ 事件功能

它的功能比treeview强大些

#### 2.2 [jqTree](https://github.com/mbraak/jqTree)

+ 界面简洁清爽
+ 帮助文档详细
+ 支持从JSON创建树
+ 支持拖拽,拖拽动画效果显示良好
+ 功能强大,可设置参数和事件多

如果需要从JSON获得数据来建tree,选择这款不错

#### 2.3 [zTree](http://www.ztree.me/hunter/zTree.html)

zTree的功能最强大了,如果要做复杂的tree就用它了
