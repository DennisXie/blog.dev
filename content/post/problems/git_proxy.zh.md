---
author: "Dennis Xie"
title: "Git访问慢问题处理方法"
date: 2022-08-09T16:05:32+08:00
description: ""
draft: false
tags: ["problems"]
categories: ["tech"]
postid: "02e07b86d9950726823604f7e85c41a0af0b2ff3"
---
# 问题
某些时候，我们拉取Git仓库的代码会遇到一些奇奇怪怪的网络问题。这些问题包含但不限于以下一些问题：

1. 拉取速度特别慢
2. 一些莫名其妙的TLS问题
3. 网络中断

# 处理方式
遇到以上问题可以尝试使用代理来解决。在Github上，远程仓库一般使用https协议或者ssh协议。下面分别提供两种协议的代理的配置方法。
## https协议
为Git命令行添加http代理配置即可，命令如下:
~~~Bash
git config --global http.proxy http://proxyUsername:proxyPassword@proxy.server.com:port
~~~
运行该命令后, ~/.gitconfig文件中会增加如下的配置:
~~~
[http]
        proxy = http://proxyUsername:proxyPassword@proxy.server.com:port
~~~
通过该方法可以解决https协议的远程仓库访问问题。
## ssh协议
使用ssh协议的仓库链接一般为 _git@github.com:gituser/repository.git_。这里User为git，Host为github.com。因为使用了ssh协议，配置一下ssh的代理即可。首先打开或者创建 _~/.ssh/config_ 文件。以Github为例，填入以下内容:
~~~
Host *.github.com
    User git
    ProxyCommand nc -v -x {proxy.server.com or ip}:{port} %h %p
~~~
如果要针对其他Git仓库代理，Host需要修改成对应域名，User也需要以仓库的地址为准。这里的代理命令使用了nc命令，所以需要系统装了netcat才行。
> **_tips: git@github.com:gituser/repository.git实际上是ssh://git@github.com/gituser/repository.git的简写_**

# 其他问题
我自己是在WSL2中配置的代理，代理服务运行在Windows中。这里就出现了WSL2访问Windows的服务的问题。在WSL2中，可以使用$(hostname).local来获取到Windows的域名。由于Windows的防火墙问题，这时实际上还是不能访问成功。有以下几种解决方案可以尝试：
1. 增加防火墙入站规则(这个方法在我本机上还没有实验通过)
2. 配置公用配置文件的受保护的网络连接
   ![image](/images/post/problems/git_proxy/windows_defender.png)
    
   取消掉图中WSL的网络连接。这个方法需要每次重启后配置一次，因为WSL这个网络需要WSL启动后才会出现。
3. 配置允许通过防火墙的应用
   
   控制面板->系统和安全->Windows Defender 防火墙->允许的应用，把需要访问的服务的公有网络打开即可。如果你在用IDEA调试程序，那么你需要把Java的公有网络打开才能在WSL2里访问这个在调试的服务。
4. 关闭公用网络防火墙

# 引用
1. [Configure Git to use a proxy](https://gist.github.com/evantoli/f8c23a37eb3558ab8765)
2. [git 设置和取消代理](https://gist.github.com/laispace/666dd7b27e9116faece6)
