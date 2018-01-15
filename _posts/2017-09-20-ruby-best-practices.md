---
layout: post
title:  “Ruby最佳实践”
date:   2017-09-20 08:43:59
author: Chris Wang
categories: ror
tags: rails ruby best practices
---

## 引言
最近上班整理了一下ruby的最佳实践,这里有的都是自己整理过后的,记下来以备后用。

1. 代码缩进

#缩进两格
```
def A
  puts 'a'
end
```
#空格缩进
```
[a, b, c].map{ ... }
```
在逗号`, ; :`后面加个空格，增加可读性

#`case`和`when`是同一级
```
case
when A < 5
  puts 'Not again!'
when B > 100
  puts 'Too long!'
when C > 200
  puts "It's too late"
else
  ...
end
```

#在`attr_...`后面换行
```
class Foo
  attr_accessor :foo

  def foo
    # 做一些事情
  end
end
```
#使用`_`语法改善大数的数值字面量的可读性。
```
num = 1_000_000
```
2. 定义方法和参数

#有参数的方法加()
```
def A
  ...
end

def B(baz, foo)
  ...
end
```
#有可选参数，放在后面
```
def A(a, b, c = 0, d = 0)
  ...
end
```
#使用符号而不是字符串做hash的键。
```
hash = { one: 1, two: 2, three: 3 }
```

3. 逻辑判断

#if-else
```
def A
  if a
    ...
  else
    ...
  end
end

# 如果遇到简单的逻辑，也可以用三目运算符
def B
  something ? 'yes' : 'no'
end
```
#不用`and`, `or`,而是使用`&&`, `||`
#如果是否定条件,可以使用`unless`
```
# if
do_something if !some_condition

# unless
do_something unless some_condition
```

4. 方法调用

#区块中如果有唯一操作, 可以用`&:`
```
names = ['a', 'b', 'c']
names.map(&:upcase)
```
#链式调用少用`do...end`,多使用`{ ... }`
```
names = ['a', 'b', 'c']
names.each { |name| puts name }
names.select { |name| name.start_with?('a') }.map(&:upcase)
```
#避免在不需要的情况下使用`self`。（只有在调用 self 的修改器、以保留字命名的方法、重载的运算符时才需要）
```
def ready?
  if self.last_reviewed_at > self.last_updated_at
    self.worker.update(self.content, self.options)
    self.status = :in_progress
  end
  self.status == :verified
end

def ready?
  if last_reviewed_at > last_updated_at
    worker.update(content, options)
    self.status = :in_progress
  end
  status == :verified
end
```
#在没有流程控制的情况下少使用`return`
```
def A(arr)
  arr.size
end
```
#区块的参数如果没有被使用到, 可以使用`_`,作用是可以抑制类似于rubocop发出变量尚未使用的警告
```
result = hash.map { |_, v| v + 1 }
```
#当只需要便利key或者value的时候，可以选择：
```
hash.each_key { |k| p k }
hash.each_value { |v| p v }
```