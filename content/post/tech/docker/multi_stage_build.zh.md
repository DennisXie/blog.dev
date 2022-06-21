---
author: "Dennis Xie"
title: "Docker多阶段构建"
date: 2022-06-21T16:38:22+08:00
description: ""
draft: true
tags: ["docker"]
categories: ["tech"]
postid: "849a0935917b12a0dce7be17c44c470d0e749817"
---
## 问题背景
如果有这么个情况，因为某些原因需要合并多个docker镜像里面的文件来组成一个单一的镜像，应该怎么办？
## 解决方案
使用Docker的多阶段构建。
## 多阶段构建
### 多阶段构建解决的问题
通常来说服务构建、测试和部署一般会使用不同的镜像。其中不同的镜像会有一些不同的工具，如构建镜像会包含一些基础构建工具。测试镜像会包含一些分析测试结果的工具。部署镜像则是最精简的镜像，仅包含正常运行所需要的工具和环境等。

在没有多阶段构建的情况下，我们可能会一个脚本来完成程序构建、测试镜像的生成和部署镜像的生成等工作。如下面的代码所示：
~~~bash
docker build -t demo:build -f Dockerfile.build .
docker container create --name demo demo:build
docker container cp demo:/path/to/build/app ./app
docker container rm -f demo
docker build -t demo:deploy-latest -f Dockerfile .      # copy the app into the deploy image
~~~
这样可以保证不同用途的镜像的精简程度，尤其是避免生产环境使用的镜像太肥。有了多阶段构建以后，我们就可以把这些操作移动到Docker镜像构建中去。
### 使用方法
多阶段构建可以将Docker镜像的构建划分成多个不同阶段，不同阶段使用不同的基础镜像，后面的构建阶段可以使用前面阶段的一些结果。示例代码如下:
~~~docker
FROM base0              # stage without name
RUN build something

FROM base1 AS stage1    # stage with a name
RUN build other things

FROM runningiamge:latest
WORKDIR /running/path
COPY --from=0 /base0/build/app ./app
COPY --from=stage1 /base1/build/files ./statics
ENTRYPOINT ["./app"]
~~~
上面代码中可以看到使用多阶段构建以后，Dockerfile的变化就是多了几个FROM语句。最终生成的镜像为最后一个阶段构建的结果。构建阶段默认为0, 1, 2... N-1的编号, 同时构建阶段也可以使用AS指令进行命名。注意代码中的COPY命令，--from参数可以指定构建阶段编号或者阶段名, 从指定的构建阶段的结果中复制内容来构建当前的镜像。
## 开篇问题解决示例
假如我们已有的两个镜像分别是前端镜像和后端镜像，现在要将两个镜像合并到一起。
### 前端镜像Dockerfile
Dockerfile(Dockerfile.fe)如下:
~~~docker
from alpine:3.16

WORKDIR /home/fe/
COPY ./statics ./
~~~
打包命令如下:
~~~sh
docker build -t demofe:latest -f Dockerfile.fe .
~~~
### 后端镜像Dockerfile
Dockerfile(Dockerfile.be)如下:
~~~docker
from gradle:7.4.2-jdk11 AS javabuild

WORKDIR /home/build/
COPY ./ ./
RUN GRADLE_USER_HOME=`pwd`/.gradle gradle build

ENTRYPOINT ["java", "-jar", "build/libs/demo-0.0.1-SNAPSHOT.jar"]
~~~
打包命令如下:
~~~sh
docker build -t demofe:latest -f Dockerfile.fe .
~~~
### 整合两个镜像
Dockerfile如下:
~~~docker
FROM demofe:latest

FROM demobe:latest

FROM openjdk:11.0.15-oraclelinux7
WORKDIR /home/demo/
COPY --from=0 /home/fe ./statics
COPY --from=1  /home/build/build/libs/*.jar ./

ENTRYPOINT ["java", "-jar", "demo-0.0.1-SNAPSHOT.jar"]
~~~
打包命令如下:
~~~
docker build -t demo:latest -f Dockerfile .
~~~
通过上面的多阶段构建Dockerfile即可整合多个镜像中的文件。
# 引用
1. [Multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/)
