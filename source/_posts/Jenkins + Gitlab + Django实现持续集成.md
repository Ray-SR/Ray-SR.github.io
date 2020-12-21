---
title: Jenkins + Gitlab + Django实现持续集成
date: 2020-12-26 19:54:40
tags: [CI/CD,Jenkins]
categories: CI/CD
---

基于ubuntu18.04搭建一套持续集成、部署的系统，在本地推送(push)代码到Gitlab后，通过Gitlab的webhook触发Jenkins任务，自动拉取代码并部署到生产服务器运行。


### 1. Jenkins搭建

- **查看是否安装了java**

`java -version`

- **如果没有安装java,则执行下面的命令安装**

`sudo apt-get install openjdk-8-jre`

- **配置java环境变量，在`/etc/profile`中添加以下内容：**
```bash
#set jdk environment 
export JAVA_HOME=/usr/lib/jvm/Java-8-openjdk-amd64 
export JRE_HOME=$JAVA_HOME/jre 
export CLASSPATH=$JAVA_HOME/lib:$JRE_HOME/lib:$CLASSPATH 
export PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH
```

- **使配置文件生效**

`source /etc/profile`


- **安装Jenkins**
- `wget -q -O - https://pkg.jenkins.io/debian/jenkins-ci.org.key | sudo apt-key add - `
- `sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list' `
- `sudo apt-get update`
- `sudo apt-get install jenkins`



