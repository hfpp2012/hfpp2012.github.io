---
layout: post
title: "嵌入式表单与多图片上传"
date: 2014-02-19 12:17
comments: true
categories: ruby on rails
---

## 1.嵌入式表单

+ [accepts_nested_attributes_for](http://api.rubyonrails.org/classes/ActiveRecord/NestedAttributes/ClassMethods.html)

当面对越来越复杂的业务逻辑时,你一定遇到过一个表单不止涉及到一个model,这个时候会添加或修改其他model的数据,例如下面的例子就是在创建一个用户(user),顺便发一篇文章(post)

<!-- more -->

``` ruby
class User < ActiveRecord::Base
  attr_accessible :name, :posts_attributes

  has_many :posts

  accepts_nested_attributes_for :posts

end

class Post < ActiveRecord::Base
  attr_accessible :name

  belongs_to :user
end
```

主要是利用了accepts_nested_attributes_for这个方法,它让user可以直接访问跟它有关系的(has_many posts)的posts中的数据,也就是说一旦更改或添加user时也会去处理相关联的posts中的数据

假如我们在创建posts的时候默认至少可以添加三篇posts

``` ruby app/controller/users_controller.rb
def new
  @user = User.new
  3.times { @user.posts.build }
end
```

3.times { @user.posts.build }就是代表初始化了三个posts

然后在view里可以创建三个posts

``` ruby app/views/users/_form.html.erb
<div id="add_posts">
  <% @user.posts.each do |post| %>
    <%= fields_for "user[posts_attributes][]", post do |task_form| %>
      <p>
        Task: <%= task_form.text_field :name %>
      </p>
    <% end %>
  <% end %>
</div>
```

fields_for的作用是生成嵌入式表单相关的域(input),关键在于```user[posts_attributes][]```

它生成的input是这样的

``` html
<p>Task: <input id="user_posts_attributes__name" name="user[posts_attributes][][name]" size="30" type="text"><p>
```

input中name属性的值必须正确

它有什么讲究呢,这就要看[accepts_nested_attributes_for](http://api.rubyonrails.org/classes/ActiveRecord/NestedAttributes/ClassMethods.html)了

只能添加三个posts肯定不够,我们需要添加无数个posts

``` ruby app/views/users/_form.html.erb
<a href="javascript:void(0)" id="add">点击我添加一个post</a>

<script>

  $("#add").click(function() {
    $('<p>Task: <input id="user_posts_attributes__name" name="user[posts_attributes][][name]" size="30" type="text"><p>').appendTo("#add_posts");
  })

</script>
```

试一下吧

## 2. 多图片上传

先说嵌入式表单,是因为在多图片上传中我们要应用到它的原理

图片上传主要是用到[carrierwave](blog/2014/02/12/carrierwavetu-pian-shang-chuan-de-jian-dan-shi-yong/)

这里我们要用到[多态](blog/2013/12/10/rails-association/),首先我们创建一个model叫attachment,这个model是用作存上传图片的信息吧

它主要包括三个字段attachmentable_id, attachmentable_type, image

``` ruby
class Attachment < ActiveRecord::Base
  attr_accessible :attachmentable_id, :attachmentable_type, :image, :image_cache

  belongs_to :attachmentable, :polymorphic => true

  mount_uploader :image, AvatarUploader
end
```

有了attachmentable这个[多态](blog/2013/12/10/rails-association/),以后要用到图片上传的model都可以跟attachment关联起来了

假如user这个model就要用到多图片上传

``` ruby
class User < ActiveRecord::Base
  has_many :attachments, :as => :attachmentable
  accepts_nested_attributes_for :attachments
end
```

还是跟上面一样,假设默认最少可以添加三个图片

``` ruby app/controllers/users_controller.rb
def new
  @user = User.new
  3.times { @user.attachments.build }
end
```

然后,view是这样的

``` ruby app/views/users/_form.html.erb
<div id="add_posts">
  <% @user.attachments.each do |attachment| %>
    <%= fields_for "user[attachments_attributes][]", attachment do |task_form| %>
      <p>
        attachment: <%= task_form.file_field :image %>
        <%= task_form.hidden_field :image_cache %>
      </p>
    <% end %>
  <% end %>
</div>
```

记得form表单添加这个属性entrype="multipart/form-data"

``` ruby
<%= form_for(@user, html: {enctype: "multipart/form-data"}) do |f| %>
```

可以添加无数个图片

``` ruby
<a href="javascript:void(0)" id="add">点击我添加一个attachment</a>

$("#add").click(function() {
  $('<p>attachment: <input id="user_attachments_attributes_image" name="user[attachments_attributes][][image]" size="30" type="file"><p>').appendTo("#add_posts");
})
```
