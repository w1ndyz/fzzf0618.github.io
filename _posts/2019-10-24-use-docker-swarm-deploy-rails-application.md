---
layout: post
title:  “使用DockerSwarm部署Rails(一)”
date:   2019-10-24 10:24:59
author: Chris Wang
categories: ror
tags: docker swarm rails ci cd
---

在使用docker部署rails项目之前，我们先来介绍gitlab的ci/cd。

Gitlab提供了持续集成(CI Continuous Integration )的服务。在项目中添加`.gitlab-ci.yml`文件，并且在gitlab页面配置一个可以使用的`Runner`，然后每次提交代码时，都会触发CI的pipline。

#### .gitlab-ci.yml

`.gitlab-ci.yml`是用来配置CI在我们的项目中做些什么工作。它位于项目的根目录。

在任何的push操作，GitLab都会寻找`.gitlab-ci.yml`文件，并对此次commit开始jobs，jobs的内容来源于`.gitlab-ci.yml`文件。

一般情况下，运行一个pipline会经过三个阶段: `build`, `test`, `deploy`。这里我们介绍`prebuild`,`build`。这是基于rails的多阶段部署，详情可以查看[Ruby China](https://ruby-china.org/topics/38766)的这一篇帖子。

我们需要先熟悉gitlab-ci的语法，下面是一个示例:

```yml
# runner的几个阶段
stages: 
  - prebuild
  - build

before_script:
  - export MY_DOCKER_IMAGE_TAG=$(cat $CI_PROJECT_DIR/.current-version)
  - export MY_DOCKER_REGISTRY=your_registry_url
  - export MY_REGISTRY_USER=your_user
  - export MY_REGISTRY_PWD=your_password
  - echo $CI_PROJECT_DIR
  - echo $PROJECT_IMAGE_NAME
  - echo $MY_DOCKER_IMAGE_TAG
  - echo $MY_DOCKER_REGISTRY
  - echo $MY_REGISTRY_USER
  - echo $MY_REGISTRY_PWD

variables:
  LC_ALL: C.UTF-8
  LANG: en_US.UTF-8
  LANGUAGE: en_US.UTF-8
  DOCKER_DRIVER: overlay
  PROJECT_IMAGE_NAME: your_image_name

construct_builder:
  stage: prebuild
  image: docker:stable
  services: 
    - docker:dind
  tags:
    - dev
  variables:
    BUILDER_IMAGE_TAG: $PROJECT_IMAGE_NAME/builder:latest
  script:
    - docker login -u "$MY_REGISTRY_USER" -p "$MY_REGISTRY_PWD" $MY_DOCKER_REGISTRY
    - docker pull $MY_DOCKER_REGISTRY/$BUILDER_IMAGE_TAG || echo "No pre-built image found."
    - docker build --cache-from $MY_DOCKER_REGISTRY/$BUILDER_IMAGE_TAG -t $MY_DOCKER_REGISTRY/$BUILDER_IMAGE_TAG -f Dockerfile.builder . || docker build -t $MY_DOCKER_REGISTRY/$BUILDER_IMAGE_TAG -f Dockerfile.builder . 
    - docker push "$MY_DOCKER_REGISTRY/$BUILDER_IMAGE_TAG"

build_image:
  stage: build
  image: docker:stable
  services:
    - docker:dind
  tags:
    - dev
  variables:
    BUILDER_IMAGE_TAG: $PROJECT_IMAGE_NAME/builder:latest    
  script:
    - docker login -u "$MY_REGISTRY_USER" -p "$MY_REGISTRY_PWD" $MY_DOCKER_REGISTRY
    - docker pull $MY_DOCKER_REGISTRY/$BUILDER_IMAGE_TAG
    - docker build --build-arg BUILDER_IMAGE_TAG=$MY_DOCKER_REGISTRY/${BUILDER_IMAGE_TAG} -t "$MY_DOCKER_REGISTRY/$PROJECT_IMAGE_NAME:$MY_DOCKER_IMAGE_TAG.$CI_BUILD_REF_NAME" .
    - docker push "$MY_DOCKER_REGISTRY/$PROJECT_IMAGE_NAME:$MY_DOCKER_IMAGE_TAG.$CI_BUILD_REF_NAME"
  only:
    - dev  #不同的分支，跑的ci不同
    
```

#### 配置Runner

Runner可以是虚拟机，VPS，裸机，docker容器，甚至一堆容器。GitLab和Runners通过API通信，所以唯一的要求就是运行Runners的机器可以联网。Runner可以对应一个项目，也可以共享给多个项目。

要创建一个能用的Runner，需要经过以下两步:

* [安装](https://docs.gitlab.com/runner/install/)
* [配置](https://docs.gitlab.com/ee/ci/runners/README.html#registering-a-specific-runner)

具体更多的信息，可以查看这个[页面](https://docs.gitlab.com/ee/ci/introduction/)。