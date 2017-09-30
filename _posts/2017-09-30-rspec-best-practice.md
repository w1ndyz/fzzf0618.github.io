---
layout: post
title:  “RSpec的最佳实践”
date:   2017-09-30 15:34:59
author: Chris Wang
categories: ROR
tags: rspec best practice
---

## 关于RSpec
RSpec是一个行为驱动测试框架(BDD)。可以写出高可读性的测试，来测试出我们所开发的程序。在RSpec中有很多方法关键字，下面我来介绍一下。
* `describe`，`context`，`subject`

首先来看一下`describe`,`describe`是用来描述方法的，我们要先弄清楚是什么样的方法，是类方法还是实例方法:
```
describe '.authenticate' do # 类方法用 .
describe '#admin?' do # 实例方法用 #

```
`context`可以让测试更加有条理，保持高度的可读性:
```
# 一般context的开头用when或者with
context 'when login' do
  it 'returns all results' do
        expect(results.length).to eq(2)
  end
end
```


`subject`是共享变量，当很多个测试都用到同一个变量的时候，可以用`subject{}`来避免重复:
```
subject { create('user') }
it '....' do
    expect(subject.function_name).to eq(...)
end

# subject也可以命名
subject(:order) { create('order') }
it 'active a order' do
    expect(order.active).to be true
end
```
* `let`和`let!`

`let`是用来创建初始数据的。和`before`是一样的。但是比`before(:each)`执行的速度更快。`let {...}`后面的`block`只执行一次,然后缓存起来，提高执行效率。`let`和`let!`的区别是，`let!`是立即创建数据,`let`是在需要的时候创建。
```
describe '#run' do
  subject      { described_class.new(params).run }
  let(:course) { create(:course) }
  let!(:course_process_version) { create(:course_process_version, course: course) }
  let(:params) do
    {
      course_id: course.id,
      name: 'Test',
      category: 'student',
      page_identifier: 'S001'
    }
  end
  it 'returns the evidence requirement course' do
      expect(subject).to include(course: course)
  end
end
```

在测试的时候，我们要考虑到所有的情况。比方说删除一个数据，通常我们只用测试是否删除成功，其实我们还要考虑到的是找不到这个数据怎么办，没有权限删除这条数据怎么办，这些都是要考虑到的。所以现在往往基于`model`的测试并不多了，现在更多的是基于用户行为的测试，基于`view`层，模仿用户的操作。

另外，在造数据的时候，我们通常使用`FactoryGirl`，还有一些假数据可以用`Faker`。要减少在`rspec`文件中写类似于:
```
user = user.create(
    name: xxx,
    age: xx,
    city: xx
)
```
这种写法，太过于冗长，不够方便。