---
title: Gitlab CI/CD + Docker部署
date: 2020-12-27 21:31:01
tags: [CI/CD,Docker]
categories: CI/CD
---

利用Gitlab的CI/CD功能可以很方便地进行项目的持续集成和部署，我们还可以配合Docker来进一步完善服务器部署。本文介绍了如何使用docker-swarm这个工具来部署一套Django + Mysql + Redis的项目，并且配合Gitlab的CI、CD功能，实现自动部署、滚动更新。

<!--more-->

## 1.Gitlab CI/CD安装及配置
- ubuntu下安装：

[https://www.yuque.com/docs/share/cf815454-54d0-49a0-9921-90564bd869e8?#](https://www.yuque.com/docs/share/cf815454-54d0-49a0-9921-90564bd869e8?#) 

CentOS安装步骤也类似，不过使用yum安装gitlab-runner可能版本太低，需要更新

- `curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | sudo bash`
- `sudo yum -y update`



并且Git的版本也需要更新

- `yum install http://opensource.wandisco.com/centos/7/git/x86_64/wandisco-git-release-7-2.noarch.rpm`
- `yum update -y git`



## 2.Docker swarm
#### docker-swarm介绍

- [https://yeasy.gitbook.io/docker_practice/swarm_mode/overview](https://yeasy.gitbook.io/docker_practice/swarm_mode/overview)
#### 安装
`docker swarm init` 启用swarm，将当前主机作为管理节点（管理节点自己也是worker）
如果要让其他节点加入swarm集群，运行init之后提示的`join`命令即可：
![1234.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1603676764667-ed921106-633a-4d96-98d0-6c14ab367e05.png#align=left&display=inline&height=196&margin=%5Bobject%20Object%5D&name=1234.png&originHeight=196&originWidth=1524&size=21322&status=done&style=none&width=1524)
#### 基础命令

- `docker node ls` 查看所有节点
- `docker service ls` 查看所有服务`
`
- `docker service create -p 9000:9000 --mount type=bind,src="<project_path>",dst="<container_path>" --replicas 3 --name <service_name> <image>:<tag>`

`-p`指定端口映射，--mount指定目录挂载，`--replicas`指定任务(task)个数，即容器数量

- `docker service update --image <new_image_name> <service_name>` 更新服务
#### docker stack

- 可以使用docker stack同时启动多个服务(service)，原理同`docker-compose`类似
- `docker stack deploy -c docker-compose.yaml <stack_name>`
- `docker-compose.yaml`示例：
```yaml
version: "3"
services:
  mysql:
    # 指定镜像
    image: mysql:5.7
    environment:
      - MYSQL_ROOT_PASSWORD=123456
      - MYSQL_DATABASE=go2home
    volumes:
      - /docker/mysql/conf:/etc/mysql
      - /docker/mysql/logs:/var/log/mysql
      - /docker/mysql/data:/var/lib/mysql
    # 端口映射
    ports:
      - 3307:3306
    # 网络和ip
    networks:
      extnetwork:
  redis:
    image: redis
    ports:
      - 6380:6379
    volumes:
      - /docker/redis/data:/data
      - /docker/redis/redis.conf:/etc/redis/redis.conf
    command: redis-server /etc/redis/redis.conf
    networks:
      extnetwork:
  go2home:
    image: go2home:latest
    volumes:
      - /docker/go2home/logs:/go2home/logs
    ports:
      - 9000:9000
    networks:
      extnetwork:
    deploy:
      mode: replicated
      replicas: 3
networks:
  extnetwork:
    ipam:
      config:
        - subnet: 192.168.0.0/16
```

- 注意：使用这种方式部署之后，在项目内就可以通过`<服务名>:端口` 的方式来访问mysql/redis了。例如使用`docker stack deploy -c docker-compose.yaml my_stack` 创建了服务之后，在`go2home`服务内就可以通过`my_stack_mysql:3306` 来访问mysql容器。这是因为docker swarm在创建服务的时候配置了虚拟ip以及DNS。

关于docker swarm的网络问题，参考：[https://www.jianshu.com/p/cacdd5ff0f14](https://www.jianshu.com/p/cacdd5ff0f14)


## 3. 项目配置

- 在项目根目录下新建文件`.gitlab-ci.yml`,配置如下：
```yaml
stages:
  - build
variables:
  DEPLOY_PATH: "/root/go2home/"               # 项目的部署路径
build-job:
  stage: build
  script:
    - OLD_IMAGE_ID=$(docker images | grep go2home | awk 'NR==1{print$3}')  # 获取最近一次构建的镜像ID
    - echo $OLD_IMAGE_ID
    - echo $CI_COMMIT_SHA
    - sudo cp -r $CI_PROJECT_DIR/* $DEPLOY_PATH                            # 将代码拷贝到部署路径
    - cd $DEPLOY_PATH
    - NEW_IMAGE_NAME=go2home:$CI_COMMIT_SHA
    - echo $NEW_IMAGE_NAME
    - docker build -t $NEW_IMAGE_NAME .                                    # 以commit ID为tag，构建新镜像
    - docker service update --image $NEW_IMAGE_NAME go2home_go2home        # 更新服务
    - docker rmi -f $OLD_IMAGE_ID                                             # 删除旧镜像
  tags:
    - new_go2home
  only:
    - master
  environment:
    name: prod
```

- 在项目根目录下新建`Dockerfile`文件以及`.dockerignore`文件(以Django项目为例)

Dockerfile
```dockerfile
# 基于python3.7.3镜像
FROM python:3.7.3

# 设置系统环境变量，用于在项目中区分环境
ENV DJANGO_ENV DOCKER

# 该设置确保python的标准输出实时
ENV PYTHONUNBUFFERED 1

# 设置工作目录
WORKDIR /go2home

# 单独将requirements.txt作为一层，这样当requirements.txt没有变化时,build镜像会使用之前的缓存
ADD ./requirements.txt /go2home/requirements.txt
RUN pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple

# 添加项目文件
ADD . /go2home

# 暴露端口
EXPOSE 9000

# 设置启动命令
ENTRYPOINT python GoToHome/manage.py migrate && \
           uwsgi --ini GoToHome/docker-uwsgi.ini
```
.dockerignore
```
*.log
celerybeat-schedule
uwsgi.pid
```
此后的部署，只需要push代码到gitlab仓库，就会自动执行.gitlab-ci.yml文件中配置的步骤，更新镜像、服务
## 4.Docker portainer安装
portainer是一个docker的图形化管理工具，开箱即用，用来管理docker服务很方便

- `docker pull portainer/portainer`
- `docker run -d -p 8080:9000 --restart=always --name portainer -v /var/run/docker.sock:/var/run/docker.sock -v /docker/portainer:/data portainer/portainer`
- 创建容器后，访问`http://<服务器ip>:8080` 即可，初次登陆设置管理员密码





