---
layout: post
title: "CarrierWave图片上传的简单使用"
date: 2014-02-12 16:02
comments: true
categories: ruby on rails
---

使用CarrierWave这个gem来完成图片上传和使用mini_magick这个gem来处理上传的图片

## 1. [CarrierWave](https://github.com/carrierwaveuploader/carrierwave)

### 1.1 安装

``` ruby
gem 'carrierwave'
```

<!-- more -->

### 1.2 生成uploader

```
rails generate uploader Avatar

# 这是生成的文件
app/uploaders/avatar_uploader.rb
```

### 1.3 添加图片的字段

```
rails g migration add_image_to_articles image:string
```

image存储是文件名

### 1.4 挂载那个uploader

``` ruby
class Article < ActiveRecord::Base
  attr_accessible :content, :title, :image, :remote_image_url
  mount_uploader :image, AvatarUploader
end
```

### 1.5 给表单加上图片上传的域

``` html
<div class="field">
  <%= f.label :image %><br />
  <%= f.file_field :image %>
  <%= f.hidden_field :image_cache %>
</div>
```

### 1.6 引用上传的图片地址

``` ruby app/views/articles/show.html.erb
<%= image_tag @article.image_url.to_s %>
```

这样就可以上传图片了,默认图片是存储在public那个文件夹中

### 1.7 远程图片

``` ruby app/views/articles/show.html.erb
<div class="field">
  <%= f.label :remote_image_url, "or image URL" %>
  <%= f.text_field :remote_image_url %>
</div>
```

## 2. MiniMagick处理图片

### 2.1 安装

``` ruby
gem 'mini_magick'
```

``` ruby app/uploaders/avatar_uploader.rb
include CarrierWave::MiniMagick
```

### 2.2 裁减图片

``` ruby app/uploaders/avatar_uploader.rb
version :thumb do
 process :resize_to_limit => [200, 200]
end
```

``` ruby app/views/articles/show.html.erb
<%= image_tag @article.image_url(:thumb).to_s %>
```

### 2.3 增加模糊

``` ruby app/uploaders/avatar_uploader.rb
# radial_blur为模糊的程度
process :radial_blur => 2
def radial_blur(amount)
  manipulate! do |img|
    img.radial_blur(amount)
    img = yield(img) if block_given?
    img
  end
end
```

### 2.4 旋转

``` ruby
process :sample_rotate => ["90%", "-80>"]
def sample_rotate(sample, rotate)
  manipulate! do |img|
    img.combine_options do |c|
      c.sample(sample)
      c.rotate(rotate)
    end
    img = yield(img) if block_given?
    img
  end
end
```

### 2.5 加水印

``` ruby
process :make_watermark => "你的图片的地址"
def make_watermark(watermark)
  manipulate! do |img|
    img = img.composite(MiniMagick::Image.open(watermark, "jpg")) do |c|
      c.gravity "SouthEast"
    end
    img = yield(img) if block_given?
    img
  end
end
```

打水印的位置可以是'Center', 'NorthWest', 'North', 'NorthEast', 'West', 'Center', 'East', 'SouthWest', 'South', 'SouthEast'

### 2.6 转化格式

``` ruby
process :convert => 'png'

def filename
  super.chomp(File.extname(super)) + '.png' if original_filename.present?
end
```

更多的用法可以看这里https://github.com/carrierwaveuploader/carrierwave/blob/master/lib/carrierwave/processing/mini_magick.rb
