---
layout: post
title:  “Ruby中Proc, lambda, block的不同之处 ”
date:   2016-12-24 10:43:59
author: Chris Wang
categories: ROR
tags:	ruby lambda proc block
cover:  "/assets/instacode.png"
---

## 简介

Proc, Lambda, block一直是Ruby元编程当中绕不过去的问题，它们经常出现，用法也是多种多样，今天就来了解一些它们之间的不同。

我们用代码示例:
```
# Block Examples

[1,2,3].each { |x| puts x*2 }   # block is in between the curly braces

[1,2,3].each do |x|
  puts x*2                    # block is everything between the do and end
end

# Proc Examples             
p = Proc.new { |x| puts x*2 }
[1,2,3].each(&p)              # The '&' tells ruby to turn the proc into a block

proc = Proc.new { puts "Hello World" }
proc.call                     # The body of the Proc object gets executed when called

# Lambda Examples            
lam = lambda { |x| puts x*2 }
[1,2,3].each(&lam)

lam = lambda { puts "Hello World" }
lam.call
```

虽然看起来都是差不多的，但是还是有细微的区别。

首先我们来看一下`Proc`和`Block`的区别：

1. `Proc`是对象, 但是`Block`不是

```
p = Proc.new { puts "Hello World" }

p.call  # prints 'Hello World'
p.class # returns 'Proc'
a = p   # a now equals p, a Proc instance
p       # returns a proc object '#<Proc:0x007f96b1a60eb0@(irb):46>'

{ puts "Hello World"}       # syntax error  
a = { puts "Hello World"}   # syntax error
[1,2,3].each {|x| puts x*2} # only works as part of the syntax of a method call
```

2. 参数列表中最多只能有一个`Block`, 但是`Proc`可以有多个

```
def multiple_procs(proc1, proc2)
  proc1.call
  proc2.call
end

a = Proc.new { puts "First proc" }
b = Proc.new { puts "Second proc" }

multiple_procs(a,b)
```

我们再来看一下`Proc`和`Lambda`的区别:

在我们讨论这两个的区别之前，我们要知道一点，那就是它们都是`Proc`对象。
```
proc = Proc.new { puts "Hello world" }
lam = lambda { puts "Hello World" }

proc.class # returns 'Proc'
lam.class  # returns 'Proc'
```

它们的不同之处在于返回(`return`)的地方不同，且`lambda`会严格检查传入的参数。


```
lam = lambda { |x| puts x }    # creates a lambda that takes 1 argument
lam.call(2)                    # prints out 2
lam.call                       # ArgumentError: wrong number of arguments (0 for 1)
lam.call(1,2,3)                # ArgumentError: wrong number of arguments (3 for 1)

lam = lambda{|x,y,z| puts x,y,z}
lam.call(2,3,4)                # returns nil
```

```
proc = Proc.new { |x| puts x } # creates a proc that takes 1 argument
proc.call(2)                   # prints out 2
proc.call                      # returns nil
proc.call(1,2,3)               # prints out 1 and forgets about the extra arguments

```

```
def lambda_test
  lam = lambda { return }
  lam.call
  puts "Hello world"
end

lambda_test                 # calling lambda_test prints 'Hello World'

#我们可以看见在lam.call之后，仍然运行了puts操作
```

```
def proc_test
  proc = Proc.new { return }
  proc.call
  puts "Hello world"
end

proc_test                 # calling proc_test prints nothing
#而proc则在proc.call时就返回了
```
