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

`
$ sudo apt-get update
`

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

## 第三步, 安装Rails和nginx

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
下面再来安装nginx：
```
$ sudo apt-get update
$ sudo apt-get install nginx
```
可以使用下面的命令启动或暂停nginx:
```
$ sudo service nginx stop
$ sudo service nginx start
$ sudo service nginx restart
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

这里有一篇文章讲[mysql在ubuntu上的简单操作](http://www.jianshu.com/p/694d7d0a170b), 可以去看看如何创建数据库，创建表。

## 第五步, 安装配置puma。
在你的项目的Gemfile中, 加入这一行:
```
gem 'puma'
```
一般现在的rails 5.1会默认添加。然后再运行
```
bundle install
```
在ubuntu服务器上, `/etc/nginx/nginx.conf`配置一下`nginx.conf`:
```
upstream puma {
  # /var/www/pepsi是你的项目的路径
  server unix:///var/www/pepsi/shared/tmp/sockets/pepsi-puma.sock;
}

server {
  listen 80 default_server deferred;
  # server_name example.com;

  root /var/www/pepsie/current/public;
  access_log /var/www/pepsi/current/log/nginx.access.log;
  error_log /var/www/pepsi/current/log/nginx.error.log info;

  location ^~ /assets/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
  }

  try_files $uri/index.html $uri @puma;
  location @puma {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;

    proxy_pass http://puma;
  }

  error_page 500 502 503 504 /500.html;
  client_max_body_size 10M;
  keepalive_timeout 10;
}

```

## 第六步, 配置capistrano。
在你项目的Gemfile中添加如下的几行:
```
gem 'capistrano', '~> 3.8'
gem 'capistrano-bundler'
gem 'capistrano-rails'
gem 'capistrano-rbenv'
gem 'capistrano3-puma'
```
`bundle install`之后，运行下面的命令配置cap:
```
cap install
```
这个命令将会创建:
1. 根目录下创建`Capfile`。
2. config目录下创建`deploy.rb`
3. config目录下创建deploy文件夹，这里面主要存放各个环境的配置内容。

复制下面的代码到`Capfile`:
```
# Load DSL and set up stages
require "capistrano/setup"

# Include default deployment tasks
require "capistrano/deploy"

# Load the SCM plugin appropriate to your project:
#
# require "capistrano/scm/hg"
# install_plugin Capistrano::SCM::Hg
# or
# require "capistrano/scm/svn"
# install_plugin Capistrano::SCM::Svn
# or
require "capistrano/scm/git"
install_plugin Capistrano::SCM::Git

# Include tasks from other gems included in your Gemfile
#
# For documentation on these, see for example:
#
#   https://github.com/capistrano/rvm
#   https://github.com/capistrano/rbenv
#   https://github.com/capistrano/chruby
#   https://github.com/capistrano/bundler
#   https://github.com/capistrano/rails
#   https://github.com/capistrano/passenger
#
# require "capistrano/rvm"
require "capistrano/rbenv"
# require "capistrano/chruby"
require "capistrano/bundler"
require 'capistrano/puma'
require "capistrano/rails/assets"
require "capistrano/rails/migrations"
# require "capistrano/passenger"
require 'capistrano/rails'
# require 'capistrano/runit/puma'
# Load custom tasks from `lib/capistrano/tasks` if you have any defined
Dir.glob("lib/capistrano/tasks/*.rake").each { |r| import r }

```
它将预加载你的配置内容。然后我们再替换`config/deploy.rb`为以下内容:
```
# config valid only for current version of Capistrano
lock "3.8.2"


append :linked_files, "config/database.yml", "config/secrets.yml"
append :linked_dirs, "log", "tmp/pids", "tmp/cache", "tmp/sockets", "public/system", "node_modules"
set :keep_releases, 5
set :rbenv_type, :user # or :system, depends on your rbenv setup
set :rbenv_ruby, '2.4.1'
set :rbenv_map_bins, %w{rake gem bundle ruby rails}
set :rbenv_roles, :all # default value
set :pty, true

namespace :puma do
  desc 'Create Directories for Puma Pids and Socket'
  task :make_dirs do
    on roles(:app) do
      execute "mkdir #{shared_path}/tmp/sockets -p"
      execute "mkdir #{shared_path}/tmp/pids -p"
    end
  end

  before :start, :make_dirs
end

namespace :deploy do
  desc "Make sure local git is in sync with remote."
  task :check_revision do
    on roles(:app) do
      unless `git rev-parse HEAD` == `git rev-parse origin/master`
        puts "WARNING: HEAD is not the same as origin/master"
        puts "Run `git push` to sync changes."
        exit
      end
    end
  end

  desc 'Initial Deploy'
  task :initial do
    on roles(:app) do
      before 'deploy:restart', 'puma:start'
      invoke 'deploy'
    end
  end

  # desc 'Restart application'
  # task :restart do
  #   on roles(:app), in: :sequence, wait: 5 do
  #     Rake::Task["puma:restart"].reenable
  #     invoke 'puma:restart'
  #   end
  # end

  before :starting,     :check_revision
  after  :finishing,    :compile_assets
  after  :finishing,    :cleanup
  # after  :finishing,    :restart
end

```
在`config/deploy/production.rb`下替换一下内容(production是生产环境，你也可以配置staging.rb的测试环境文件)：
```
set :stage, :production
#'**.***.**.**'这里填写你自己服务器的IP
server '**.***.**.**' , user: 'deploy', roles: %w{web app db}, port: 25000
#项目名称
set :application, "pepsi"
#这里填写你自己的github项目
set :repo_url, "git@github.com:xxx/xxx.git"
set :branch, 'master'
#服务器上你自己的项目路径
set :deploy_to, "/var/www/pepsi"
set :user, "deploy"
set :use_sudo, true
set :ssh_options, {
 forward_agent: false,
 user: 'deploy',
 port: 25000
}

set :puma_role, :app
set :deploy_via,      :remote_cache
set :puma_bind,       "unix://#{shared_path}/tmp/sockets/#{fetch(:application)}-puma.sock"
set :puma_state,      "#{shared_path}/tmp/pids/puma.state"
set :puma_pid,        "#{shared_path}/tmp/pids/puma.pid"
set :puma_access_log, "#{release_path}/log/puma.error.log"
set :puma_error_log,  "#{release_path}/log/puma.access.log"
set :puma_preload_app, true
set :puma_worker_timeout, nil
set :puma_init_active_record, true  # Change to false when not using ActiveRecord
set :puma_threads, [0, 16]
set :puma_workers, 0
set :puma_init_active_record, false
set :puma_preload_app, true

```
然后就可以提交你的代码到github。
```
$ git add -A
$ git commit -m "Set up Puma, Nginx & Capistrano"
$ git push origin master
```

最后在你本地机器运行:
```
$ cap production deploy:initial

```
这个命令会将你的应用部署到服务器，等待的时间会有一点久。有可能会有一些error存在，你需要去stackoverflow上查找答案，直到你配置正确。

## 结论
Rails App 会有很多种部署方式，cap只是其中的一种，mina和他比较更为小巧。有兴趣的可以去看看mina的部署，google上的文章也有很多。如果有什么不明白的，也欢迎给我发邮件。
