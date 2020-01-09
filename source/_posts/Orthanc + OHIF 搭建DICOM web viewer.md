---
title: Orthanc + OHIF 搭建DICOM web viewer
date: 2019-12-08 22:38:01
tags: [Orthanc,OHIF,DICOM]
categories: DICOM
---

常规的DICOM viewer一般是桌面软件，如果需要通过浏览器查看DICOM影像，可以借助一些开源软件。本文将介绍如何利用开源DICOM server `Orhtanc`以及DICOM web viewer `OHIF viewer`搭建一套DICOM阅片系统。

<!--more-->

## Orthanc
Orthanc是一个轻量级的、模块化的DICOM服务器，除了实现DICOM协议、WADO协议，它还提供了REST API以及丰富的插件。由于它提供了便捷的REST API，orthanc可以使用任何语言开发，它存储的DICOM图像标签可以通过JSON格式下载，并且orthanc对于存储的DICOM实例可以动态生成对应的PNG图像。

关于Orthanc的使用可以参考其官方文档：
[https://book.orthanc-server.com/index.html](https://book.orthanc-server.com/index.html)

Orthanc可以运行在windows以及Linux平台下，可以使用其二进制包安装或者编译源码，同时它也提供了Docker镜像，以下介绍通过Docker的方式运行一个Orthanc服务：

- 查看镜像

`docker search orthanc`

![lgt2jJ.png](https://s2.ax1x.com/2020/01/08/lgt2jJ.png)
- 拉取镜像(注意，只有orthanc-plugins才提供REST API)

`docker pull jodogne/orthanc-plugins`

- 启动容器

`mkdir /tmp/orthanc-db` (创建文件夹用于存放orthanc数据,即DICOM数据)
`sudo docker run --name orthanc -p 4242:4242 -p 8042:8042 --restart=always -v /tmp/orthanc-db/:/var/lib/orthanc/db/ jodogne/orthanc-plugins`

- 网页查看

`http://orthanc所在的服务器ip:8042`
默认账号：  orthanc
默认密码：  orthanc
可以通过Upload按钮进行文件上传页面，上传DICOM影像

至此orthanc就安装完毕，DICOM viewer（例如OHIF）就可以通过REST API调用，具体API参考：[https://book.orthanc-server.com/users/rest.html](https://book.orthanc-server.com/users/rest.html)

在前面创建容器时挂载的目录`/var/lib/orthanc/db`即orthanc中DICOM文件存放的路径，对应宿主机即`/tmp/orthanc-db/`.Orthanc会将DICOM原始文件、DICOM tag分别作为2个文件存放在这个路径下
![lgafBj.png](https://s2.ax1x.com/2020/01/08/lgafBj.png)

## OHIF Viewer
OHIF Viewer是一套基于Cornerstone（一套JavaScript底层组件，用于支持医学影像的显示与交互）开发的纯网页版医学影像浏览前端。其github地址为：[https://github.com/OHIF/Viewers](https://github.com/OHIF/Viewers)

启动项目非常简单，只需要clone下来然后在项目根目录执行`yarn install`、`yarn run dev`即可（官方文档[https://docs.ohif.org/](https://docs.ohif.org/)），不过它默认是连接OHIF提供的一个远程DICOM服务，我们需要修改配置，让它连接到Orthanc。
打开`platform/viewer/package.json`修改proxy的值为之前启动的Orthanc服务地址，例如`"proxy": "http://192.168.0.23:8042"`，然后执行`yarn run dev:orthanc`即可对接Orthanc,初次查看影像需要输入orthanc的账号密码（默认）。
