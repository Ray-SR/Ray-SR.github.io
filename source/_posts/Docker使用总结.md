---
title: Docker使用总结
date: 2020-01-08 20:54:40
tags: Docker
categories: Docker
---

# Docker使用总结

Docker是一个开源容器引擎，它可以将我们的开发环境、代码、配置文件等打包到一个容器中，并发布到任意平台。Docker实现的是操作系统级别的虚拟化，区别于传统的虚拟机，它占用资源少，灵活方便，非常适合web应用的自动化打包和发布以及自动化测试和持续集成、发布。

## 基本概念

- **镜像 image**

Docker中镜像的概念类似于虚拟机的镜像，它是一个包含有文件系统的面向Docker引擎的只读模板，简单的理解它就是构建一个容器的基础。我们可以基于镜像去创建一个容器，也可以将容器打包成镜像。Docker的官方镜像仓库(Docker hub)中提供了许多软件的镜像，例如我们执行`docker search python`就可以看到各种python镜像。
![1.png](/img/Docker/1.png)
实际应用中，我们通常会将自己的开发环境、代码、配置等打包成一个镜像，然后将它发布到生产环境中，这样，就不必为环境是否兼容而担心，因为代码运行所需的一切资源都已经包含的镜像当中了，我们只需要基于它创建容器并启动运行即可。

要下载某个镜像，可以直接从Docker官方镜像仓库中拉取，例如执行`docker pull python`就可以直接拉取一个官方python镜像，再执行`docker images`就可以看到电脑中所有镜像了。以后对镜像指定的操作，可以基于镜像的名字，或者镜像ID（IMAGE ID）。
![2.png](/img/Docker/2.png)

- **容器 container**

有了镜像我们就可以创建一个容器，容器类似于一个轻量级的沙盒，Docker利用容器来运行、隔离不同的应用，它是镜像创建的应用实例，各个容器之间互不影响。需要注意，镜像本身是只读的，我们基于一个镜像启动了容器，Docker只是在镜像的上层创建了一个可写层，镜像本身不变。执行`docker ps -a`或者`docker container ls -a`就可以看到所有正在运行或不在运行的容器：
![3.png](/img/Docker/3.png)

我们可以将一个web应用所有环境都放在同一个容器中，但为了解耦，也可以将不同的应用隔离开来，放在不同容器当中。同样的，每个容器都有自己的名字和ID，如上图所示，`IMAGE`一栏表示当前容器是基于哪个镜像创建的；`COMMAND`代表容器启动时执行的命令，通常是某个服务启动的命令，这样容器启动后服务就可以同时启动；`PORTS`则展示了从宿主机到容器的ip、端口映射关系，例如我们想给某个`nginx`应用绑定`80`端口，就可以将宿主机的80端口映射到nginx容器的80端口，这样，访问`http://宿主机ip:80`将相当于访问`nginx`应用了。

关于容器还有一些其他重要概念,主要是在启动容器时可能会使用的一些配置，通常我们使用`docker run -xxx xxx image_name`创建容器，常用的一些参数如下：

--后台运行
通常启动容器后不加特别参数会直接进入容器，如果想让它在后台运行，可以加上`-d`参数

-- 重新启动
通过`--restart=always`的形式指定，增加这项配置可以让容器在出现一些情况时自动重启，保证服务正常运行。restart支持4种参数：
`no`:容器退出时不要自动重启（默认值）
`on-failure[:max-retries]`:只在容器以非0状态码(正常退出)退出时重启，可以指定尝试重启容器的次数
`always`:不管退出状态码是什么始终重启容器，docker daemon将无限次数地重启容器
`unless-stopped`:不管退出状态码是什么始终重启容器，但是不考虑在Docker守护进程启动时就已经停止了的容器

-- root权限
正常启动容器后，进入容器是以root用户执行操作，但它并不完全具有root权限，可以在创建容器时加上`--privileged=true`来赋予容器真正root权限

-- 文件挂载
文件挂载是Docker一个非常重要的特性，我们可以将代码、配置文件等放在宿主机，然后在创建容器时挂载这些目录到容器中，这样，代码更新或者配置更新时，我们只需要更新宿主机路径下的文件，容器内就自动变更了

-- 指定ip
docker容器启动后，宿主机默认ip为`172.17.0.1`，启动的容器按照时间顺序绑定ip,例如`172.17.0.2`，如果想给容器绑定一个固定的ip,可以创建一个docker 网络，然后在创建容器时绑定一个固定ip。例如执行`docker network create --subnet=192.168.0.0/16 staticnet`创建一个名为staticnet的私有网络，网段为`192.168.0.0/16`,之后在创建容器时，就可以通过`--net staticnet --ip 192.168.0.2`来指定容器ip（注意，在这个子网段中，192.168.0.1是宿主机ip）。

- **仓库 repository**

