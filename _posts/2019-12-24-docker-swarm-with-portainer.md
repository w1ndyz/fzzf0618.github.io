---
layout: post
title:  “使用DockerSwarm部署Rails(三)”
date:   2019-12-24 10:24:59
author: Chris Wang
categories: ror
tags: docker swarm portainer config
---

#### Portainer

在使用Docker swarm部署项目时，我们先要了解一个工具--[Portainer](https://github.com/portainer/portainer)。

Portainer是一个轻量级的管理Docker的UI。他允许我们管理所有的Docker资源(容器，映像，卷，网络等)。他与独立的Docker引擎和Docker Swarm模式兼容。

#### Docker Swarm

Docker Swarm 是Docker引擎内置(原生)的集群管理和编排工具。

使用 `Swarm` 集群之前需要了解以下几个概念。

* 节点
* 服务和任务

节点分为管理(manager)节点和工作(worker)节点。一个Swarm可以有多个管理节点，但只有一个管理节点可以成为`leader`，`leader`通过`raft`协议实现。工作节点是任务执行的节点，管理节点将任务或服务(service)下发到工作节点执行。管理节点默认也作为工作节点。下面这张图片展示了集群中管理节点和工作节点的关系:

![](/assets/swarm-diagram.png)

任务(Task)是Swarm中的最小调度单位,目前来说就是一个单一的容器。

服务(Services)是指一组任务的集合，服务有两种模式:

* `replicated services` 按照一定规则在各个工作节点上运行指定个数的任务。
* `global services` 每个工作节点上运行一个任务

两种模式通过 `docker service create` 的 `--mode` 参数指定。

接下来我们来模拟Docker Swarm的部署，从下面的示例来观察一些细节。

#### 部署示例

前期准备:

* web服务器2台
* 数据库1台
* 队列1台

##### 上传docker-compose及配置文件至cron队列服务器，将队列服务器作为swarm的manager节点。

```shell
# 本地
$ rsync app-docker.tar cron:/data/

# cron服务器
$ cd /data
$ tar -xvf app-docker.tar
# 安装docker
$ apt install docker.io -y
# 创建docker swarm
$ docker swarm init --advertise-addr your_ip

​```
Swarm initialized: current node (s1fof4pev4rj8mzn0q8ymmqx4) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-508f7r079pxy9ggos64nhm50dznkdmni4gafmj6aqhbr7hd86g-6pbu4jfmn3wm4hu00h1593s3q 192.168.0.1:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
​```


# 进入其他两台web服务器
$ docker swarm join --token SWMTKN-1-508f7r079pxy9ggos64nhm50dznkdmni4gafmj6aqhbr7hd86g-6pbu4jfmn3wm4hu00h1593s3q 192.168.0.1:2377

​```
This node joined a swarm as a worker.
​```

# cron服务器查看各个节点信息
$ docker node ls

​```
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
ium2q83ucn30m62q0oz6y0pq4     appweb1            Ready               Active                                  18.09.7
zz22kpep91y35v62s3yrvg7yp     appweb2            Ready               Active                                  18.09.7
s1fof4pev4rj8mzn0q8ymmqx4 *   quene               Ready               Active              Leader              18.09.7
​```

# 修改conf/secrets.yml中的队列开关
​```
	# job
  schedule_job_enabled: true
​```
```

##### 创建Docker Swarm监控

```shell
# 启动docker监控portainer
$ docker stack deploy --compose-file=portainer-agent-stack.yml portainer
```

其中`portainer-agent-stack.yml`的配置可以看[这里](https://portainer.readthedocs.io/en/latest/deployment.html)。

##### 接下来第一次部署，登陆服务器拉取最新的镜像

```shell
# 登陆私有docker registry仓库
$ docker login -u username -p pwd xxxxx.com:5000

# 拉取最新的镜像
$ docker pull xxxxx.com:5000/app:1.0.0.master
```

##### 创建配置文件，并进行部署

```shell
# 创建docker配置文件
$ ./scripts/init_config

# 启动容器部署
$ docker stack deploy --with-registry-auth -c docker-compose.yml app_name

# 查看容器运行状态
$ docker service ls
```

其中`init_config`文件

```shell
#!/bin/bash

cd `pwd`/config
conf_size=`docker config ls|grep database|wc -l`

if [ $conf_size -gt 0 ]; then
  docker config rm database_conf \
    && docker config rm mongoid_conf \
    && docker config rm secrets_conf \
    && docker config rm puma_conf \
    && docker config rm sidekiq_conf \
fi

docker config create database_conf database.yml \
  && docker config create mongoid_conf mongoid.yml \
  && docker config create secrets_conf secrets.yml \
  && docker config create puma_conf puma.rb \
  && docker config create sidekiq_conf sidekiq.yml

```

`docker config `子命令，用来管理集群中的配置信息，无需将配置文件放入镜像或挂载到容器中就可以实现对服务的配置。

##### 关于监控Portainer的使用

* 在`Registries`新增自己的私有镜像仓库
* 每次更新可以先在`Images`下拉取最新的镜像到对应的服务器中
* 更新可以进入`Stacks`点击对应的服务进去，选择并`Update`，可勾选是否使用最新的镜像
* 配置文件可以在`Configs`中查看
* 日志可以在`Services`中点击对应的服务进入，点击`Service logs`查看日志
* 更新失败的日志，可以在`Stacks`中点击对应服务器的日志查看

##### 关于配置文件的滚动更新

当需要修改config配置的内容时，创建新配置（使用docker config create）然后更新服务以删除先前配置的config，并添加对新配置是一种常见模式。服务命令`--config-rm`和`--config-add`:

比如我修改了secrets.yml，而现在在执行`docker config ls`时候已经存在secrets_conf

这时候我执行下面命令，重新创建个新的配置，比如创建secrets_conf_1
docker config create secrets_conf_1 config/secrets.yml

然后执行下面进行config更新
docker service update --config-rm database_log_conf --config-add src=database_log_conf_1,target=/home/app/app_name/config/database_log.yml app_name