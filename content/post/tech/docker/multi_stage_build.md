---
author: "Dennis Xie"
title: "Multi-Stage Builds of Docker"
date: 2022-06-21T16:32:52+08:00
description: ""
draft: false
tags: ["docker"]
categories: ["tech"]
postid: "82dfdefba4df3346024c94216677faadb9f7dee7"
---
## Background
What can we do if we should merge several Docker images to a single Docker image for some reason?
## Solution
We can use multi-stage builds of Docker.
## Multi-stage builds
### Problem solved by multi-stage builds
It's very common to have different images for building, testing, and production. Different images contain different tools. Building images may contain building tools such as GCC, Gradle, and so on. Testing images may contain perf tools, analysis tools, and so on. The image for production is the slimmed-down one and only contains the tools for running.

If we can't use multi-stage builds, we need a script to create the building, testing, and production images. The pseudo-code is shown below:
~~~bash
docker build -t demo:build -f Dockerfile.build .
docker container create --name demo demo:build
docker container cp demo:/path/to/build/app ./app
docker container rm -f demo
docker build -t demo:deploy-latest -f Dockerfile .      # copy the app into the deploy image
~~~
We can ensure that the images keep slim in this way. We can move these steps into the docker building stage with the help of multi-stage builds.
### The way to use multi-stage builds
Multi-stage builds divide the image-building process into several stages. The different stages can use different bases. We can selectively copy artifacts from the previous stage to the current stage. The pseudo-code is shown below:
~~~docker
FROM base0              # stage without a name
RUN build something

FROM base1 AS stage1    # stage with a name
RUN build other things

FROM runningiamge:latest
WORKDIR /running/path
COPY --from=0 /base0/build/app ./app
COPY --from=stage1 /base1/build/files ./statics
ENTRYPOINT ["./app"]
~~~
As you can see, the multi-stage builds just use several FROM instructions. The image is built by the final stage. The stage is numbered from 0 to N. The stage can also be named with AS instruction. We can use --from parameter in the COPY instruction to copy some artifacts from the pervious stage to the current image.
## The way to solve the background problem
Assume we already have two images, one is the front-end image and the other is the back-end image. Now, we need to merge these two images into a single image.
### The front-end Dockerfile
Dockerfile:
~~~docker
from alpine:3.16

WORKDIR /home/fe/
COPY ./statics ./
~~~
building command:
~~~
docker build -t demofe:latest -f Dockerfile.fe .
~~~
### The back-end Dockerfile
Dockerfile:
~~~docker
from gradle:7.4.2-jdk11 AS javabuild

WORKDIR /home/build/
COPY ./ ./
RUN GRADLE_USER_HOME=`pwd`/.gradle gradle build

ENTRYPOINT ["java", "-jar", "build/libs/demo-0.0.1-SNAPSHOT.jar"]
~~~
building command:
~~~sh
docker build -t demofe:latest -f Dockerfile.fe .
~~~
### Merging Dockerfile
Dockerfile:
~~~docker
FROM demofe:latest

FROM demobe:latest

FROM openjdk:11.0.15-oraclelinux7
WORKDIR /home/demo/
COPY --from=0 /home/fe ./statics
COPY --from=1  /home/build/build/libs/*.jar ./

ENTRYPOINT ["java", "-jar", "demo-0.0.1-SNAPSHOT.jar"]
~~~
building command:
~~~
docker build -t demo:latest -f Dockerfile .
~~~
By doing this, we can merge these two images into one single image.
# Reference
1. [Multi-stage builds](https://docs.docker.com/develop/develop-images/multistage-build/)
