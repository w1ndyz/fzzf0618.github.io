---
layout: post
title:  “How to deploy Rails app with Cap3, puma and Nginx  on Ubuntu 14.04!”
date:   2017-02-18 08:43:59
author: Chris Wang
categories: ROR
tags:	deploy rails capistrano3 puma mysql nginx
cover:  "/assets/instacode.png"
---

## 简介
Ruby On Rails 应用的部署方式有很多种。比如capistrano, mina, heroku, passenger等等。在这里介绍的是capistrano3, 在Ubuntu 14.04的机器上部署, 所需要的包括ruby的环境, rbenv版本控制, nginx, mysql, puma, ssh的设置等等。

## 第一步, 设置ssh登录
这里推荐学习阮一峰老师的[Linux服务器的初步配置流程](http://www.ruanyifeng.com/blog/2014/03/server_setup.html)。在这里面需要注意的是:一般像Aliyun的服务器,或者腾讯云的服务器,完成前面三个步骤就行了。

## 第二步, 安装Rbenv, 并且配置Ruby
首先，我们安装Rbenv

```
$ sudo apt-get update
```

安装rbenv和ruby的一些依赖

```
$ sudo apt-get install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev python-software-properties libffi-dev
```

现在我们安装好了rbenv, 运行下面的命令来配置好它。

```
$ cd
$ git clone git://github.com/sstephenson/rbenv.git .rbenv
$ echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
$ echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
$
$ git clone git://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
$ echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bash_profile
$ source ~/.bash_profile
```

现在我们来安装ruby。

```
rbenv install -v 2.4.0
rbenv global 2.4.0
```
参数`global`表示全局设置ruby的默认版本。如果想设置其他的版本，可以参考rbenv的其他命令去配置，这里就不详细说明了。

下面，我们来验证一下ruby的版本。

```
$ ruby -v
```
你可能还需要安装`bundler`去管理你的依赖。

```
$ gem install bundler
```

## 第三步, 安装Rails

使用相同的用户, 安装不同的rails版本可以用命令`-v`来控制。
```
$ gem install rails
```
可以运行下面的命令, 让rbenv使用可执行文件。
```
$ gem install rails
```
验证你的rails版本, 可以用：
```
$ rails -v
```

## 第四步, 安装对应的数据库mysql(以mysql为例)。

你需要运行下面的命令。
```
$ sudo apt-get update
$ sudo apt-get install mysql-server
```
我们是以mysql 5.6为例子来安装的。
```
$ sudo apt-get update
$ sudo apt-get install mysql-server-5.6
```
下面配置你的mysql。
```
$ sudo mysql_secure_installation
```
运行这个命令之后, 会创建你的password, 你可以直接回车不设置, 也可以设置, 你需要记牢的你的password。接下来, 会创建mysql的目录。安装成功后, 可以运行下面的命令查看版本:
```
$ mysql --version
```
你会看到和下面相似的回复:
```
mysql  Ver 14.14 Distrib 5.6.0, for debian-linux-gnu (x86_64) using readline 6.3
```
