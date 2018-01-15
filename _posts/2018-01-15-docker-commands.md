---
layout: post
title:  “Docker基本命令详解”
date:   2018-01-15 11:38:59
author: Chris Wang
categories: Docker
tags: docker commads 详解
---

最近在学习如何部署`Rails App`在`Docker`上时，将所有的docker命令整理了一下，如下：
 
 `docker --help` 查看所有的docker命令
##### Commands
* `attach` 进入docker容器
```
$ docker ps -a
# 列出所有的容器
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS              PORTS                               NAMES
b74da051b403        blablablabla   "/bin/sh -c 'python …"   12 minutes ago      Up 12 minutes       51438/tcp, 0.0.0.0:8532->8888/tcp   www
$ docker attach b74da051b403
# 进入ID为b74da051b403的容器
# 我们还可以通过ssh, nsenter, exec进入, 通过inspect查看容器详细信息
```
* `build` 通过`Dockerfile`创建镜像
* `commit` 创建本地镜像推送到docker pub中。`$ docker push user_name/image_name`
* `create` 创建一个新的容器
* `diff` 类似于github, 查看容器内发生改变的文件
* `cp` 在容器和主机间相互拷贝文件
* `export` 将文件系统打包成`tar`
* `import` 和`export`对应, 引入`tar`文件，创建新的镜像
* `images` 列出所有镜像
* `info` 查看docker的系统信息, 与`inspect`不同的是, info显示的是docker在主机系统中的内存占用, inspect显示一些配置信息，例如环境变量
* `kill`, `stop` (1)强制终止容器。(2)终止容器。`stop`有option`-t`，选择终止延时时间
* `login` 登录docker Hub，使用`logout`退出
* `logs` 查看容器日志
* `ps` 查看所有容器
* `pull` 从docker pub下载你想要的镜像

```
$ docker search ubuntu
# Output

NAME                                                   DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
ubuntu                                                 Ubuntu is a Debian-based Linux operating sys…   7106                [OK]
dorowu/ubuntu-desktop-lxde-vnc                         Ubuntu with openssh-server and NoVNC            156                                     [OK]
rastasheep/ubuntu-sshd                                 Dockerized SSH service, built on top of offi…   127                                     [OK]
ansible/ubuntu14.04-ansible                            Ubuntu 14.04 LTS with ansible                   90                                      [OK]
ubuntu-upstart                                         Upstart is an event-based replacement for th…   80                  [OK]
neurodebian                                            NeuroDebian provides neuroscience research s…   41                  [OK]
ubuntu-debootstrap                                     debootstrap --variant=minbase --components=m…   34                  [OK]
1and1internet/ubuntu-16-nginx-php-phpmyadmin-mysql-5   ubuntu-16-nginx-php-phpmyadmin-mysql-5          23                                      [OK]
nuagebec/ubuntu                                        Simple always updated Ubuntu docker images w…   22                                      [OK]
tutum/ubuntu                                           Simple Ubuntu docker images with SSH access     19
ppc64le/ubuntu                                         Ubuntu is a Debian-based Linux operating sys…   11
i386/ubuntu                                            Ubuntu is a Debian-based Linux operating sys…   8
1and1internet/ubuntu-16-apache-php-7.0                 ubuntu-16-apache-php-7.0                        6                                       [OK]
..........

$ docker pull ubuntu

```

* `push` 与`pull`对应，将镜像上传到docker pub
* `rename` 重命名容器
* `restart` 重启
* `rm` 通过ID删除容器
* `rmi` 删除镜像
* `stats` 动态显示容器的资源消耗，如: CPU,IO,NETWORK...
* `top` 查看容器中正在运行的进程
* `version` 查看docker的版本