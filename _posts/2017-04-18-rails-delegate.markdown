---
layout: post
title:  “Rails delegate的理解和用法”
date:   2017-01-15 08:43:59
author: Chris Wang
categories: ROR
tags: rails delegate
---

## 简介

##### `delegate`方法提供了一个`delegate`类方法，去将自己包含的对象轻易的暴露出来。

## 提供的Options

* :to => 指定一个对象
* :prefix => 在对象方法前是否添加前缀
* :allow_nil => 如果设置为true, 则阻止` NoMethodError `抛出异常

## 例子

```
class Greeter < ActiveRecord::Base
  def hello
    'hello'
  end

  def goodbye
    'goodbye'
  end
end

class Foo < ActiveRecord::Base
  belongs_to :greeter
  delegate :hello, to: :greeter
end

Foo.new.hello   # => "hello"
Foo.new.goodbye # => NoMethodError: undefined method `goodbye' for #<Foo:0x1af30c>
```
##### 多个关联

```
class Foo < ActiveRecord::Base
  belongs_to :greeter
  delegate :hello, :goodbye, to: :greeter
end

Foo.new.goodbye # => "goodbye"
```

##### 方法可以委托给实例变量,类变量或常量提供了他们作为一个symbols:

```
class Foo
  CONSTANT_ARRAY = [0,1,2,3]
  @@class_array  = [4,5,6,7]

  def initialize
    @instance_array = [8,9,10,11]
  end
  delegate :sum, to: :CONSTANT_ARRAY
  delegate :min, to: :@@class_array
  delegate :max, to: :@instance_array
end

Foo.new.sum # => 6
Foo.new.min # => 4
Foo.new.max # => 11
```

##### 当然也可以`delegate`给自己的类方法, 只需要使用`:class`

```
class Foo
  def self.hello
    "world"
  end

  delegate :hello, to: :class
end

Foo.new.hello # => "world"
```

##### 下面是使用`prefix`的例子

```
class Invoice < Struct.new(:client)
  delegate :name, :address, to: :client, prefix: :customer
end

invoice = Invoice.new(john_doe)
invoice.customer_name    # => 'John Doe'
invoice.customer_address # => 'Vimmersvej 13'
```

##### 如果你不希望返回` NoMethodError `, 你可以使用`allow_nil: true`

```
class User < ActiveRecord::Base
  has_one :profile
  delegate :age, to: :profile, allow_nil: true
end

User.new.age # nil
```

==注意:== 如果对象不为空的话,`allow_nil: true`仍然会抛出`NoMethodError`

```
class Foo
  def initialize(bar)
    @bar = bar
  end

  delegate :name, to: :@bar, allow_nil: true
end

Foo.new("Bar").name # raises NoMethodError: undefined method `name'
```
