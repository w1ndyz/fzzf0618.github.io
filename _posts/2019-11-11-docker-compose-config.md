---
layout: post
title:  “使用DockerSwarm部署Rails(二)”
date:   2019-11-11 10:24:59
author: Chris Wang
categories: ror
tags: docker compose multi stage
---

#### 多阶段构建

在构建镜像的时候，我们会发现:

* rails的应用体积很大
* 构建时间过长

于是多阶段构建出现了，它将一个镜像的构建分为多个阶段。比如rails应用，ruby环境为一个构建阶段，rails应用所需的数据库，webpacker为下一个构建阶段。

具体构建方式，可参照[高效构建镜像](https://blog.wildcat.io/2019/06/rails-with-docker-part-1-zh/),这篇blog详细介绍了各个构建阶段的步骤，以及为何要这么做。可根据自己的需要，构建自己的rails镜像。

#### Docker Compose

Docker Compose 是一个轻松、高效的管理容器，他是一个用于定义和运行多容器的Docker的应用程序。他将所管理的容器分为了三层:

* 工程 project
* 服务 service
* 容器 container

Docker Compose 会运行目录下的`docker-compsoe.yml`配置文件组成一个工程，一个工程将包含多个服务，每个服务又定义了容器运行的镜像、参数、依赖，一个服务也可包含多个容器实例。

下面将会举一个例子来看看，具体的compose文件如何配置:

```yaml
---
#声明docker-compose的版本
version: "3.6"

services:
  app: &app_base
    container_name: 'app
    env_file:
      - docker.env
    #镜像地址
    image: app.1.0.master
    volumes:
      - assets:/home/app/app/public
      - /etc/localtime:/etc/localtime:ro
    configs:
      - source: database_conf
        target: /home/app/app_name/config/database.yml
      - source: mongoid_conf
        target: /home/app/app_name/config/mongoid.yml
      - source: puma_conf
        target: /home/app/app_name/config/puma.rb
      - source: secrets_conf
        target: /home/app/app_name/config/secrets.yml
      - source: sidekiq_conf
        target: /home/app/app_name/config/sidekiq.yml
    logging:
      options:
        max-size: "1g"
        max-file: "10"
    command: /home/app/app_name/bin/docker-start
    #docker-swarm设置worker角色
    deploy:
      replicas: 2
      placement:
        constraints: [node.role == worker]
    ports:
      - "7000:7000"
    #健康检查，用于服务器挂了之后自动重启
    healthcheck:
      test: "curl -fs localhost:7000 || exit 1"
      interval: 30s
      timeout: 40s
      retries: 60
    networks:
      - overlay
      
 #sidekiq
  worker:
    <<: *app_base
    container_name: 'app_name_worker'
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]
    ports: []
    command: bundle exec sidekiq -C config/sidekiq.yml -L /home/app/app_name/log/sidekiq.log
    healthcheck:
      test: "stat /home/app/app_name/tmp/sidekiq.pid || exit 1"
      interval: 30s
      timeout: 40s
      retries: 60

configs:
  database_conf:
    external: true
  mongoid_conf:
    external: true
  secrets_conf:
    external: true
  puma_conf:
    external: true
  sidekiq_conf:
    external: true

networks:
  overlay:

volumes:
  assets:

```

上面代码所用到的`bin/docker-start`

```shell
#!/usr/bin/env sh
bundle exec rails assets:precompile
bundle exec rails db:migrate
bundle exec puma -C config/puma.rb

```

在后面我们将会使用docker-compose来构建一个rails应用。

[注释]
* [Docker Compose简介及使用](https://yeasy.gitbooks.io/docker_practice/compose/introduction.html)