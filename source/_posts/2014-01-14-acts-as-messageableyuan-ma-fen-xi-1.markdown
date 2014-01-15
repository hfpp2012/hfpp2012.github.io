---
layout: post
title: "acts-as-messageable源码分析(一)"
date: 2014-01-14 11:05
comments: true
categories: ruby on rails
---

[acts-as-messageable](https://github.com/LTe/acts-as-messageable)

这个gem可以实现类似站内消息的机制。


数据库结构:

{% img /images/acts-as-messageable/table.png %}

<!-- more -->

源码结构:

```
lib
├── acts-as-messageable
│   ├── message.rb 定义了Message这个model
│   ├── model.rb 主要定义了acts_as_messageable这个方法
│   ├── rails3.rb
│   ├── rails4.rb
│   ├── railtie.rb 加载时用到
│   ├── relation.rb 主要定义了process方法
│   └── scopes.rb 定义了几个scope
├── acts-as-messageable.rb  主程序
```

### acts_as_messageable

这个方法定义在lib/acts-as-messageable/message.rb中。

``` ruby
def acts_as_messageable(options = {})
  # 定义默认的参数
  default_options = {
    :table_name => "messages",  # 表名
    :class_name => "ActsAsMessageable::Message", # 类名
    :required => [:topic, :body], # 必须的参数填写
    :dependent => :nullify,
    :group_messages => false,
  }
  options = default_options.merge(options)

  # 两个多态
  has_many  :received_messages_relation,  # 接收者
            :as => :received_messageable,
            :class_name => options[:class_name],
            :dependent => options[:dependent]
  has_many  :sent_messages_relation, # 发送者
            :as => :sent_messageable,
            :class_name => options[:class_name],
            :dependent => options[:dependent]

  # 字符串转成class
  self.messages_class_name = options[:class_name].constantize
  # 使用了has_ancestry这个gem
  self.messages_class_name.has_ancestry

  if self.messages_class_name.respond_to?(:table_name=)
    # 设置表名
    self.messages_class_name.table_name = options[:table_name]
    # 导入scopes.rb文件定义的几个scope
    self.messages_class_name.initialize_scopes
  else
    # 老式用法
    self.messages_class_name.set_table_name(options[:table_name])
    ActiveSupport::Deprecation.warn("Calling set_table_name is deprecated. Please use `self.table_name = 'the_name'` instead.")
  end

  # 包裹成数据
  self.messages_class_name.required = Array.wrap(options[:required])
  # 验证参数的存在性
  self.messages_class_name.validates_presence_of self.messages_class_name.required
  self.group_messages = options[:group_messages]

  include ActsAsMessageable::Model::InstanceMethods
end

def initialize_scopes
  # 查找发送消息的对象
  scope :are_from,          lambda { |*args| where(:sent_messageable_id => args.first, :sent_messageable_type => args.first.class.name) }
  # 查找接收消息的对象
  scope :are_to,            lambda { |*args| where(:received_messageable_id => args.first, :received_messageable_type => args.first.class.name) }
  # 模糊查找body和topic的内容
  scope :search,            lambda { |*args|  where("body like :search_txt or topic like :search_txt",:search_txt => "%#{args.first}%")}
  # 传两个参数，第一个是发送消息的对象或者接收消息的对象，第二个为布尔值，是否删除
  scope :connected_with,    lambda { |*args|  where("(sent_messageable_type = :sent_type and
                                            sent_messageable_id = :sent_id and
                                            sender_delete = :s_delete and sender_permanent_delete = :s_perm_delete) or
                                            (received_messageable_type = :received_type and
                                            received_messageable_id = :received_id and
                                            recipient_delete = :r_delete and recipient_permanent_delete = :r_perm_delete)",
                                            :sent_type      => args.first.class.resolve_active_record_ancestor.to_s,
                                            :sent_id        => args.first.id,
                                            :received_type  => args.first.class.resolve_active_record_ancestor.to_s,
                                            :received_id    => args.first.id,
                                            :r_delete       => args.last,
                                            :s_delete       => args.last,
                                            :r_perm_delete  => false,
                                            :s_perm_delete  => false)
  }
  # 查找已经被查阅的
  scope :readed,            lambda { where(:opened => true)  }
  # 查找还未被查阅的
  scope :unreaded,          lambda { where(:opened => false) }
  # 查找删除过的
  scope :deleted,           lambda { where(:recipient_delete => true, :sender_delete => true) }
end
```

这样来使用的。

``` ruby
class User < ActiveRecord::Base
  attr_accessible :name
  acts_as_messageable
end

ActsAsMessageable::Message.are_from(User.first)
=> SELECT `messages`.* FROM `messages` WHERE `messages`.`sent_messageable_id` = 1 AND `messages`.`sent_messageable_type` = 'User' ORDER BY created_at desc

ActsAsMessageable::Message.connected_with(User.first, false)
=> SELECT `messages`.* FROM `messages` WHERE ((sent_messageable_type = 'User' and
 sent_messageable_id = 1 and
 sender_delete = 0 and sender_permanent_delete = 0) or
 (received_messageable_type = 'User' and
 received_messageable_id = 1 and
 recipient_delete = 0 and recipient_permanent_delete = 0)) ORDER BY created_at desc
```

### send_message

发送消息

``` ruby
def send_message(to, *args)
  message_attributes = {}

  # 判断第一个参数的类型
  case args.first
    when String
      # attribute为topic, body之类
      self.class.messages_class_name.required.each_with_index do |attribute, index|
        # index为0,1之类
        message_attributes[attribute] = args[index]
      end
    when Hash
      message_attributes = args.first
  end

  # 存topic, body的值
  message = self.class.messages_class_name.new message_attributes
  # 存接收者对象的class和id
  message.received_messageable = to
  # 存发送者对象的class和id
  message.sent_messageable = self
  message.save

  message
end
```

``` ruby
User.first.send_message(User.last, 'h1, u2', 'hello u2')
=> INSERT INTO `messages` (`ancestry`, `body`, `created_at`, `opened`, `received_messageable_id`, `received_messageable_type`, `recipient_delete`, `recipient_permanent_delete`, `sender_delete`, `sender_permanent_delete`, `sent_messageable_id`, `sent_messageable_type`, `topic`, `updated_at`) VALUES (NULL, 'hello u2', '2014-01-14 07:02:44', 0, 3, 'User', 0, 0, 0, 0, 1, 'User', 'h1, u2', '2014-01-14 07:02:44')
```

### reply_to

回复

``` ruby
def reply_to(message, *args)
  current_user = self

  if message.participant?(current_user)
    # 根据real_receiver反方向发送等于回复，产生一个新的消息(即回复)
    reply_message = send_message(message.real_receiver(current_user), *args)
    # 新产生的回复的父是刚才的消息
    reply_message.parent = message
    reply_message.save

    # 返回回复的消息
    reply_message
  end
end

def participant?(user)
  user.class.group_messages || (to == user) || (from == user)
end

alias :from :sent_messageable
alias :to   :received_messageable

# 定义的消息接收者和发送者对象
belongs_to :received_messageable, :polymorphic => true
belongs_to :sent_messageable,     :polymorphic => true

def real_receiver(user)
  user == from ? to : from
end
```

```
m1 = User.first.send_message(User.last, 'hi, u2', 'hello u2')
User.first.reply_to(m1, 'reply: u1', "I'm fine!")
=> INSERT INTO `messages` (`ancestry`, `body`, `created_at`, `opened`, `received_messageable_id`, `received_messageable_type`, `recipient_delete`, `recipient_permanent_delete`, `sender_delete`, `sender_permanent_delete`, `sent_messageable_id`, `sent_messageable_type`, `topic`, `updated_at`) VALUES (NULL, 'I\'m fine!', '2014-01-14 07:25:39', 0, 3, 'User', 0, 0, 0, 0, 1, 'User', 'reply: u1', '2014-01-14 07:25:39')
```

### process和conversations

删除和恢复还有查找对话

执行m1.delete => true

``` ruby
attr_accessor   :removed, :restored

def delete
  self.removed = true
end
```

如果要真正删除

``` ruby
User.first.messages.process do |message|
  message.delete
end
```

来看下process方法

``` ruby
ActiveRecord::Relation.send :include, ActsAsMessageable::Relation

# relation_context这个变量是在ActiveRecord::Relation基础上定义的
attr_accessor :relation_context

def messages(trash = false)
  result = self.class.messages_class_name.connected_with(self, trash)
  # relation_context存的是user
  result.relation_context = self

  result
end

def process(context = self.relation_context, &block)
  self.each do |message|
    # 执行传过来的block,这个方法会执行delete
    block.call(message) if block_given?
    # 如果原先执行过delete，就执行delete_message方法
    context.delete_message(message)   if message.removed
    context.restore_message(message)  if message.restored
  end
end

def delete_message(message)
  current_user = self

  # 查找message是发送的还是接收的, 如果被删除过的话就要永久删除
  case current_user
    when message.to
      attribute = message.recipient_delete ? :recipient_permanent_delete : :recipient_delete
    when message.from
      attribute = message.sender_delete ? :sender_permanent_delete : :sender_delete
    else
      raise "#{current_user} can't delete this message"
  end

  message.update_attributes!(attribute => true)
end

def conversations
  map { |r| r.root.subtree.order("id desc").first }.uniq
end
```
