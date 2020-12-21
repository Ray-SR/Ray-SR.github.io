---
title: Gitlab CI/CD配置
date: 2020-12-27 20:30:00
tags: [CI/CD,Git]
categories: CI/CD
---

Gitlab有自带的CI/CD功能，只需要在项目根目录新建一个`.gitlab-ci.yml`文件，推送到仓库中，即可创建一个pipeline，并通过注册好的runner部署、运行项目

<!--more-->

## 1.注册gitlab-runner
gitlab-runner相当于gitlab代码仓库和服务器之间的中间人，当我们触发了pipelines中的任务，它就可以将代码仓库中的代码拉到服务器，执行我们配置好的脚本，从而实现自动部署


注意：为保持前后一致，所有命令都要以sudo权限运行


- 安装

`curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash`


`sudo apt-get install gitlab-ci-multi-runner`


- 注册

`sudo gitlab-runner register`


①URL以及token，去gitlab项目中的setting中复制，如下图：
![1.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1594975213918-21a88996-f654-49f4-98b0-5a0353f6505e.png#align=left&display=inline&height=829&margin=%5Bobject%20Object%5D&name=1.png&originHeight=829&originWidth=1591&size=83750&status=done&style=stroke&width=1591)
![2.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1594975228369-c31bcdd7-0fcc-4dd8-b184-d9311fcedc50.png#align=left&display=inline&height=852&margin=%5Bobject%20Object%5D&name=2.png&originHeight=852&originWidth=957&size=76152&status=done&style=stroke&width=957)
②tag
tag是作为这个gitlab-runner的标识，此后在`.gitlab-ci.yml`文件中配置的时候需要用到
③是否运行在没有tag的build上面
选`true`
④选择执行器
选`shell`


## 2.启动gitlab-runner

- install

`sudo gitlab-runner install -n "service_name" -d /home/ubuntu -u ubuntu`
-n: 服务名称
-u: 以哪个用户的身份执行部署脚本中的shell命令
-d: 工作目录（如果不指定，则默认是执行install命令时所在的路径）
![1111111.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1597821172707-704fcac2-f70a-418b-a37d-e81bbef039c6.png#align=left&display=inline&height=213&margin=%5Bobject%20Object%5D&name=1111111.png&originHeight=213&originWidth=1361&size=20018&status=done&style=none&width=1361)

- start

`sudo gitlab-runner start -n "service_name"`
这里的service_name即install时指定的名称
启动成功之后就可以在gitlab settings --> CI/CD --> Runners 下查看新增的runner,显示绿色则表示启动成功
![1597822667(1).jpg](https://cdn.nlark.com/yuque/0/2020/jpeg/480333/1597822695447-c21ffb03-172a-42e7-860d-260112b29ea6.jpeg#align=left&display=inline&height=810&margin=%5Bobject%20Object%5D&name=1597822667%281%29.jpg&originHeight=810&originWidth=1606&size=94738&status=done&style=stroke&width=1606)
## 3.配置`.gitlab-ci.yml`
关于gitlab-ci.yaml中各项配置的含义，详见：`https://juejin.im/post/6844904045581172744`

- 在项目根目录下新建`.gitlab-ci.yml`文件，配置构建时需要执行的一些列操作。以下是Django项目部署的一个示例：
```yaml
stages:
  - build
  - test
variables:
  DEPLOY_PATH: "/home/ubuntu/deploy/car_wash/"               # 项目的部署路径
  PYTHON_PATH: "/home/ubuntu/.pyenv/versions/car_wash/bin"   # python可执行文件(bin)的路径
build-job:
  stage: build
  script:
    - sudo cp -r $CI_PROJECT_DIR/* $DEPLOY_PATH              # 将gitlab-runner拉下来的代码复制到部署路径
    - cd $DEPLOY_PATH
    - sudo $PYTHON_PATH/pip install -r requirements.txt      # 安装依赖
    - $PYTHON_PATH/python CarWash/manage.py migrate          # 执行迁移
    - sudo supervisorctl restart car_wash                    # 使用supervisor重启服务
    - sudo supervisorctl restart celery
  tags:
    - CarWash
  only:
    - master
  environment:
    name: prod
test-job:
  stage: test
  script:
    - cd $DEPLOY_PATH
    - $PYTHON_PATH/python CarWash/manage.py test Common --keepdb  # 执行测试
  tags:
    - CarWash
  only:
    - master
  environment:
    name: prod
```

- stage

定义一次pipeline需要执行的构建步骤，比如部署代码--->执行测试--->部署到生产环境；
每个stage可以包含多个job,同一个stage下面的job并行执行；


- variables

变量，可以自定义一些下面job中需要用到的变量，如项目路径等；
gitlab也有一些系统预设的变量，比如`$CI_PROJECT_DIR` 是gitlab-runner将代码从仓库中拉下来之后存放的路径；
对于一些不方便写到配置文件中的私密变量（如secrect key、SSH key等），可以在gitlab的 settings-->CI/CD-->variables 中配置；


- job

具体执行的构建动作，每个job必须指定对应的stage


- script

执行的命令


- tags

指定以哪个gitlab-runner来执行该job，这里需要和注册gitlab-runner时指定的tag对应


- only

限定git分支


- environment

用于定义一个job部署到的环境，定义了之后会自动创建一个环境，在gitlab的Operations --> Environments 中可以看到该环境下执行的job，并进行重新部署、回滚等操作
![微信截图_20200904162536.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1599207997711-4f309391-4c1b-4abf-870e-852ab8839db2.png#align=left&display=inline&height=646&margin=%5Bobject%20Object%5D&name=%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20200904162536.png&originHeight=646&originWidth=1730&size=45518&status=done&style=stroke&width=1730)
![微信截图_20200904162612.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1599208015050-b25b578c-5f50-43d6-8d44-8d668f7d7856.png#align=left&display=inline&height=651&margin=%5Bobject%20Object%5D&name=%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20200904162612.png&originHeight=651&originWidth=1793&size=43336&status=done&style=stroke&width=1793)


- 将`.gitlab-ci.yml` 文件推到(push)代码仓库之后，就可以发现pipeline已经创建，点击进入查看详细日志
- 状态为passed则表示部署成功

![222222222.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1597822807524-fa71b867-e5c2-42eb-b49f-6b00edbe4a41.png#align=left&display=inline&height=779&margin=%5Bobject%20Object%5D&name=222222222.png&originHeight=779&originWidth=1782&size=51915&status=done&style=stroke&width=1782)

- Vue项目部署配置示例
```yaml
stages:
  - build
variables:
  DEPLOY_PATH: "/home/ubuntu/deploy/car_wash_admin/"               # 项目的部署路径
build-job:
  stage: build
  script:
    - cd $CI_PROJECT_DIR                                           # 进入代码目录
    - rm -rf $CI_PROJECT_DIR/node_modules
    - npm install                                                  # 安装依赖
    - npm run build                                                # 打包构建
    - cp -r $CI_PROJECT_DIR/dist/* $DEPLOY_PATH                    # 将打包完成的文件复制到部署路径
    - sudo nginx -s reload                                         # 重启Nginx
  tags:
    - car_wash_admin_tag                                           # gitlab-runner tag
  only:
    - master                                                       # 允许部署的分支
  environment:
    name: prod                                                     # 环境名称
```
## 3.参考链接
[https://www.jianshu.com/p/306cf4c6789a](https://www.jianshu.com/p/306cf4c6789a)
[https://www.jianshu.com/p/2b43151fb92e](https://www.jianshu.com/p/2b43151fb92e)
