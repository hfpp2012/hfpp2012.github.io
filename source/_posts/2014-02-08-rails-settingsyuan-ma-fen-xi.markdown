---
layout: post
title: "rails-settings源码分析"
date: 2014-02-08 09:34
comments: true
categories: ruby on rails
---

[rails-settings](https://github.com/ledermann/rails-settings)

### 1. 数据库结构

var: string 键

value: text 值,存的是hash

target_id: integer  多态的id

target_type: string 多态的类

### 2. 源码文件结构

```
lib
├── rails-settings
│   ├── base.rb 定义了settings和settings=方法
│   ├── configuration.rb 操作has_settings的symbol列表和定义了default_settings
│   ├── scopes.rb  一些查询方法
│   ├── setting_object.rb 操作settings表
│   └── version.rb
└── rails-settings.rb
```

<!-- more -->

### 3. has_settings

``` ruby
ActiveRecord::Base.class_eval do
  def self.has_settings(*args, &block)
    RailsSettings::Configuration.new(*args.unshift(self), &block)

    include RailsSettings::Base
    extend RailsSettings::Scopes
  end
end
```

也就是说任何model有has_settings这个方法

现在来看下它是如何使用的

``` ruby
class User < ActiveRecord::Base
  has_settings :dashboard
end
```

这是has_settings最简单的使用方法了

```
u = User.first
u.settings(:dashboard)
=> SELECT `settings`.* FROM `settings` WHERE `settings`.`target_id` = 1 AND `settings`.`target_type` = 'User'
   #<RailsSettings::SettingObject id: nil, var: "dashboard", value: {}, target_id: 1, target_type: "User", created_at: nil, updated_at: nil>
```

有两个关键的方法是has_settings和settings方法

由```include RailsSettings::Base```我们先来看lib/rails-settings/base.rb这个文件的内容

### 4. settings和settings=方法

``` ruby
module RailsSettings
  module Base
    def self.included(base)
      base.class_eval do
        has_many :setting_objects,
                 # settings表中的target_id和target_type
                 :as         => :target,
                 :autosave   => true,
                 :dependent  => :delete_all,
                 # 类的名字我们可以自定义
                 :class_name => self.setting_object_class_name

        # 返回值
        def settings(var)
          # 必须存一个symbol
          raise ArgumentError unless var.is_a?(Symbol)
          # 而且key是必须存在的
          raise ArgumentError.new("Unknown key: #{var}") unless self.class.default_settings[var]

          if ActiveRecord::VERSION::MAJOR < 4
            setting_objects.detect { |s| s.var == var.to_s } || setting_objects.build({ :var => var.to_s }, :without_protection => true)
          else
            setting_objects.detect { |s| s.var == var.to_s } || setting_objects.build(:var => var.to_s, :target => self)
          end
        end

        def settings=(value)
          if value.nil?
            setting_objects.each(&:mark_for_destruction)
          else
            raise ArgumentError
          end
        end

        # 判断值的存在性
        def settings?(var=nil)
          if var.nil?
            setting_objects.any? { |setting_object| !setting_object.marked_for_destruction? && setting_object.value.present? }
          else
            settings(var).value.present?
          end
        end

        # 返回成hash
        def to_settings_hash
          settings_hash = self.class.default_settings
          settings_hash.each do |var, vals|
            settings_hash[var] = settings_hash[var].merge(settings(var.to_sym).value)
          end
          settings_hash
        end
      end
    end
  end
end
```

上面的代码(has_many :setting_objects)现在还不明白具体的含义,因为我们不知道setting_objects这个是什么东西

我们来看下lib/rails-settings/setting_objects.rb这个文件的内容

### 5. setting_objects和settings表


``` ruby
module RailsSettings
  class SettingObject < ActiveRecord::Base
    # 定义了表名
    self.table_name = 'settings'

    # 多态
    belongs_to :target, :polymorphic => true

    validates_presence_of :var, :value, :target_type
    validate do
      unless _target_class.default_settings[var.to_sym]
        errors.add(:var, "#{var} is not defined!")
      end
    end

    serialize :value, Hash

    # attr_protected can not be here used because it touches the database which is not connected yet.
    # So allow no attributes and override <tt>#sanitize_for_mass_assignment</tt>
    if ActiveRecord::VERSION::MAJOR < 4
      attr_accessible
    end

    REGEX_SETTER = /\A([a-z]\w+)=\Z/i
    REGEX_GETTER = /\A([a-z]\w+)\Z/i

    def respond_to?(method_name, include_priv=false)
      super || method_name.to_s =~ REGEX_SETTER
    end

    # 当执行u.settings(:dashboard).theme时会用到
    def method_missing(method_name, *args, &block)
      if block_given?
        super
      else
        # 当执行.var时
        if attribute_names.include?(method_name.to_s.sub('=',''))
          super
        elsif method_name.to_s =~ REGEX_SETTER && args.size == 1
          ＃ 当有给值的时候
          _set_value($1, args.first)
        elsif method_name.to_s =~ REGEX_GETTER && args.size == 0
          # 当没有值的时候就取值
          _get_value($1)
        else
          super
        end
      end
    end

  protected
    if ActiveRecord::VERSION::MAJOR < 4
      # Simulate attr_protected by removing all regular attributes
      def sanitize_for_mass_assignment(attributes, role = nil)
        attributes.except('id', 'var', 'value', 'target_id', 'target_type', 'created_at', 'updated_at')
      end
    end

  private
    # 取值
    def _get_value(name)
      if value[name].nil?
        # _target_class为使用has_settings的model
        _target_class.default_settings[var.to_sym][name]
      else
        # 直接从数据库取
        value[name]
      end
    end

    # 设置值
    def _set_value(name, v)
      if value[name] != v
        value_will_change!

        # 如果value是nil就把值删除掉
        if v.nil?
          value.delete(name)
        else
          # 设置值
          value[name] = v
        end
      end
    end

    def _target_class
      target_type.constantize
    end

    # Patch ActiveRecord to save serialized attributes only if they are changed
    if ActiveRecord::VERSION::MAJOR < 4
      # https://github.com/rails/rails/blob/3-2-stable/activerecord/lib/active_record/attribute_methods/dirty.rb#L70
      def update(*)
        super(changed) if changed?
      end
    else
      # https://github.com/rails/rails/blob/4-0-stable/activerecord/lib/active_record/attribute_methods/dirty.rb#L73
      def update_record(*)
        super(keys_for_partial_write) if changed?
      end
    end
  end
end
```

这个setting_objects主要是用来操作settings表的内容的

attribute_names返回的是一个对象的所有属性名

serialize :value, Hash 把value变成Hash对象来存储

default_settings是有些不明白

我们来看下lib/rails-settings/configuration.rb文件

### 6. has_settings的symbol列表

``` ruby
ActiveRecord::Base.class_eval do
  def self.has_settings(*args, &block)
    RailsSettings::Configuration.new(*args.unshift(self), &block)

    include RailsSettings::Base
    extend RailsSettings::Scopes
  end
end

module RailsSettings
  class Configuration
    def initialize(*args, &block)
      # 如果最后一个参数是hash就弹出来
      options = args.extract_options!
      # 把第一个参数弹出来
      klass = args.shift
      # 剩下的给keys
      keys = args

      raise ArgumentError unless klass

      @klass其实就是User这个model类
      @klass = klass
      @klass.class_attribute :default_settings, :setting_object_class_name
      # 默认为空的hash
      @klass.default_settings = {}
      # 定义默认的class_name
      @klass.setting_object_class_name = options[:class_name] || 'RailsSettings::SettingObject'

      # keys可能会包含:dashboard
      if block_given?
        yield(self)
      else
        keys.each do |k|
          # 可能default_settings的值会变成这样{:dashboard=>{}}
          key(k)
        end
      end

      raise ArgumentError.new('has_settings: No keys defined') if @klass.default_settings.blank?
    end

    def key(name, options={})
      # 必须是symbol
      raise ArgumentError.new("has_settings: Symbol expected, but got a #{name.class}") unless name.is_a?(Symbol)
      raise ArgumentError.new("has_settings: Option :defaults expected, but got #{options.keys.join(', ')}") unless options.blank? || (options.keys == [:defaults])
      @klass.default_settings[name] = (options[:defaults] || {}).stringify_keys.freeze
    end
  end
end
```

```
User.default_settings
=> {:dashboard=>{}}
```

stringify_keys把hash的key转成字符串

has_settings定义的Symbol列表就是放到default_settings中的
