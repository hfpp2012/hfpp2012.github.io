---
layout: post
title: "Rails Association Reference"
date: 2013-12-11 10:09
comments: true
categories: ruby on rails
---

## rails关联模型参数的使用

上一篇[rails association](http://yinsigan.github.io/blog/2013/12/10/rails-association/)说明了rails的各种模型的原理和设计,这篇我们来说说rails association具体参数的使用,通过一些案例或代码来分析它们的使用场景

{% img /images/rails_association_reference/has_many_through.png %}

``` ruby
class School < ActiveRecord::Base
  has_many :squads
  has_many :students, :through => :squads
end

class Squad < ActiveRecord::Base
  belongs_to :school
  has_many :students
end

class Student < ActiveRecord::Base
  belongs_to :squad
end
```

### Association(关联关系的使用)

+ association(force_reload = false)强制reload查数据库

``` ruby
s = Squad.first
s.students
=> [#<Student id: 40, name: "学员1", squad_id: 1, created_at: "2013-12-11 02:43:53", updated_at: "2013-12-11 02:43:53", active: true>,
 #<Student id: 41, name: "学员2", squad_id: 1, created_at: "2013-12-11 02:43:56", updated_at: "2013-12-11 02:43:56", active: true>]


s.students.last.delete #把最后一个学员删除,那班级s只剩下一个学员
# 可是还是两个学员
s.students
=> [#<Student id: 40, name: "学员1", squad_id: 1, created_at: "2013-12-11 02:43:53", updated_at: "2013-12-11 02:43:53", active: true>,
 #<Student id: 41, name: "学员2", squad_id: 1, created_at: "2013-12-11 02:43:56", updated_at: "2013-12-11 02:43:56", active: true>]


# 如果传入一个参数true
s.students(true)
=> [#<Student id: 40, name: "学员1", squad_id: 1, created_at: "2013-12-11 02:43:53", updated_at: "2013-12-11 02:43:53", active: true>]
```

其实传入true就是去数据库查询,不直接从变量(**s.students**)取值
```
Student Load (0.8ms)  SELECT `students`.* FROM `students` WHERE `students`.`squad_id` = 1 AND `students`.`active` = 1
```

+ association=(associate)

``` ruby
# good
s = Student.first
s.squad = Squad.last
s.save

# bad
s.squad_id = '9'.to_i
s.save
```

这种方式有两个好处,第一个,你不在白名单(attr_accessible)写上:squad_id,第二个你能保证squad_id就是正确存在的吗

最佳实践也是推鉴这种方式的

``` ruby
current_user.posts.build(params[:post])
```

+ collection<<(object, …)创建记录

这是我们创建记录的主要手段之一

``` ruby
s = Squad.first
s.students << Student.create(:name => "学员")
```

+ collection.delete(object, …)删除关联关系(不删除记录)

``` ruby
s = Squad.first
s.students.delete(s.students.last) #删除最后一条,只是把值设为NULL
=> SQL (0.6ms)  UPDATE `students` SET `squad_id` = NULL WHERE `students`.`squad_id` = 1 AND `students`.`active` = 1 AND `students`.`id` IN (43)

```

delete_all不用传参数就可以删除所有关联关系了

要保留数据的时候又要去掉关联关系用这种方式最好了

+ collection.delete(object, …)删除记录

``` ruby
s = Squad.first
st = s.first
s.students.destroy(st)

# 是真正的删除记录
DELETE FROM `students` WHERE `students`.`id` = 46

```

destroy_all不用传参数就可以删除所有关联的记录了

+ collection_singular_ids=ids(更改关联关系)

``` ruby
s = Squad.first

s.student_ids
=> [42, 43]

s.students_ids = [40, 43]
#会查一遍数据表students,没有的id你可别乱来哦
Student Load (0.8ms)  SELECT `students`.* FROM `students` WHERE `students`.`id` IN (40, 43)

# 不要42了就把关联关系去掉(设为NULL)
SQL (0.6ms)  UPDATE `students` SET `squad_id` = NULL WHERE `students`.`squad_id` = 1 AND `students`.`active` = 1 AND `students`.`id` IN (42)

# 把40加进去
UPDATE `students` SET `squad_id` = 1, `updated_at` = '2013-12-11 03:19:40' WHERE `students`.`id` = 40

```

从表单的复选框传来的数据用这种方式解决最好了

最佳实践就是这么做滴

``` ruby 多角色管理
<% for role in Role.all %>
<div>
  <%= check_box_tag "user[role_ids][]", role.id, @user.roles.include?(role) %>
  <%=h role.name %>
</div>
<% end %>
<%= hidden_field_tag "user[role_ids][]", "" %>
```

+ collection.clear会把所有关联关系清除或者删掉

``` ruby
s = Squad.first
s.students.clear

# 只是把关联关系清除而已
UPDATE `students` SET `squad_id` = NULL WHERE `students`.`squad_id` = 1 AND `students`.`active` = 1 AND `students`.`id` IN (42, 42)

```

如果不止想关联关系清除呢,那个就是要在has_many时加入:dependent => :destroy就可以了

其实看下clear的源码就知道原理

``` ruby
# File activerecord/lib/active_record/associations/association_collection.rb, line 244
def clear
  return self if length.zero? # forces load_target if it hasn't happened already

  if @reflection.options[:dependent] && @reflection.options[:dependent] == :destroy
    destroy_all
  else
    delete_all
  end

  self
end
```

+ collection.exists?判断记录是否存在

``` ruby
s = Squad.first

s.students.exists?(42)
=> true

s.students.exists?(Student.first)
=> true

s.students.exists?(:id => 42)
=> true
```

用**exists?**来判断比**include?**更灵活,**include?**只支持类似这样`inlcude?(Student.first)`的判断


###Association Parameter Reference(参数的使用)

有些参数在上一篇[rails association](http://yinsigan.github.io/blog/2013/12/10/rails-association/)已经提及过，例如polymorphic, :through等,这里再阐述几个比较常用或用处比较大的


+ class_name

查找实际的类名(model)

``` ruby
# 默认情况下是:students的,如果要换个名字,就要指定实际的class_name
has_many :newest_students, :class_name => 'Student', :order => 'created_at DESC'

SELECT `students`.* FROM `students` WHERE `students`.`squad_id` = 1 ORDER BY created_at DESC
```

+ conditions

指定查询条件

``` ruby
has_many :students, :conditions => "active = 1"
```

在rails 4已经用where代替conditions了

conditions有个要注意的地方

`:conditions => "active = 1"`和`{active: true}`(Hash)是不一样的
当你用s.students.create或s.students.build创建student的时候,active的值会不一样
用`{active: true}`形式创建时不管你的active的默认值是否是true，创建的student的active都是true
而`:conditions => "active = 1"`就是看active的默认值

:conditions也可以接proc:`:conditions => proc { ["orders.created_at > ?", 10.hours.ago] }`

实际项目的代码如下:

``` ruby
class School < ActiveRecord::Base
  has_many :students, :include => :user, :order => "users.name DESC", :conditions => "users.tp = 0"
  has_many :staff_users, :class_name => "User", :order => :name, :conditions => "tp = 1"
  has_many :staffs, :through => :users, :order => "users.name DESC", :conditions => "users.tp = 1"
end
```

+ counter_cache

在数据库中记录孩子的数量

``` bash
rails g migration add_students_count_to_squads students_count:integer
```

``` ruby
class AddStudentsCountToSquads < ActiveRecord::Migration
  def change
    add_column :squads, :students_count, :string
    Squad.reset_column_information
    Squad.find_each do |p|
      Squad.update_counters p.id, :students_count => p.students.length
    end
  end
end
```

``` ruby
class Squad < ActiveRecord::Base
  has_many :students
end

class Student < ActiveRecord::Base
  belongs_to :squad, :counter_cache => true
end
```

默认的字段名是跟关联(has_many)的model是一致的,在上述的例子中就是**students_count**,如果想改变这个name也可以,给:counter_cache指定值就可以了,例如`:counter_cache => :children_count`

当删除关联关系时(squad_id设为NULL)时,students_count不会更新

+ dependent

依赖性

``` ruby
class Squad < ActiveRecord::Base
  has_many :students, :dependent => :destroy
end

class Student < ActiveRecord::Base
  belongs_to :squad
end
```

如果传入`:dependent => :nullify`不删除数据,只删除关联数据


+ foreign_key

外键

``` ruby
class Topic < ActiveRecord::Base
  belongs_to :creater, :class_name => 'User', :foreign_key => "creater_id"
end
```

当你的model name跟foreign_key不一致时就可以手动指定关联的foreign_key
这样写当`topic.creater.try(:name)`就可以简单取得创建者的name了

+ include

包含查询

``` ruby
class Squad < ActiveRecord::Base
  has_many :students
end

class Student < ActiveRecord::Base
  belongs_to :squad, :include => :school
end

class School < ActiveRecord::Base
  has_many :squads
end

s = Student.first
s.squad
# 查了schools这个表,把数据一并取了出来
Squad Load (0.3ms)  SELECT `squads`.* FROM `squads` WHERE `squads`.`id` = 14 LIMIT 1
School Load (0.3ms)  SELECT `schools`.* FROM `schools` WHERE `schools`.`id` IN (1)

# 前面已经查了schools的数据了,现在不会查sql的
s.squad.school
```

当你要查`Student.first.squad.school'就方便多了

+ touch
+ validate
+ source
+ group
+ extend
+ :order
+ :uniq

除了这些参数还有:readonly, :select, :limit, :offset等


###Association Callbacks

+ before_add
+ after_add
+ before_remove
+ after_remove
