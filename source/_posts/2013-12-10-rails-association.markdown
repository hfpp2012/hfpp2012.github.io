---
layout: post
title: "rails association"
date: 2013-12-10 13:13
comments: true
categories: ruby on rails
---

+ [rails guides association](http://guides.rubyonrails.org/association_basics.html)
+ [rails api association](http://guides.rubyonrails.org/association_basics.html)
+ [rails 实战圣经-资料表关联](http://ihower.tw/rails3/activerecord-relationships.html)
+ [rails polymorphic-association](http://railscasts.com/episodes/154-polymorphic-association)

## 模型关联

通过关键字(belongs_to, has_many等)把两个**model**(模型)关联起来,下面以我的开发经验和一些源码来解说每个模型关联

所有关键字如下:

+ belongs_to
+ has_one
+ has_many
+ has_many :through
+ has_one :through
+ has_and_belongs_to_many

### 一对多

通过**has_many**或**has_many :through**或自关联来定义,一个**modelA**有很多**modelB**,那些**modelB**都属于**modelA**例如一篇文章有很多评论, 一个班级有很多学员

#### has_many

{% img /images/rails_association/has_many_and_belongs_to.png %}

``` ruby
class Comment < ActiveRecord::Base
  belongs_to :artilce
end

class Article < ActiveRecord::Base
  has_many :comments
end
```

**comments**表必须要有一个**foreign_key**(article_id)才能实现两个表的关联

#### has_many :through

{% img /images/rails_association/has_many_through_three.png %}

``` ruby
class kindergarten < ActiveRecord::Base
  has_many :squads
  has_many :students, :through => :squads
end

class Squad < ActiveRecord::Base
  belongs_to :kindergarten
  has_many :students
end

class Student < ActiveRecord::Base
  belongs_to :squad
end
```

这种方式适用于三层关系(祖,父,孙)的，而且在第三层不用写belongs_to第一层

#### 自关联(self join)

foreign_key是可以指定你想要的任何字段,通过foreign_key我们可以实现自关联(Self Joins),即一个表中即定义了has_many又定义了belongs_to,例如[acts_as_tree](http://yinsigan.github.io/blog/2013/12/09/acts-as-treeyuan-ma-fen-xi/)就是这样实现的,它是一个树结构,它是这样实现的

``` ruby comment.rb
class Comment < ActiveRecord::Base
  has_many   :children, :class_name => 'Comment', :foreign_key => :parent_id
  belongs_to :parent,   :class_name => 'Comment', :foreign_key => :parent_id
end
```

### 一对一

通过**has_one**或**has_one :through**来定义,一个**modelA**有一个**modelB**, 那么**modelB**属于**modelA**,他们之间是一对一的关系,例如一个男人只有一个妻子,一个人只有一个身份证都是现实中一对一的关系

#### has_one

{% img /images/rails_association/has_one_belongs_to.png %}

``` ruby
class User < ActiveRecord::Base
  has_one :profile
end

class Profile < ActiveRecord::Base
  belongs_to :user
end
```

一对一的关系也是要通过foreign_key来关联的

#### has_one :through

{% img /images/rails_association/has_one_through.png %}

``` ruby
class User < ActiveRecord::Base
  has_one :profile
  has_one :proile_detail, :through => :profile
end

class Profile < ActiveRecord::Base
  belongs_to :user
  has_one :proile_detail
end

class ProfileDetail < ActiveRecord::Base
  belongs_to :profile
end
```

这个**has_one :through**的原理跟**has_many :through**的差不多,都是三级关系

### 多对多

能过**has_many :through**或**has_many_and_belongs_to**来定义,**modelA**有很多**modelB**,**modelB**也有很多**modelA**,它们之间是多对多的关系,例如人有很多的钱,钱属于很多人

#### has_many :through

{% img /images/rails_association/has_many_through.png %}

``` ruby
class Staff < ActiveRecord::Base
  has_many :teachers
  has_many :squads, :through => :teachers
end

class Squad < ActiveRecord::Base
  has_many :teachers
  has_many :squads, :through => :teachers
end

class Teacher < ActiveRecord::Base
  belongs_to :squad
  belongs_to :staff
end
```

多对多的实现是要通过中间表的,而has_many :through这种方式的中间表有两个好处,第一个是表名你可以自己随便命名(比较人性化),第二个好处你可以在中间表上加属性

#### has_many_and_belongs_to

{% img /images/rails_association/has_many_and_belongs_to.png %}

``` ruby
class Role < ActiveRecord::Base
  has_and_belongs_to_many :operates
end
class Operate < ActiveRecord::Base
  has_and_belongs_to_many :roles
end
```

这种方式对中间表的命名是有规定的,连中间表的model也不用写,**has_many_and_belongs_to**是rails提供的,缺点是不能灵活控制,优点是方便

在migration创建表不创建id(没必要)

### Polymorphic Associations(多态)

上述都是两个表进行关联,有的不需要中间表,有的不需要,如果要实现三个表以上的关联呢,这个时候就要用到多态了,例如我们不止在文章里可以加评论,在新闻里也可以评论,在贴子里都可以

{% img /images/rails_association/polymorphic.png %}

``` ruby
class Followable < ActiveRecord::Base
  belongs_to :followable, :polymorphic => true
end

class Book < ActiveRecord::Base
  has_many :follow, :as => :followable
end

class User < ActiveRecord::Base
  has_many :follow, :as => :followable
end

class Comment < ActiveRecord::Base
  has_many :follow, :as => :followable
end
```

多态的方法很灵活,它也需要一个中间表,但是他不止能关联一个表,而关联无数个表,在中间表中一个字段是存关联表的类型,一个存id,它通过这两个字段来实现关联所有表的


### 单表继承

有些模型的相似度达到百分之九十,甚至有些只有一个字段的值不同,没必要复制一遍model，再改不同的字段

这个时候可以用一个字段但存的值是不同的,这种方式可以的,只是因为类型不同,数据和逻辑不同,控制器即使写两套，还是要做各种判断,代码会变得很乱,很麻烦

这个时候一个好的解决方案就是用单表继承

{% img /images/rails_association/polymorphic.png %}

``` ruby
class GrowthRecord < ActiveRecord::Base
end

class HomeGrowthRecord < GrowthRecord
end

class GardenGrowthRecord < GrowthRecord
end
```

GrowthRecord必须存一个字段叫type:string的,这个是自动处理的

如果不让GrowthRecord被实例化,可以加self.abstract_class = true,这样GrowthRecord就被锁住了