Docker中的仓库用于存放Docker镜像，注意仓库Repository和注册服务器Registry的区别，注册服务器是存放仓库的地方，而仓库是存放镜像的地方。Docker官方镜像仓库是Docker Hub: [https://hub.docker.com/](https://hub.docker.com/)

## 制作镜像
我们可以从Docker Hub中拉取一些需要镜像，然后在这些镜像的基础上，安装配置代码运行所需的环境，从而制作成一个新的镜像，而制作镜像主要有2种方式，以下以制作python镜像为例：

- **docker commit的方式**

--在获取了一个基础镜像后，我们可以基于它启动一个容器，如`docker run -itd --name python_container python /bin/bash`命令创建并启动一个python容器：
![6.png](/img/Docker/6.png)

查看容器
![4.png](/img/Docker/4.png)

--随后执行`docker exec -it python_container /bin/bash`进入容器，我们可以在容器中安装依赖环境，例如将项目中的`requirements.txt`放入容器后执行`pip install -r requirements.txt`，执行完毕，退出容器

--最后，通过`docker commit python_container new_python_image`命令将安装好依赖的python容器打包成一个名为`new_python_image`的镜像，此时通过`docker images`就可以看到新的镜像了.
提交镜像:
![5.png](/img/Docker/5.png)

查看制作好的镜像:
![7.png](/img/Docker/7.png)

- **Dockerfile的方式**

通过docker commit将一个容器打包成镜像操作简便、容易理解，但是在安装依赖、环境过程中，大量缓存、无关内容被添加，并且由于Docker镜像是分层存储的，修改容器只是在当前层进行标记、添加、修改，并不会实际改变上一层，因此使用docker commit 的方式制作镜像，会导致镜像越来越臃肿，同时也让后续维护的人无法得知镜像构建的具体步骤。因此，最好使用Dockerfile来构建镜像。

Dockerfile是一个用来构建镜像的文本文件，其中包含了构建镜像所需的指令和说明，详细文档可参考[https://docs.docker.com/engine/reference/builder/](https://docs.docker.com/engine/reference/builder/)

下面是一个简单的Dockerfile示例：
```
# 基于python3.6.8镜像
FROM python:3.6.8

# 复制Django项目所需的依赖文件清单到容器中的/目录下
COPY requirements.txt /

# 安装项目所需的python依赖
RUN pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple -r /requirements.txt
```
注意，Dockerfile最好和容器所需的其他文件一起单独放在一个文件夹中。编写完Dockerfile，就可以在Dockerfile所在路径通过`docker build -t image_name .`开始制作镜像，等待完成即可

## 创建容器
创建容器最常见的方式是通过`docker run`命令，详细参数可参考：[https://docs.docker.com/engine/reference/commandline/run/](https://docs.docker.com/engine/reference/commandline/run/)

当我们有多个容器需要创建并配置时，可以使用`docker-compose`工具来编排容器并启动，只需要编写一个yaml文件，再通过一个命令就可以创建并启动所有容器，一个简单示例如下：
```yaml
# docker_compose.yaml 配置实例
version: "3"
services: 
    nginx: 
        # 指定镜像
        image: nginx
        
        # 指定容器名称
        container_name: nginx_container
        
        # 端口映射
        ports: 
            - 8080:8080
            
        # 文件挂载路径
        volumes: 
            - /DJANGO_PROJECT:/DJANGO_PROJECT
            - /nginx.conf:/etc/nginx/nginx.conf
            
        # 网络和ip
        networks: 
            extnetwork: 
                ipv4_address: 192.168.0.2
                
        # 启动后指定的命令
        command: nginx -g 'daemon off;'
        
        privileged: true
        restart: always
        
networks: 
    extnetwork: 
        ipam: 
            config: 
            - subnet: 192.168.0.0/16
```
（需要注意，编写yaml时不能使用tag缩进，冒号后必须有空格）

编写完docker_compose.yaml文件，就可以在该文件路径下执行`docker-compose up -d .`,一键创建、启动容器。

## 参考及链接

参考文章：[https://zhuanlan.zhihu.com/p/23599229](https://zhuanlan.zhihu.com/p/23599229)

Docker入门介绍及常用命令：[https://mp.weixin.qq.com/s?__biz=MzAwOTQ4MzY1Nw==&mid=2247488151&idx=1&sn=b65d355055746b8720c0a989b704666a&chksm=9b5fb671ac283f6715f7abc4b2f5b8711ef1b332b8a697a3142531c7691774ed7b7d08936807&mpshare=1&scene=1&srcid=&key=e85982dd5b83123872824b0bdfaf85bf9311bd5553fa62110ac5171516755e9ded814f7adcdd789a6754a7ebfbd51fbd4926aad92bb4ffdd5cb2ea24f9651c799a4039658b35b500fcd1aee6a10384a8&ascene=1&uin=NjE4ODY0Mzg0&devicetype=Windows+10&version=62060739&lang=zh_CN&pass_ticket=L9QR67EoPhd2C5D7R1VdlOjgtM6umNlAXPO7aCcqFh8IFpzRjjKzNQc3ITX03%2Fm7](https://mp.weixin.qq.com/s?__biz=MzAwOTQ4MzY1Nw==&mid=2247488151&idx=1&sn=b65d355055746b8720c0a989b704666a&chksm=9b5fb671ac283f6715f7abc4b2f5b8711ef1b332b8a697a3142531c7691774ed7b7d08936807&mpshare=1&scene=1&srcid=&key=e85982dd5b83123872824b0bdfaf85bf9311bd5553fa62110ac5171516755e9ded814f7adcdd789a6754a7ebfbd51fbd4926aad92bb4ffdd5cb2ea24f9651c799a4039658b35b500fcd1aee6a10384a8&ascene=1&uin=NjE4ODY0Mzg0&devicetype=Windows+10&version=62060739&lang=zh_CN&pass_ticket=L9QR67EoPhd2C5D7R1VdlOjgtM6umNlAXPO7aCcqFh8IFpzRjjKzNQc3ITX03%2Fm7)

使用Docker部署一套Nginx + uWSGI + Django的范例：[https://ray-sr.github.io/2019/12/10/Docker%E9%83%A8%E7%BD%B2Nginx%20+%20uWSGI%20+%20Django%E5%AE%9E%E8%B7%B5/](https://ray-sr.github.io/2019/12/10/Docker%E9%83%A8%E7%BD%B2Nginx%20+%20uWSGI%20+%20Django%E5%AE%9E%E8%B7%B5/)
















