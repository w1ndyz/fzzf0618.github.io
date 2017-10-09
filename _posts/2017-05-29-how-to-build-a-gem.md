---
layout: post
title:  “如何创建一个Gem包”
date:   2017-05-29 10:31:59
author: Chris Wang
categories: ROR
tags: ruby gem
---

## 引言
`Gem`是一个function的外部集成，它将一些常用，通用的方法集成在一起，更方便调用。
* 创建一个`gem`: `bundle gem xxx` xxx是gem的名字。
* 需要修改`.gemspec`的内容,包括`spec.summary`, `spec.description`, `spec.homepage`。
* 通过`spec.add_dependency`添加包的依赖
* `gem build xxx.gemspec`在本地创建`gem`包。
* `gem push xxx`将`gem`包push到`https://rubygems.org`
* 注意`Version`的填写，`gem`的版本号管理，`0.0.0`中第一个0是发布的版本号。中间的0是修改和优化后的版本号，一般随数字递增，这样push到rubygem好管理。