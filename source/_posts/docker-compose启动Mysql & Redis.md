---
title: docker-compose启动Mysql和Redis
date: 2020-12-21 22:38:49
tags: [Docker]
categories: Docker
---

本文介绍了如何使用Docker快速在本地部署一套Mysql + Redis

<!--more-->

### 1.创建文件夹、下载Redis配置文件
`mkdir -p /docker/mysql /docker/redis`
`cd /docker/redis`
`wget http://download.redis.io/redis-stable/redis.conf`


### 2.修改Redis配置文件
`vim redis.conf`
修改以下配置以允许redis远程连接：
```
protected-mode no

# bind 127.0.0.1
```
### 3.下载docker-compose
`curl -L https://get.daocloud.io/docker/compose/releases/download/1.24.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose`
`sudo chmod +x /usr/local/bin/docker-compose`


### 4.编写docker-compose.yaml
```yaml
version: "3"
services:
  mysql:
    # 指定镜像
    image: mysql:5.7

    # 指定容器名称
    container_name: mysql_container

    environment:
      - MYSQL_ROOT_PASSWORD=123456

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
        ipv4_address: 192.168.0.2

    privileged: true
    restart: always

  redis:
    image: redis

    container_name: redis_container

    ports:
      - 6380:6379

    volumes:
      - /docker/redis/data:/data
      - /docker/redis/redis.conf:/etc/redis/redis.conf

    command: redis-server /etc/redis/redis.conf

    networks:
      extnetwork:
        ipv4_address: 192.168.0.3

    privileged: true
    restart: always

networks:
  extnetwork:
    ipam:
      config:
        - subnet: 192.168.0.0/16
```
### 5.启动
`docker-compose -f docker-compose.yaml up -d`

