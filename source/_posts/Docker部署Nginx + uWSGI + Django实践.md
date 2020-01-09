---
title: Docker部署Nginx + uWSGI + Django实践
date: 2019-12-10 20:29:49
tags: [Docker,Django,Nginx]
categories: Docker
---

## 简介
Docker容器技术是目前热门的代码部署手段，通过将代码、配置文件、开发环境等打包到一个容器中，就能将其快速部署到生产环境，十分方便快捷。本文将介绍如何使用Docker来部署一套Nginx + uWSGI + Django的生产环境web服务器（Centos7系统）.

<!--more-->

阅读前需要对一些基本技术有所了解，以下仅供参考：

Docker:  [https://yeasy.gitbooks.io/docker_practice/content/introduction/what.html](https://yeasy.gitbooks.io/docker_practice/content/introduction/what.html)
 [https://zhuanlan.zhihu.com/p/23599229](https://zhuanlan.zhihu.com/p/23599229)
Nginx、uWSGI：[https://blog.csdn.net/weixin_40907382/article/details/80824167](https://blog.csdn.net/weixin_40907382/article/details/80824167)

## Docker安装

```shell
# 查看centos系统版本，内核版本要求不低于3.10
uname -r

# 更新yum
sudo yum -y update

# 安装需要的软件包
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

# 设置yum源
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 查看仓库中所有docker版本
yum list docker-ce --showduplicates | sort -r

# 选择其中一个版本安装
yum -y install docker-ce-18.06.3.ce

# 启动Docker服务，并设置为开机启动
systemctl start docker
systemctl enable docker

# 测试是否安装成功
docker version

# 安装docker-compose工具（用于容器编排）
sudo curl -L https://github.com/docker/compose/releases/download/1.24.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

```

## 镜像准备
Nginx：可直接从docker hub中拉取，执行`docker pull nginx`
Python：构建uWSGI容器可以基于centos、ubuntu等进行，但为了更轻量化，也可以使用python镜像，同样从docker hub中拉取所需python版本的镜像，例如：`docker pull python:3.6.8`

接下来，需要基于python镜像制作一个uWSGI服务器镜像。简单的方法是基于python镜像启动一个容器，然后进行python依赖包以及其他环境的安装，随后使用`docker commit`打包为镜像，但这样做有一些坏处：
在容器内安装一些依赖、环境时，会有大量无关内容被添加，并且由于镜像的分层存储，修改容器仅仅是在当前层进行标记、添加、修改，并不会改变上一层，因此使用`docker commit`制作镜像，会导致镜像越来越臃肿；其次，由于使用`docker commit`制作的镜像对于其他人来说是黑箱操作，后续进行维护的人无法得知镜像是如何构建的，增加了维护风险和难度。
因此，我们使用Dockerfile来构建uWSGI容器，关于Dockerfile的编写本文不再赘述，请参考：[https://yeasy.gitbooks.io/docker_practice/content/image/build.html](https://yeasy.gitbooks.io/docker_practice/content/image/build.html)，一个简单的Dockerfile如下：
```shell
# 基于python3.6.8镜像
FROM python:3.6.8

# 复制Django项目所需的依赖文件清单到容器中
COPY requirements.txt /

# 安装依赖
RUN pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple -r /requirements.txt

# 创建uwsgi软链接
RUN ln -s /usr/local/python3/bin/uwsgi /usr/bin/uwsgi
```

## 创建容器
镜像制作完成，就可以开始创建容器了，可以直接使用`docker run`命令来创建，也可以使用`docker-compose`工具，下面首先介绍`docker run`的方式：

Nginx: 
```shell
docker run -it --name nginx_container --privileged=true -p 443:443 -p 8080:8080 -v /DJANGO_PROJECT:/DJANGO_PROJECT -v /nginx.conf:/etc/nginx/nginx.conf --net staticnet --ip 192.168.0.2 --restart=always -d nginx
```

uWSGI：
```shell
docker run -it --name uwsgi_container --privileged=true -p 8090:8090 -v /DJANGO_PROJECT:/DJANGO_PROJECT -v /.virtualenvs/django_project/lib/python3.6/site-packages:/usr/local/lib/python3.6/site-packages --net staticnet --ip 192.168.0.3 --restart=always -d uwsgi_image uwsgi --ini /DJANGO_PROJECT/project/project/uwsgi.ini
```

参数解释：
`-it`：让Docker分配一个伪终端,并绑定到容器的标准输入上
`--name`：指定容器的名称
`--privileged`：让容器内的root用户拥有真正root权限
`-p`：指定端口映射，将宿主机的端口映射到容器端口
`-v`：指定文件挂载路径，将Django项目、nginx配置文件放在宿主机，然后挂载到容器
  说明：uWSGI容器启动时，将宿主机中一个虚拟环境（安装有Django项目所需的python依赖）的依赖包存放路径挂载到容器中python3依赖包存放路径，这样做的初衷是方便后续有新的依赖包需要安装时，可以直接在宿主机下安装，而不需要进入容器。但在实际应用中发现，某些依赖包使用这样方式安装后不会产生软链接，容器内无法使用，因此实际在制作uWSGI镜像时已经进行了依赖安装。后续将会尝试同时挂载python的site-packages目录和bin目录，也许可以解决这个问题。
`--net`：指定容器所在的网段（需要提前创建一个网段）
`--ip`：指定容器的ip。如果不特别指定，容器默认使用172.17.0.x的ip，并且会根据启动顺序变动
`--restart`：在容器退出时总是重启容器，保证容器始终运行
`-d`：让容器在后台运行

## 使用docker-compose工具
使用`docker run`命令可以创建并启动容器，但需要一个个容器地启动，这里推荐使用docker-compose工具来进行容器编排和启动。只需要编写一个`docker_compose.yaml`文件，然后通过`docker-compose up -d .`命令就可以一次性编排、启动多个容器。以下是docker_compose.yaml文件示例：

```shell
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
    
    uwsgi:
        container_name: uwsgi_container
        ports: 
            - 8090:8090
        volumes:
            - /DJANGO_PROJECT:/DJANGO_PROJECT
            - /.virtualenvs/django_project/lib/python3.6/site-packages:/usr/local/lib/python3.6/site-packages
        networks:
            extnetwork: 
                ipv4_address: 192.168.0.3
        privileged: true
        command: uwsgi --ini /DJANGO_PROJECT/project/project/uwsgi.ini
        restart: always        

networks: 
    extnetwork: 
        ipam: 
            config: 
            - subnet: 192.168.0.0/16



```

注意：Nginx容器的启动命令是`nginx -g 'daemon off;'`，这是因为容器启动后执行nginx启动命令开启的nginx进程是第一个进程(pid=1)，而Docker容器将pid=1的进程是否存在作为容器是否正在运行的依据，nginx默认以daemon方式运行，执行完启动命令就在后台运行，此时Docker判断pid=1的进程终止，容器就会退出（如果设置了restart=always参数，则容器会一直重启），因此需要加上`daemon off`参数。

## 总结
以上使用Docker技术部署一套Nginx + uWSGI + Django的web服务器，是使用Docker技术的一次尝试，主要是利用了Docker文件挂载的这一方式来实现，在实际使用过程中，也发现一些小问题需要优化，欢迎大家有更好的想法和我交流。