- **运行jenkins**, 访问 [http://localhost:8080/](http://localhost:8080/)

`java -jar /usr/share/jenkins/jenkins.war`
由于8080端口可能被占用，建议改端口:
`vim /etc/default/jenkins` ,找到`HTTP_PORT=8080` 这一项，将其改为其他端口
然后重启`sudo systemctl restart jenkins`


参考：[https://blog.csdn.net/lsl520hah/article/details/102839940](https://blog.csdn.net/lsl520hah/article/details/102839940)


### 2. 生产服务器准备(ubuntu18.04)

- 将Django项目文件夹`mipin`拷贝到生产服务器`/home/ubuntu`路径下
#### 2.1 python安装

- python3.7安装参考：[https://zhuanlan.zhihu.com/p/62930419](https://zhuanlan.zhihu.com/p/62930419)
- python安装完成后，开始安装虚拟环境`virtualenv`

`sudo pip3.7 install virtualenv`
`sudo pip3.7 install virtualenvwrapper`

- 配置环境变量`vim ~/.bashrc` ,加入以下内容
```bash
export VIRTUALENVWRAPPER_PYTHON=/usr/local/bin/python3.7
export WORKON_HOME=/home/ubuntu/.virtualenvs
export VIRTUALENVWRAPPER_VIRTUALENV=/usr/local/bin/virtualenv
source /usr/local/bin/virtualenvwrapper.sh
```

- `source ~/.bashrc`
- 创建名为`mipin`的虚拟环境（作为Django项目运行的虚拟环境）

`mkvirtualenv mipin`
`pip install -r /home/ubuntu/mipin/requirements.txt`
#### 2.2 部署Django项目   
这里使用uwsgi来运行Django项目，并通过supervisor来管理进程
详见：[https://www.yuque.com/docs/share/2ccfaa59-827c-4549-9a6e-1c214c02c2b6?#](https://www.yuque.com/docs/share/2ccfaa59-827c-4549-9a6e-1c214c02c2b6?#) 《Supervisor使用》
### 3. Jenkins及Gitlab配置
#### 3.1 安装jenkins插件

- 需要安装的插件有`Gitlab Hook`、 `Build Authorization Token Root` 、`Publish Over SSH`、 `Gitlab Authentication`、 `Gitlab` 、 `Git Parameter`

![1.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1583828951202-5ad3ebf3-ca8c-4094-b89d-6d8561804df0.png?x-oss-process=image%2Fresize%2Cw_746)

![2.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1583828973801-7484a035-641b-4ae2-9e4e-49073f690bd8.png?x-oss-process=image%2Fresize%2Cw_746)


![3.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1583828993629-bdcb0ae6-583c-4f6c-a4f5-1f64ce770d46.png#align=left&display=inline&height=738&margin=%5Bobject%20Object%5D&name=3.png&originHeight=847&originWidth=856&size=62473&status=done&style=stroke&width=746)


- 安装完插件，重启jenkins



#### 3.2 配置SSH密钥

- 首先在jenkins所在的主机上生成一对SSH密钥

`ssh-keygen -t rsa -P ''`
执行这条命令，会在`~/.ssh`目录下创建2个文件: `id_rsa` (私钥) 、`id_rsa.pub` (公钥)

- 然后将公钥发送到需要部署代码的主机

`ssh-copy-id -i ~/.ssh/id_rsa.pub <IP>` <IP>即需要部署代码的目标主机的ip
之后在jenkins所在主机上直接使用`ssh <IP>` 看是否能免密码登录到目标主机
若还有问题请参考：[https://www.cnblogs.com/jager/p/5986563.html](https://www.cnblogs.com/jager/p/5986563.html)

- 将私钥以及SSH主机配置到jenkins系统设置中

打开系统设置
![4.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1583829985077-c86998ae-ad86-42dc-ac6b-488fe094e3c2.png#align=left&display=inline&height=563&margin=%5Bobject%20Object%5D&name=4.png&originHeight=735&originWidth=974&size=75785&status=done&style=stroke&width=746)
复制私钥内容填入
![5.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1583830248101-93403417-4ee7-4cc1-af1a-02f08a915d7e.png#align=left&display=inline&height=159&margin=%5Bobject%20Object%5D&name=5.png&originHeight=333&originWidth=1559&size=35133&status=done&style=stroke&width=746)
添加目标服务器（点击Add）
![6.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1583830601295-27cab8d0-193b-4193-a3b0-d89c73ff80ed.png#align=left&display=inline&height=212&margin=%5Bobject%20Object%5D&name=6.png&originHeight=343&originWidth=1205&size=23845&status=done&style=stroke&width=746)
（如果需要配置多台服务器，同样按上面方法将jenkins所在主机的公钥发到目标服务器，再到这里点击Add添加即可）
点击save保存
#### 3.3 新建jenkins项目

- 新建项目

![7.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1583830970033-db9c5bae-6a33-415b-9830-ddc5cd164b86.png#align=left&display=inline&height=574&margin=%5Bobject%20Object%5D&name=7.png&originHeight=574&originWidth=381&size=22969&status=done&style=stroke&width=381)


![8.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1583831000929-b495b4b5-d5de-438c-be80-ba99854e8194.png#align=left&display=inline&height=424&margin=%5Bobject%20Object%5D&name=8.png&originHeight=739&originWidth=1300&size=81736&status=done&style=stroke&width=746)


- 配置项目

点击Add添加Git认证密钥
![9.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1583832896451-4e9bcb15-9972-4ce7-ba92-3703b27c2186.png#align=left&display=inline&height=545&margin=%5Bobject%20Object%5D&name=9.png&originHeight=545&originWidth=1462&size=34995&status=done&style=none&width=1462)
复制自己开发电脑的私钥到这里（前提是已经在Gitlab上添加了自己的公钥）
![10.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1583833269657-a234b492-5cc2-426f-a48c-9847d533e022.png#align=left&display=inline&height=360&margin=%5Bobject%20Object%5D&name=10.png&originHeight=690&originWidth=1431&size=48284&status=done&style=stroke&width=746)


- 如何在Gitlab上添加公钥

点击Gitlab右上角个人中心的Settings进入设置页面
![11.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1583833629109-956ca706-a023-4b46-a14b-bb2ba49b7806.png#align=left&display=inline&height=378&margin=%5Bobject%20Object%5D&name=11.png&originHeight=796&originWidth=1571&size=59650&status=done&style=stroke&width=746)

- 配置build triggers（记录url以及secret token,用于后面Gitlab中webhook的配置）

![12.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1583834522906-50d40cce-81c3-4253-b007-d656079d956a.png#align=left&display=inline&height=465&margin=%5Bobject%20Object%5D&name=12.png&originHeight=627&originWidth=1005&size=40222&status=done&style=stroke&width=746)
点击Advanced
![13.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1583834550396-889bcc55-30d8-4f7d-8365-46f2ee5a763d.png#align=left&display=inline&height=290&margin=%5Bobject%20Object%5D&name=13.png&originHeight=392&originWidth=1010&size=25759&status=done&style=stroke&width=746)

- 配置部署参数

在部署的时候，有时可能需要配置一些可变参数，这些参数能够在后续执行部署脚本（Exec command）的时候获取到
![微信截图_20201221100828.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1608516530013-3035fc36-905b-4148-9c05-8691021dcd8e.png#align=left&display=inline&height=875&margin=%5Bobject%20Object%5D&name=%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20201221100828.png&originHeight=875&originWidth=1064&size=43899&status=done&style=none&width=1064)

- 配置构建步骤

这里同样可以配置多个服务器，点击Add server添加即可
另外，建议将Advanced中Exec timeout(命令执行的超时时间)调整大一点  ， 按照下图配置


![14.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1583834823283-e0ab6c91-6d17-4a1b-ae02-d90a43f2ba04.png#align=left&display=inline&height=381&margin=%5Bobject%20Object%5D&name=14.png&originHeight=736&originWidth=1441&size=63525&status=done&style=stroke&width=746)
Exec command一栏即代码传输到服务器之后需要执行的构建步骤，上图是一个简单的示例。实际生产中，我们是使用supervisor来管理进程，并且还需要进行代码备份，以方便后续回滚，一个完整的脚本如下：
```shell
case $Status  in
  Deploy)
    echo "Status:$Status"
    bak_path="/home/ubuntu/bak/mipin/${BUILD_NUMBER}"      #创建每次要备份的目录
    if [ -d $bak_path ];
    then
        echo "The files is already  exists "
    else
        sudo mkdir -p $bak_path
    fi
    \sudo cp -rf /home/ubuntu/mipin/* $bak_path        #将项目备份到相应目录,覆盖已存在的目标
    sudo /home/ubuntu/.virtualenvs/mipin/bin/pip install -r /home/ubuntu/mipin/requirements.txt
    sudo supervisorctl restart mipin
    echo "Completing!"
    ;;
  Rollback)
      echo "Status:$Status"
      echo "Version:$Version"
      cd /home/ubuntu/bak/mipin/$Version            #进入备份目录
      \sudo cp -rf ./* /home/ubuntu/mipin/       #将备份拷贝到程序打包目录中
      sudo supervisorctl restart mipin
      ;;
  *)
  exit
      ;;
esac

# 删除旧的代码
ReservedNum=5  #保留文件数
FileDir=/home/ubuntu/bak/mipin
date=$(date "+%Y%m%d-%H%M%S")

cd $FileDir   #进入备份目录
FileNum=$(ls -l | grep '^d' | wc -l)   #当前有几个文件夹，即几个备份

while(( $FileNum > $ReservedNum))
do
    OldFile=$(ls -rt | head -1)         #获取最旧的那个备份文件夹
    echo  $date "Delete File:"$OldFile
    sudo rm -rf $FileDir/$OldFile
    let "FileNum--"
done 
```


#### 3.4 Gitlab配置webhook
![15.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1583835155725-e1fd4754-6815-4616-af55-702f4beb2447.png#align=left&display=inline&height=424&margin=%5Bobject%20Object%5D&name=15.png&originHeight=895&originWidth=1574&size=77341&status=done&style=stroke&width=746)
测试
![16.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1583835253463-46417bb9-70b7-4870-9b42-c8d5414615e7.png#align=left&display=inline&height=424&margin=%5Bobject%20Object%5D&name=16.png&originHeight=473&originWidth=832&size=32858&status=done&style=stroke&width=746)


#### 3.5 如何查看部署结果
做完以上步骤，可以尝试在本地push代码到Gitlab，然后登陆Jenkins查看部署结果
![17.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1583835612136-db25d301-2a62-4cc3-8162-31f48d14d8ec.png#align=left&display=inline&height=721&margin=%5Bobject%20Object%5D&name=17.png&originHeight=778&originWidth=805&size=63084&status=done&style=stroke&width=746)
点击console output查看控制台输出
![18.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1583835595912-9860a832-5d31-4db3-8d17-87dd04c3ecaa.png#align=left&display=inline&height=426&margin=%5Bobject%20Object%5D&name=18.png&originHeight=875&originWidth=1534&size=94438&status=done&style=stroke&width=746)


#### 3.6 回滚
由于在配置时加入了构建参数，因此我们可以在手动构建的时候根据情况选择
![jenkins222.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1608519958808-ebb8f773-de75-4975-96c4-5fb408c71ee4.png#align=left&display=inline&height=564&margin=%5Bobject%20Object%5D&name=jenkins222.png&originHeight=564&originWidth=815&size=46029&status=done&style=stroke&width=815)
### 4. 参考文章
Ubuntu18.04安装Jenkins ： [https://blog.csdn.net/lsl520hah/article/details/102839940](https://blog.csdn.net/lsl520hah/article/details/102839940)
Gitlab + Jenkins实现自动部署：[https://blog.51cto.com/bigboss/2129477](https://blog.51cto.com/bigboss/2129477)
