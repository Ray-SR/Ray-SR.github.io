---
title: Git简介及常用操作
date: 2020-02-20 13:38:20
tags: [Git]
categories: Git
---

Git是一个开源的分布式版本控制系统，可以有效、高速地处理从很小到非常大的项目版本管理，本文介绍了Ｇｉｔ的一些基本概念及常用操作。

<!--more-->

## 1.Git基本概念
Git是一个分布式版本控制系统，可以有效、高速地处理从很小到非常大的项目版本管理。

- 工作区 (Workspace)

即我们进行代码/文件的增、删、改等操作的本机目录

- 暂存区 (Index/Stage)

用来暂存被改动文件的空间。在本地修改代码后，可以通过`git add`命令将变动保存到暂存区
暂存区的作用：` commit`操作是原子性的，我们可以挑选自己想记录提交的内容形成一次`commit`
     方便撤销修改`git checkout`
     对比工作区和暂存区的文件差异`git diff`

- 本地仓库 (Repository)

即项目目录下的`.git`文件夹，保存了所有和该项目有关的历史记录，`commit`的结果被保存在本地仓库中
`git init`命令可以初始化一个git本地仓库


- 远程仓库 (Remote)

用于项目协作的远程仓库，存放在远程主机上

![1.jpg](/img/Git/1.jpg)


## 2.常用操作

- **git init**

初始化一个git本地仓库，即生成`.git`文件夹

- **git clone**

从远程主机克隆版本库到本地，一般使用HTTP(s)协议，还支持SSH、Git、ftp等协议。
克隆时，所使用的远程主机自动被命名为`origin`,可以通过`-o`参数自定义命名，例如`git clone -o jQuery https://github.com/jquery/jquery.git`

- **git add**

在工作区修改完文件，可以通过`git add .`将所有改动提交到暂存区，也可以指定个别文件，如`git add a.txt`

- **git commit**

暂存区的文件需要经过`git commit -m 'commit info'`才会提交到本地仓库中，`commit`会生成一条版本记录

- **git push**

推送代码到远程仓库，例如`git push 远程主机名 本地分支名:远程分支名`
如果当前分支与远程分支存在追踪关系，则可以写成`git push origin`
如果当前分支只有一个追踪分支（只关联了一个远程主机），则可直接写`git push`
如果当前分支与多个远程主机存在追踪关系，则可使用`-u`参数指定默认主机，如`git push -u origin master`,之后`git push`便会默认推送到`origin`主机


注意，`git push`有`matching`模式和`simple`模式，matching模式会推送所有有对应的远程分支的本地分支，而simple模式只会推送当前分支。Git2.0版本前，默认采用`matching`模式，之后默认`simple`模式。可以通过`git config`修改：`git config --global push.default simple`


- **git pull**

拉取远程主机的某个分支，并与本地分支合并，例如`git pull 远程主机名 远程分支名:本地分支名`
如果需要拉取的远程分支是与当前分支合并，则可以写成`git pull origin master`,这一步相当于先执行`git fetch origin`,再执行`git merge origin/master`

- **git status**

查看文件状态


- **git log**

查看`commit`历史


- **git branch**

`git branch -a`查看本地与远程主机的所有分支
`git branch branch_name` 新建一个名为branch_name的分支，但仍然停留在当前分支
`git checkout -b branch_name` 新建一个名为branch_name的分支，并切换到该分支
`git branch --set-upstream local_branch remote_branch` 将本地分支local_branch与远程分支remote_branch建立追踪关系
`git branch -d branch_name` 删除本地分支
`git push origin --delete remote_branch` 删除远程分支


- **git checkout**

拉取远程主机的分支后，可以在它的基础上新建分支，如`git checkout -b new_branch`
使用`git checout branch_name`可以在本地将工作区切换到另一个分支
也可用于切换到某次commit : `git checkout commit_id`

- **git remote**

为了便于管理，每个远程主机都必须指定一个主机名，`git remote -v`可以列出所有远程主机名及其地址
`git remote add 主机名 项目地址` 用于添加远程主机
`git remote rm 主机名` 用于删除远程主机
`git remote rename 原主机名 新主机名` 可以重命名远程主机

- **git reset**

`git reset`可以重置当前分支到指定`commit`,有3种模式：`mixed`、`soft`、`hard`
`git reset --mixed commit_id`：重置当前分支的指针为指定commit，同时重置暂存区，但工作区不变
`git reset --soft commit_id`：重置当前分支的指针为指定commit，但保持暂存区和工作区不变
`git reset --hard commit_id`：重置当前分支的指针为指定commit，同时重置暂存区和工作区，与指定commit一致(这种方式会丢失本地文件的修改)

![2.png](/img/Git/2.png)

- **git tag**

在一个版本完成之后，可以利用git tag打上标签,方便管理与回溯，其作用与commit_id(commit-sha1)类似.
`git tag` 查看所有标签
`git tag -a tag_name -m "tag description"` 给当前分支打上名称为tag_name的标签，描述为"tag description"
`git push origin tag_name` 推送指定tag到远程
`git push origin --tags` 推送所有tag到远程
`git tag -d tag_name` 删除指定tag


## 3.其他

- **gitignore**

有时项目中有些文件不需要放入远程仓库，可以创建一个`.gitignore`文件来配置推送时需要忽略的文件，例如：
*.py[cod]   忽略所有以.pyc、.pyo、.pyd结尾的文件
.idea/        忽略.idea文件夹

- **配置公钥**

如果不想每次推送代码都要使用用户名、密码进行认证，可以通过ssh公钥的方式认证，以下是Gitlab配置公钥的示例
-- 生成公钥
`ssh-keygen -t rsa -C "YOUR EMAIL"`
-- 查看公钥并复制
`cat ~/.ssh/id_rsa.pub`
-- 将公钥添加到Gitlab
Settings >> SSH Keys

![3.png](/img/Git/3.png)

## 4.参考链接
Git原理入门： [http://www.ruanyifeng.com/blog/2018/10/git-internals.html](http://www.ruanyifeng.com/blog/2018/10/git-internals.html)
Git常用命令清单：[https://www.ruanyifeng.com/blog/2015/12/git-cheat-sheet.html](https://www.ruanyifeng.com/blog/2015/12/git-cheat-sheet.html)
Git reset三种模式：[https://www.jianshu.com/p/c2ec5f06cf1a](https://www.jianshu.com/p/c2ec5f06cf1a)
