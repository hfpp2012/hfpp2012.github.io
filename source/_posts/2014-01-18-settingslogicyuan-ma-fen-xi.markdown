---
layout: post
title: "settingslogic源码分析"
date: 2014-01-18 21:57
comments: true
categories: ruby on rails
---

+ [settingslogic](https://github.com/binarylogic/settingslogic)
+ [Ruby/Rails 中的YAML](http://blog.csdn.net/smilewater/article/details/1679835)
+ [How do I use YAML in ruby/rails?](http://stackoverflow.com/questions/4002092/how-do-i-use-yaml-in-ruby-rails)

这个gem是用来做配置文件的

### 使用

#### 1.定义你的class

在app/models下创建settings.rb文件,内容为:

``` ruby
class Settings < Settingslogic
  source "#{Rails.root}/config/application.yml"
  namespace Rails.env
end
```

<!-- more -->

#### 2.创建你的配置

在config下创建application.yml文件

``` ruby
# config/application.yml
defaults: &defaults
  cool:
    saweet: nested settings
  neat_setting: 24
  awesome_setting: <%= "Did you know 5 + 5 = #{5 + 5}?" %>

development:
  <<: *defaults
  neat_setting: 800

test:
  <<: *defaults

production:
  <<: *defaults
```

#### 3.访问你的配置

``` ruby
>> Rails.env
=> "development"

>> Settings.cool
=> "#<Settingslogic::Settings ... >"

>> Settings.cool.saweet
=> "nested settings"

>> Settings.neat_setting
=> 800

>> Settings.awesome_setting
=> "Did you know 5 + 5 = 10?"
```

### 分析

源码中只有一个文件叫settingslogic.rb

我们从app/models/settings.rb文件入手

``` ruby app/models/settings.rb
class Settings < Settingslogic
  source "#{Rails.root}/config/application.yml"
  namespace Rails.env
end
```

首先Settings继承自Settingslogic,而且用了两个方法source和namespace

``` ruby
class Settingslogic < Hash
end
```

Settingslogic是个hash

``` ruby
def source(value = nil)
  @source ||= value
end

def namespace(value = nil)
  @namespace ||= value
end
```

source和namespace不过是把值存到变量中去

先来了解Settingslogic类的结构

``` ruby
class Settingslogic < Hash
  # 定义了一个标准错误类
  class MissingSetting < StandardError; end

  # 下面都是类方法
  class << self
    def name # :nodoc:
      self.superclass != Hash && instance.key?("name") ? instance.name : super
    end
    ...

    # 除了用send,不能直接调用private里的私有类方法
    # 当调用上面的类方法时会调用私有类方法
    # 例如可以这样Settings.send(:instance)
    private
      def instance
        return @instance if @instance
        @instance = new
        create_accessors!
        @instance
      end

      def method_missing(name, *args, &block)
        instance.send(name, *args, &block)
      end
  end

  # 这之后定义的都是实例方法,也就是说Settingslogic.new之后调用的
  def initialize(hash_or_file = self.class.source, section = nil)
    ...
  end

end
```

我们先来了解initialize方法,initialize方法会利用source和namespace的值

``` ruby
# 第一个参数是yml配置文件的路径
def initialize(hash_or_file = self.class.source, section = nil)
  #puts "new! #{hash_or_file}"
  case hash_or_file
  # 没有指定就报错
  when nil
    raise Errno::ENOENT, "No file specified as Settingslogic source"
  # 如果是一个Hash的话就被代替
  when Hash
    self.replace hash_or_file
  else
    # 读这个文件的内容
    file_contents = open(hash_or_file).read
    # 如果内容为空就是一个空的hash,否则的话用YAML把它的结果转成hash
    hash = file_contents.empty? ? {} : YAML.load(ERB.new(file_contents).result).to_hash
    if self.class.namespace
      # 只取出相应namespace
      hash = hash[self.class.namespace] or return missing_key("Missing setting '#{self.class.namespace}' in #{hash_or_file}")
    end
    self.replace hash
  end
  # section是看value是不是hash
  @section = section || self.class.source  # so end of error says "in application.yml"
  create_accessors!
end

# 创建实例方法
def create_accessors!
  self.each do |key,val|
    create_accessor_for(key)
  end
end

# 以key定义实例方法
def create_accessor_for(key, val=nil)
  # 如果不是以字母开头就退出
  return unless key.to_s =~ /^\w+$/  # could have "some-setting:" which blows up eval
  # 设置实例变量的值,默认为nil
  instance_variable_set("@#{key}", val)
  self.class.class_eval <<-EndEval
    def #{key}
      # 如果已经存在就返回
      return @#{key} if @#{key}
      # 没有key就报错
      return missing_key("Missing setting '#{key}' in #{@section}") unless has_key? '#{key}'
      value = fetch('#{key}')
      # 如果是一个hash,那就去初始化
      @#{key} = if value.is_a?(Hash)
        self.class.new(value, "'#{key}' section in #{@section}")
      # 遍历
      elsif value.is_a?(Array) && value.all?{|v| v.is_a? Hash}
        value.map{|v| self.class.new(v)}
      # 直接返回
      else
        value
      end
    end
  EndEval
end
```

在这一步，从yml文件中读取内容转化成hash,然后定义了类似cool,neat_setting这样的实例方法

``` ruby
Settings.new.cool
=> {"saweet"=>"nested settings"}

Settings.new.instance_variables
=> [:@section, :@cool, :@neat_setting, :@awesome_setting]

Settings.new.cool.instance_variables
=> [:@section, :@saweet]

Settings.new.cool.instance_variable_get(:@section)
=> "'cool' section in /home/yinsigan/g/config/application.yml"
```

继承来看其他的实例方法

``` ruby
def [](key)
  fetch(key.to_s, nil)
end

def []=(key,val)
  # Setting[:key][:key2] = 'value' for dynamic settings
  val = self.class.new(val, @section) if val.is_a? Hash
  # 设置它的值
  store(key.to_s, val)
  # 创建实例方法
  create_accessor_for(key, val)
end

# 没有设置值之前
s['name']
=> nil

s['name'] = 'xiaozi'
=> "xiaozi"

s.name
=> "xiaozi"

s.instance_variables()
=> [:@section, :@cool, :@neat_setting, :@awesome_setting, :@name]
```

如果执行Settings.cool,会先去执行类方法中的method_missing

``` ruby
private
  # 这个方法执行之前会执行instance
  def method_missing(name, *args, &block)
    # 因为instance是new出来的,所以这个时候会去执行实例方法中的method_missing
    instance.send(name, *args, &block)
  end

  def instance
    # 如果有@instance这个变量返回就可以
    return @instance if @instance
    # 生成一个新hash
    @instance = new
    # 创建类方法
    create_accessors!
    @instance
  end

  # 通过遍历定义新的类方法
  def create_accessors!
    instance.each do |key,val|
      create_accessor_for(key)
    end
  end

  def create_accessor_for(key)
    # 如果不是以字母开头就退出
    return unless key.to_s =~ /^\w+$/  # could have "some-setting:" which blows up eval
    # 定义一个类方法
    instance_eval "def #{key}; instance.send(:#{key}); end"
  end
```

当你首次执行Settings.cool,就会去创建Settings类的实例,也就是@instance,会执行Settings中的方法,也就是@instance中有很多实例方法,以后执行Settings.neat_setting等方法就会去读取@instance

``` ruby
Settings.send(:instance)
=> {"cool"=>{"saweet"=>"nested settings"},
 "neat_setting"=>800,
 "awesome_setting"=>"Did you know 5 + 5 = 10?"}

Settings.instance_variable_get(:@instance).instance_variables
=> [:@section, :@cool, :@neat_setting, :@awesome_setting]
```

suppress_errors Rails.env.production?这个方法可以控制不报错

``` ruby
def suppress_errors(value = nil)
  @suppress_errors ||= value
end

def missing_key(msg)
  return nil if self.class.suppress_errors

  raise MissingSetting, msg
end
```

其他的方法可以自己分析

``` ruby
def name # :nodoc:
  self.superclass != Hash && instance.key?("name") ? instance.name : super
end

# Enables Settings.get('nested.key.name') for dynamic access
def get(key)
  parts = key.split('.')
  curs = self
  while p = parts.shift
    curs = curs.send(p)
  end
  curs
end

def symbolize_keys

  inject({}) do |memo, tuple|

    k = (tuple.first.to_sym rescue tuple.first) || tuple.first

    v = k.is_a?(Symbol) ? send(k) : tuple.last # make sure the value is accessed the same way Settings.foo.bar works

    memo[k] = v && v.respond_to?(:symbolize_keys) ? v.symbolize_keys : v #recurse for nested hashes

    memo
  end

end
```
