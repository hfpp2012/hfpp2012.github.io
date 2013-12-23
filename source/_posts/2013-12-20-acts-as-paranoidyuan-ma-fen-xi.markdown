---
layout: post
title: "acts_as_paranoid源码分析"
date: 2013-12-20 21:57
comments: true
categories: ruby on rails
---

+ [acts_as_paranoid](https://github.com/goncalossilva/acts_as_paranoid)

它的作用就是假删除,在实际中还是会很有用的。有一天,客户说,她刚才误删了一个东西,要你把它找回来,这个时候它就派上用场了。

回收站或者垃圾箱也是假删除的典型应用.假如一个邮箱系统中,用户把邮件删除掉,这个时候这些邮件并没有被彻底删除掉,它其实是先到了垃圾箱中,如果要彻底删除它,请到垃圾箱那里再删一次吧

其实它实际上不删除数据中的数据,只不过是隐藏起来而已,只要让用户看不到,它就等于删除了,实际上,要还原的话修改一下数据库就可以回来了

它实现的原理很简单,只不过是用一个标志来实现隐藏数据,在数据表中加一个字段,把它的值改一下,它就删除了(隐藏),修改回来,它又出现了

它的目的就是保护数据的安全,让用户能在误操作的情况下也能恢复数据。但是缺点也很明显,由于不是真正的删除,数据库中仍然保留着那条数据,数据库会越来越庞大,垃圾的信息也是越来越多

默认情况下用**"deleted_at"**这个字段,用一个参数**column**来指定,它可以有三种类型boolean, string, time,这三个类型用**column_type**参数来指定

boolean:布尔型,被删除时值为true

string:字符串型,被删除时值为"deleted",这个值可以用deleted_value参数来指定

time:时间型,被删除时值为当前时间(删除操作的时间)

以上三个类型未删除时值都为NULL(nil)。建议使用time类型

使用举例如下:

``` ruby
class School < ActiveRecord::Base
  acts_as_paranoid :column_type => :boolean
  或
  acts_as_paranoid :column => "is_deleted", :column_type => :string, :deleted_value => "deleted"
  ...
end
```

分析该gem的源码可以懂得下面的知识

1. attr_protected的使用
2. 写一个callback
3. included_modules的用法
4. scoped unscoped的用法
5. activerecord/Reflection的用法
6. 自己写一个validator
7. class_attribute的用法
8. activerecord/relation中klass的用法


## 代码分析

### 代码结构

```
├── lib
│   ├── acts_as_paranoid
│   │   ├── associations.rb   关于关联模型的
│   │   ├── core.rb   核心文件
│   │   ├── relation.rb     关于relation的操作
│   │   └── validations.rb   一个验证方法
│   └── acts_as_paranoid.rb  主程序
```

### Filtering(过滤)

``` ruby
School.only_deleted # 找到所有已经假删除的数据
School.with_deleted # 找到所有数据(不管有没有假删除的)
```

{% img /images/acts_as_paranoid/delete_record.png %}

#### only_deleted

现在我们执行School.only_deleted来看看有什么效果

```
School Load (0.8ms)  SELECT `schools`.* FROM `schools` WHERE (schools.deleted_at IS NOT NULL)
```

如果我们自己写only_deleted的实现代码可能会写成这样

``` ruby
deleted_at = options[:column]
School.where("#{deleted_at} IS NOT NULL")
```

其实acts_as_paranoid的源码也差不多

``` ruby acts_as_paranoid
def only_deleted
  # 判断是不是string类型
  if string_type_with_deleted_value?
    without_paranoid_default_scope.where("#{paranoid_column_reference} IS ?", paranoid_configuration[:deleted_value])
  else
    without_paranoid_default_scope.where("#{paranoid_column_reference} IS NOT ?", nil)
  end
end
```

由于string这种类型的不能判断是不是等于NULL,而是要看是不是等于一个值(默认为"deleted", 由":deleted_value"这个参数决定)

于是

``` ruby 伪代码
def string_type_with_deleted_value?
  if options[:column_type] == :string
    return true
  end
  false
end
```

acts_as_paranoid的代码为

``` ruby
def string_type_with_deleted_value?
  paranoid_column_type == :string && !paranoid_configuration[:deleted_value].nil?
end
```

without_paranoid_default_scope是?

``` ruby
def without_paranoid_default_scope
  scope = self.scoped.with_default_scope
  scope.where_values.delete(paranoid_default_scope_sql)

  scope
end
```

如果不是一个gem它不要也行,我们写成School.where...照样目的能达到

既然做成了一个gem,每个用acts_as_paranoid的model都默认加了一个default_scope,一般情况下,假删除的数据是查不到的,所以要隐藏起来,with_deleted会使用,或者你执行School.all也会使用的

default_scope的效果大约是这样的

``` ruby
School.where("deleted_at IS NULL")
```

所以without_paranoid_default_scope的作用就是去掉这个效果

其实要去掉只要用unscoped就可以了,但acts_as_paranoid把with_deleted,和only_deleted做成了scope(activerecord/relations)的效果

``` ruby
School.scoped.with_default_scope.where_values
=> ["`schools`.`deleted_at` IS NULL"]
School.paranoid_default_scope_sql
=> "`schools`.`deleted_at` IS NULL"
```

so

``` ruby
School.with_deleted.where(...)
School.where(...).with_deleted
```

而acts_as_paranoid的default_scope是这样的

``` ruby
default_scope { where(paranoid_default_scope_sql) }

def paranoid_default_scope_sql
  if string_type_with_deleted_value?
    self.scoped.table[paranoid_column].eq(nil).
      or(self.scoped.table[paranoid_column].not_eq(paranoid_configuration[:deleted_value])).
      to_sql
  else
    self.scoped.table[paranoid_column].eq(nil).to_sql
  end
end
```

scoped.table不懂的看[arel-table](https://github.com/rails/arel)

paranoid_column_reference是?

``` ruby
self.paranoid_column_reference = "#{self.table_name}.#{paranoid_configuration[:column]}"
```

paranoid_configuration是?

``` ruby
raise ArgumentError, "Hash expected, got #{options.class.name}" if not options.is_a?(Hash) and not options.empty?
class_attribute :paranoid_configuration, :paranoid_column_reference

self.paranoid_configuration = { :column => "deleted_at", :column_type => "time", :recover_dependent_associations => true, :dependent_recovery_window => 2.minutes }
self.paranoid_configuration.merge!({ :deleted_value => "deleted" }) if options[:column_type] == "string"
self.paranoid_configuration.merge!(options) # user options

raise ArgumentError, "'time', 'boolean' or 'string' expected for :column_type option, got #{paranoid_configuration[:column_type]}" unless ['time', 'boolean', 'string'].include? paranoid_configuration[:column_type]

self.paranoid_column_reference = "#{self.table_name}.#{paranoid_configuration[:column]}"

return if paranoid?
```

paranoid_configuration是个关于配置的,我们传入的column和column_type等都会存到这个hash里面

[class_attribute](http://apidock.com/rails/Class/class_attribute)这个和[cattr_accessor](http://apidock.com/rails/Class/cattr_accessor)可以作这个比较

class_attribute和cattr_accessor都包含read(get)和write(set),但是class_attribute只属于类,子类都不能共享,而cattr_accessor是类和子类和实例共享的,具体的看文档就知道了

merge和merge!的区别是后者会改变原来的hash

merge和update差不多,如果有相同的key,会替换原来的值,没有的会添加

``` ruby
def paranoid?
  self.included_modules.include?(ActsAsParanoid::Core)
end
```

included_modules是类方法,返回所以它include的module,用这个就可以判断是否有加载过ActsAsParanoid,加载过的就可以不用再次加载了

至此only_deleted已经解释完了

#### with_deleted

来看下with_deleted,它就是无论有没有被假删除,都读出来

``` ruby
def with_deleted
  without_paranoid_default_scope
end
```

without_paranoid_default_scope上面已经解释过


#### deleted_after_time, deleted_before_time, deleted_inside_time_window

deleted_after_time: 在一个时间之后假删除的记录

deleted_before_time: 在一个时间之前假删除的记录

deleted_inside_time_window: 在一定时间内假删除的记录

{% img /images/acts_as_paranoid/deleted_time.png %}

实现方法也很简单

``` ruby
if paranoid_configuration[:column_type] == 'time'
  scope :deleted_inside_time_window, lambda {|time, window|
    deleted_after_time((time - window)).deleted_before_time((time + window))
  }

  scope :deleted_after_time, lambda  { |time| where("#{paranoid_column} > ?", time) }
  scope :deleted_before_time, lambda { |time| where("#{paranoid_column} < ?", time) }
end
```

### Real deletion(真删除)

实际上我们执行School.first.destroy时是假删除而已,只是把字段(deleted_at)的值改了一下,要想真正删除还是要再执行一次destroy

{% img /images/acts_as_paranoid/destroy_record.png %}

如果不想执行两次destroy,那就用destroy!一次搞定

#### destroy 单条记录假删除

我们来看一下它实现的代码,它把本来的destroy方法给重写了

``` ruby
def destroy
  # 还没有被假删除的情况下
  if !deleted?
    with_transaction_returning_status do
      run_callbacks :destroy do
        # Handle composite keys, otherwise we would just use `self.class.primary_key.to_sym => self.id`.
        self.class.delete_all(Hash[[Array(self.class.primary_key), Array(self.id)].transpose]) if persisted?
        self.paranoid_value = self.class.delete_now_value
        self
      end
    end
  else
    destroy!
  end
end
```

读懂上面的代码必须先弄懂deleted?

``` ruby
# 通过deleted_at的值来检验记录是否被删除
def deleted?
  !(paranoid_value.nil? ||
    (self.class.string_type_with_deleted_value? && paranoid_value != self.class.delete_now_value))
end

# destroyed?是rails提供的,现在的效果跟:deleted?一样
alias_method :destroyed?, :deleted?

# 判断一条记录是否是常归存在
def persisted?
  !(new_record? || @destroyed)
end

# 这个方法是rails提供的,destroyed?会利用到@destroyed
def destroyed?
  @destroyed
end
```

总而言之,deleted?是判断是不是被假删除

[with_transaction_returning_status](http://apidock.com/rails/ActiveRecord/Transactions/with_transaction_returning_status)是ActiveRecord::Base提供的方法,作用就是运行一个transaction,然后再返回一个状态值

``` ruby with_transaction_returning_status的源码
def with_transaction_returning_status
  status = nil
  self.class.transaction do
    add_to_transaction
    status = yield
    raise ActiveRecord::Rollback unless status
  end
  status
end
```

现在我们知道了,原来destroy?是包在一个transaction里的

接下来,run_callbacks :destroy是?

关于run_callbacks可以看[callback](http://api.rubyonrails.org/classes/ActiveSupport/Callbacks.html)

运行run_callbacks :destroy的时候如果有定义before_destroy和after_destroy的话,会运行它们

run_callbacks :destroy里面的内容才是真正的删除操作

``` ruby
self.class.delete_all(Hash[[Array(self.class.primary_key), Array(self.id)].transpose]) if persisted?
self.paranoid_value = self.class.delete_now_value
self
```

先来看下self.class.delete_all!

``` ruby
def delete_all(conditions = nil)
  # update_all相比于update_attributes来说后面可以加查询条件conditions,还有update_all是类方法
  # 如果不加条件(conditions),update_all会修改所有记录的deleted_at值
  update_all ["#{paranoid_configuration[:column]} = ?", delete_now_value], conditions
end
```

delete_now_value它的作用就是把deleted_all改为当前时间(如果deleted_at为time类型)或True(如果delete_at为boolean类型)或一个值(如果delete_at为string类型)

so

``` ruby
def delete_now_value
  case paranoid_configuration[:column_type]
  when "time" then Time.now
  when "boolean" then true
  when "string" then paranoid_configuration[:deleted_value]
  end
end
```

#### destroy! 单条记录真删除

实现的代码跟destroy差不多,它不改deleted_at的值,直接删除数据

``` ruby
def destroy!
  with_transaction_returning_status do
    run_callbacks :destroy do
      destroy_dependent_associations!
      # Handle composite keys, otherwise we would just use `self.class.primary_key.to_sym => self.id`.
      self.class.delete_all!(Hash[[Array(self.class.primary_key), Array(self.id)].transpose]) if persisted?
      self.paranoid_value = self.class.delete_now_value
      freeze
    end
  end
end
```

destroy_dependent_associations!这个跟关联删除(dependent: destroy)有关的,也说是说假如school has_many squads,删除一条school的时候会先把所有squads先删除掉,这个留待下文详述

来看delete_all!方法

``` ruby
def delete_all!(conditions = nil)
  without_paranoid_default_scope.delete_all!(conditions)
end
```

without_paranoid_default_scope是不带default_scope的scoped查询,也就是去掉那个默认的查询(不含假删除过的记录),上文有说过

without_paranoid_default_scope.delete_all!这个delete_all!可不是类方法,而是scoped中的方法,因为without_paranoid_default_scope是一个scoped(可以看without_paranoid_default_scope的定义)

在哪里找到delete_all!呢?

由于scoped是定义在ActiveRecord/Relation中的,lib/acts_as_paranoid.rb有一条语句

``` ruby
# Override ActiveRecord::Relation's behavior
ActiveRecord::Relation.send :include, ActsAsParanoid::Relation
```

so

``` ruby lib/acts_as_paranoid/relation.rb
module ActsAsParanoid
  module Relation
    def self.included(base)
      # 实例方法
      base.class_eval do
        ....
        alias_method :orig_delete_all, :delete_all
        def delete_all!(conditions = nil)
          if conditions
            # 没有条件就会去调用orig_delete_all
            where(conditions).delete_all!
          else
            orig_delete_all
          end
        end
        ...
      end
    end
  end
end
```

其实把那个查询条件交给where,再用orig_delete_all删除

上面的例子前后已经出现过两次alias_method了,我们来总结一下

在上面的例子中alias_method会把:delete_all(这个时候的delete_all是rails默认提供的那个)的代码复制一份给orig_delete_all,然后把delete_all重写一次,以后调用都会用重写过的,被重写那个(rails提供)就变成了orig_delete_all

freeze的作用是它那个记录锁住,已经被删除了那就不能修改,虽然存到变量之后还能读取

#### delete_all 多条记录假删除

多条记录真删除的那个方法就lib/acts_as_paranoid/relation.rb中的delete_all!方法

``` ruby
def delete_all(conditions = nil)
  # 判断是否有加载过acts_as_paranoid,对于没有使用这个gem的会使用rails提供的delete_all!(真删除)来删除
  if paranoid?
    update_all(paranoid_deletion_attributes, conditions)
  else
    delete_all!(conditions)
  end
end

# 传入数组
def destroy!(id_or_array)
  where(primary_key => id_or_array).orig_delete_all
end
```

### Associations 关联关系

有一种情况是这样的,我们可能在school model里有这样的东西`has_many :squads, :dependent => :destroy`(班级),假如你删除了一条school,那squads怎么办呢

这种情况下squad model分为两种可能: 1:要么没使用acts_as_paranoid 2:要么使用acts_as_paranoid

对于没使用acts_as_paranoid的squad model那就很简单,本来该怎么着就怎么着

对于使用acts_as_paranoid的squad model那就是**连带假删除**

#### 连带假删除

``` ruby
class School < ActiveRecord::Base
  has_many :squads, :dependent => :destroy
  acts_as_paranoid
end

class squad < ActiveRecord::Base
  belongs_to :school
  acts_as_paranoid
end
```
{% img /images/acts_as_paranoid/dependent_destroy.png %}

其实关于acts_as_paranoid的假删除上面已经说过它的代码了

就是那个destroy方法,按照rails的机制,当设置:dependent => :destroy,父类执行destroy方法,子类也是会执行自己的destroy的

也就是squad model会执行acts_as_paranoid重写的destroy方法,因为squad model使用了acts_as_paranoid

所以它(squad)使用了跟父类(school)同样的destroy方法,上文说过,这里就不做多解释了

#### 连带真删除

上文说过,真删除使用的是destroy!方法,在父类(school)使用destroy!,子类(squad)并没有执行任何方法,但它又需要把属于school的所有squads(班级)全部删除

所以在destroy!需要处理子类(把它删除掉)

``` ruby
def destroy!
  with_transaction_returning_status do
    run_callbacks :destroy do
      destroy_dependent_associations!
      ...
    end
  end
end
```

destroy_dependent_associations这个方法才是处理子类的关键

``` ruby
def destroy_dependent_associations!
  self.class.dependent_associations.each do |reflection|
    # 这句也很关键,判断子类有没有使用acts_as_paranoid,
    # 没有的话就没必要进行下去,应该根本没有destroy!方法,而rails默认就会删除的
    next unless reflection.klass.paranoid?

    scope = reflection.klass.only_deleted

    # Merge in the association's scope
    scope = scope.merge(association(reflection.name).association_scope)

    scope.each do |object|
      # 这句才是真正的删除
      object.destroy!
    end
  end
end
```

要真删除school,总得知道它有哪些子类belongs_to school吧,所以这个destroy_dependent_associations方法的前半部分都在找school究终has_many了哪些model,最后一句object.destroy!才是真正的删除


dependent_associations是?

``` ruby
def dependent_associations
  # 过滤父类中has_many dependent等于:destroy或:delete_all的association
  self.reflect_on_all_associations.select {|a| [:destroy, :delete_all].include?(a.options[:dependent]) }
end
```

[reflect_on_all_associations](http://api.rubyonrails.org/classes/ActiveRecord/Reflection/ClassMethods.html)是rails提供的方法

其实reflect_on_all_associations就是返回model所有包含的associations(关联关系的数组),而reflect_on_association可以取得某个特定的关联关系,例如School.reflect_on_association(:squads)

{% img /images/acts_as_paranoid/reflect_on_association.png %}

association是activerecord/associations.rb提供的

``` ruby
def association(name) #:nodoc:
  association = association_instance_get(name)

  if association.nil?
    reflection  = self.class.reflect_on_association(name)
    association = reflection.association_class.new(self, reflection)
    association_instance_set(name, association)
  end

  association
end
```

其实它内部主要也是在用reflect_on_association,它用于某个实例,找到这个实例相关联的squads,然后交给下面的代码删除

#### belongs_to的参数with_deleted

把一个school假删除之后,它包含的squads也被假删除了,但是执行squad.school来查找school的时候会找不到,因为它使用了default_scope,那个default_scope本来就不查假删除的数据的

{% img /images/acts_as_paranoid/belongs_to_with_deleted.png %}

而acts_as_paranoid提供了一个方式(:with_deleted => true)解决这个问题

看下面的代码

``` ruby
class Squad < ActiveRecord::Base
  belongs_to :deleted_school, :class_name => 'School', :foreign_key => "school_id", :with_deleted => true
end
```

当执行squad.deleted_school就可以查到那条假删除的school数据了

{% img /images/acts_as_paranoid/belongs_to_search.png %}

其实那个:with_deleted => true会破掉default_scope

它的源码定义在lib/acts_as_paranoid/associations.rb文件

在acts_as_paranoid.rb主程序文件里引入它

``` ruby
# Extend ActiveRecord::Base with paranoid associations
ActiveRecord::Base.send :include, ActsAsParanoid::Associations
```

``` ruby
module ActsAsParanoid
  module Associations
    def self.included(base)
      base.extend ClassMethods
      class << base
        alias_method_chain :belongs_to, :deleted
      end
    end

    module ClassMethods
      def belongs_to_with_deleted(target, options = {})
        with_deleted = options.delete(:with_deleted)
        result = belongs_to_without_deleted(target, options)

        if with_deleted
          result.options[:with_deleted] = with_deleted
          unless method_defined? "#{target}_with_unscoped"
            class_eval <<-RUBY, __FILE__, __LINE__
              def #{target}_with_unscoped(*args)
                association = association(:#{target})
                return nil if association.options[:polymorphic] && association.klass.nil?
                return #{target}_without_unscoped(*args) unless association.klass.paranoid?
                association.klass.with_deleted.scoping { #{target}_without_unscoped(*args) }
              end
              alias_method_chain :#{target}, :unscoped
            RUBY
          end
        end

        result
      end
    end
  end
end
```

### Recovery 还原

假如把一个邮件放到垃圾箱里,那个垃圾箱应该有个清空(真删除)的操作,也要有个还原的操作,现在我们来介绍这个还原的功能

{% img /images/acts_as_paranoid/recover.png %}

源码是这样的

``` ruby
def recover(options={})
  # 这样组织参数很灵活
  options = {
    :recursive => self.class.paranoid_configuration[:recover_dependent_associations],
    :recovery_window => self.class.paranoid_configuration[:dependent_recovery_window]
  }.merge(options)

  self.class.transaction do
    run_callbacks :recover do
      # 通过options[:recursive]的值来判断是否要连带还原
      recover_dependent_associations(options[:recovery_window], options) if options[:recursive]

      self.paranoid_value = nil
      self.save
    end
  end
end
```

它的主要内容是`self.paranoid_value = nil`,然后再保存(save),这样就可以还原成功

它还带了两个参数,一个叫recursive,它是关于**连带还原**的,也就是说你把school还原了,顺便把属于school的squads(班级)也给还原了

还有另外一个叫recovery_window,它是一个间隔时间,它的作用跟上文说过deleted_inside_time_window差不多,这个recovery_window参数是跟子类的deleted_at时间比较的

而这两个参数都有默认值,而且是从paranoid_configuration这个变量读取的,也就是说在使用acts_as_paranoid就可以给它指定值了,不一定要从recover(options={})中的options传入

看这个就知道了

``` ruby lib/acts_as_paranoid.rb
self.paranoid_configuration = { :column => "deleted_at", :column_type => "time", :recover_dependent_associations => true, :dependent_recovery_window => 2.minutes }
```

主要的内容在这一行```recover_dependent_associations(options[:recovery_window], options) if options[:recursive]```

``` ruby
def recover_dependent_associations(window, options)
  self.class.dependent_associations.each do |reflection|
    # 同样的,没有使用acts_as_paranoid的子类model跳过
    next unless reflection.klass.paranoid?

    scope = reflection.klass.only_deleted

    # Merge in the association's scope
    scope = scope.merge(association(reflection.name).association_scope)

    # We can only recover by window if both parent and dependant have a
    # paranoid column type of :time.
    if self.class.paranoid_column_type == :time && reflection.klass.paranoid_column_type == :time
      scope = scope.merge(reflection.klass.deleted_inside_time_window(paranoid_value, window))
    end

    scope.each do |object|
      object.recover(options)
    end
  end
end
```

这个方法的内容跟destroy_dependent_associations差不多,也是想通过association找到相关联的记录(squads),有区别是下面这几行

``` ruby
# self是自己,reflection.class是每个关联的子类model
if self.class.paranoid_column_type == :time && reflection.klass.paranoid_column_type == :time
  scope = scope.merge(reflection.klass.deleted_inside_time_window(paranoid_value, window))
end

scope.each do |object|
  object.recover(options)
end
```

首先通过判断paranoid_column_type的类型是否是time,因为只有time类型才会用到deleted_inside_time_window,然后遍历scope来执行recover


