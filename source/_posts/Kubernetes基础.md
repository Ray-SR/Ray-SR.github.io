---
title: Kubernetes基础
date: 2020-12-27 21:59:01
tags: [Docker,kubernetes]
categories: kubernetes
---

**k8s 把数量众多的服务器重新抽象为一个统一的资源池**，对于运维人员来说，他们面前没有服务器1、服务器2的概念，而是一个统一的资源池，增加新的服务器对运维人员来说，只是增加自资源池的可用量。不仅如此，k8s 把所有能用的东西都抽象成了资源的概念，从而提供了一套更统一，更简洁的管理方式。

<!--more-->

## 1.基本概念
### 1.1 Pod
Pod是K8S中的最小工作单位， 和Docker中的容器类似， 不过pod是将一个或多个docker容器封装成一个统一的整体进行管理，并对外提供服务，同一个Pod内的容器可以共享网络栈、存储卷等。


例如查看本机有哪些pod:  `kubectl get pods`
查看命名空间为kube-system下的所有pod: `kubectl get pods -n kube-system`


- **创建资源**

k8s中所有资源都可以通过`kube create`命令创建，使用create创建资源有2种方式，一种是直接使用命令指定参数，另一种是通过`yaml`文件创建。例如新建一个名为`kubia-pod.yaml`的文件如下：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual
spec:
  containers:
  - image: luksa/kubia
    name: kubia
    ports:
    - containerPort: 8080
      protocol: TCP
```
然后使用`kubectl create -f kubia-pod.yaml` 就可以创建一个pod
之后如果修改了配置，可以使用`kubectl apply -f kubia-pod.yaml`来重新应用配置
### 1.2 namespace
命名空间namespace是k8s中组的概念，提供同一服务的pod应该被放置在同一命名空间下， 如果不指定，pod默认被放在命名空间`default` 下


### 1.3 ReplicationController（副本控制器）
ReplicationController(RC)是pod的复制抽象，用于解决pod的扩容缩容问题, 它会确保任何时间Kubernetes中都有指定数量的Pod在运行。在此基础上，RC还提供了一些更高级的特性，比如滚动升级、升级回滚等。
RC通过label来关联pod， 对于pod，需要设置其自身的label进行表示，label是一些列的key/value对。在创建RC的时候，需要指定**标签选择器**(Label Selector)，生成之后，它就会通过选择器查找pod并将其纳入自己的管辖

- **标签：标签是k8s中用于分类资源而提供的一个属性，一个标签包含`标签名`和`标签值`， 例如`app=kubia`, k8s中绝大多数资源都是通过标签来筛选和控制pod的**

一个简单的RC配置文件`kubia-rc.yaml`如下：
```yaml
apiVersion: v1
# 说明类型为 rc
kind: ReplicationController
metadata:
  name: kubia-controller
spec:
  # 这里指定了期望的副本数量
  replicas: 3
  # 这里指定了目标 pod 的选择器
  selector:
    # 目标 pod: app 标签的值为 kubia
    app: kubia
  # pod 模板
  template:
    metadata:
      # 指定 pod 的标签
      labels:
        app: kubia
    spec:
      # 指定 pod 容器的内容
      containers:
      - name: kubia
        image: luksa/kubia
        ports:
        - containerPort: 8080
```
使用`kubectl create -f kubia-rc.yaml`即可创建该副本控制器
此后，即使我们使用`kubectl delete`删除了该副本控制器管辖的3个pod，它仍然会自动创建新的。需要注意，RC是通过**标签**来管理pod的，如果修改了某个pod的标签，那么它就自动脱离了RC的控制


- **存活探针**

副本控制器RC是通过存活探针来讲pod的数量控制在期望数量， 存活探针的检测方式包括3种：`HTTP GET请求` 、`TCP套接字` 、`Exec执行命令`
一个带有存活探针的pod配置文件如下：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-liveness
spec:
  containers:
  - image: luksa/kubia-unhealthy
    name: kubia
    # 这下面就是探针！
    livenessProbe:
      httpGet:
        path: /
        port: 8080
```
### 1.4 ReplicationSet
ReplicationSet(RS)是升级版的RC，区别在于RS引入了对基于子集的selector查询条件，而RC仅支持基于值相等的selector条件查询
一个RS的配置文件`kubia-rs.yaml`如下：
```yaml
# 需要指定对应的 apiVersion
apiVersion: apps/v1beta2
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    # 只有这里和 rc 的写法不同
    matchLabels:
      app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
        ports:
        - containerPort: 8080
```
### 1.5 Deployment
Deployment为pod和RS提供声明式的更新能力, 通过在Deployment中描述期望的集群状态， Deployment Controller会将现在的集群状态逐步更新为期望的集群状态，其职责同样是为了保证pod的数量和健康，并且提供了RS之外的新特性：
事件和状态查看：可以查看Deployment的升级详细进度和状态。
　　　　**回滚**：当升级pod镜像或者相关参数的时候发现问题，可以使用回滚操作回滚到上一个稳定的版本或者指定的版本。
　　　　**版本记录**: 每一次对Deployment的操作，都能保存下来，给予后续可能的回滚使用。
　　　　**暂停和启动**：对于每一次升级，都能够随时暂停和启动。
　　　　**多种升级方案**：Recreate----删除所有已存在的pod,重新创建新的; RollingUpdate----滚动升级，逐步替换的策略，同时滚动升级时，支持更多的附加参数，例如设置最大不可用pod数量，最小升级间隔时间等等。


### 1.6 Service
由于pod在受到RC调控时，其副本和虚拟ip的变化的，比如发生迁移或伸缩的时候。因此需要一个统一固定的ip或域名来访问，Service是pod的路由代理抽象，用于解决pod之间的路由发现问题，访问端只需要知道service的地址，由service来提供代理，Service和Pod之间同样是通过label来进行关联
![service.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1607671122045-2e14c059-845e-4090-a9ba-35935b1dd1f9.png#align=left&display=inline&height=421&margin=%5Bobject%20Object%5D&name=service.png&originHeight=421&originWidth=864&size=28962&status=done&style=stroke&width=864)
接下来演示如何创建一个service:

- 首先使用1.4小节中的`kubia-rs.yaml`来创建3个pod：`kubectl create -f kubia-rs.yaml`

  创建成功之后就可以看到3个pod
![rs.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1607673418280-1df7615c-7200-4101-9f33-5a7268923a6d.png#align=left&display=inline&height=160&margin=%5Bobject%20Object%5D&name=rs.png&originHeight=160&originWidth=1268&size=39467&status=done&style=none&width=1268)

- 接下来新建service来提供对这3个pod的访问，新建`kubia-svc.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  ports:
  - name: port80 # 为所有端口指定名称
    port: 80 # 对外开放的服务端口
    targetPort: 8080 # 后方 pod 的服务端口
  selector:
    app: kubia
```
执行`kubectl create -f kubia-svc.yaml`来创建该服务，之后就可以看到它的信息：
![image.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1607673617273-2a231480-7e30-49c8-bda5-94b31baeb8b4.png#align=left&display=inline&height=141&margin=%5Bobject%20Object%5D&name=image.png&originHeight=141&originWidth=1360&size=44737&status=done&style=none&width=1360)
CLUSTER-IP一栏就是该服务的ip，节点内其他pod就可以通过这个ip来访问服务，例如通过pod`kubia-4p45t`来访问服务：
![service3.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1607673956066-7132bfd9-1b31-4e11-8399-fb9f45bb3330.png#align=left&display=inline&height=136&margin=%5Bobject%20Object%5D&name=service3.png&originHeight=136&originWidth=1058&size=32024&status=done&style=none&width=1058)
可以看到请求被转向了`kubia-k66bq`和`kubia-5dlks`这2个pod，并且实现了负载均衡

### 1.7 Endpoints
因为`svc`是通过我们事先定义好的标签选择器来查找 pod 的，所以 pod 的 ip 地址变动对于`svc`毫无影响，其实在`svc`和`pod`之间还包含了一个资源叫做`endpoint`，`endpoint`(简称`ep`)是一组地址及其端口的合集，如下图，只要一个`svc`有标签选择器的话，他就会自动创建一个同名的`ep`来标记出自己的要管理的 pod

![endpoint.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1607671225632-e4497afa-1552-49cc-99d0-03fc94e1ea0e.png#align=left&display=inline&height=513&margin=%5Bobject%20Object%5D&name=endpoint.png&originHeight=513&originWidth=408&size=23720&status=done&style=stroke&width=408)

- 通过域名访问服务

在1.6小节中我们通过ip的方式访问了服务，实际上k8s也提供了`FQDN`（全限定域名）的方式访问服务：
![name.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1607674417105-d629ed47-b1ff-4666-83bf-537d92ace1b3.png#align=left&display=inline&height=68&margin=%5Bobject%20Object%5D&name=name.png&originHeight=68&originWidth=1142&size=16547&status=done&style=none&width=1142)
上图中的kubia就是我们在创建`service`时指定的`name` 属性
(这种方式可以访问同一命名空间中的服务，k8s还支持访问不同命名空间中的服务)


- 访问集群外部服务

如果一个集群内部的pod要访问外部服务器，如云数据库、公共API等，就可以利用`Endpoint`来指定外部 服务的ip和端口，并将其绑定到一个服务`serivce`上
![13523736-e4c4aa978679fee1.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1607674767028-ba0918b6-aca8-4fae-9fba-e4105948242b.png#align=left&display=inline&height=503&margin=%5Bobject%20Object%5D&name=13523736-e4c4aa978679fee1.png&originHeight=503&originWidth=177&size=15742&status=done&style=stroke&width=177)
新建`external-service.yaml`如下，它只是提供了一个80端口供其他pod访问：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - port: 80
```


然后新建`external-endpoints.yaml`如下：
```yaml
apiVersion: v1
kind: Endpoints
metadata:
  # 和 svc 相同的名称
  name: external-service
subsets:
  - addresses:
    # 这里指定了外部服务的 ip
    - ip: 13.125.137.129
    # 可以指定多个
    # - ip: 22.22.22.22
    # 还要指定端口号
    ports:
    - port: 8000
```
执行创建命令： `kubectl create -f external-service.yaml`
`kubectl create -f external-endpoints.yaml`
然后查看服务详情，可以看到他已经和endpoint完成绑定
![endpoint2.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1607675210199-c22fbed8-b99c-4337-94b5-decd1ee35364.png#align=left&display=inline&height=497&margin=%5Bobject%20Object%5D&name=endpoint2.png&originHeight=497&originWidth=1381&size=90330&status=done&style=none&width=1381)


另外，也可以指定svc的类型type为`ExternalName`,再通过`ExternalName`字段来指定外部服务的域名，如下：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  # 要先指定 svc 的类型
  type: ExternalName
  # 再在这里指定外部服务的完全限定域名
  externalName: someapi.some.company.com
  ports:
  - port: 80
```
## 2.常用命令



| `get` | 查 | 列出某个类型的下属资源 |
| :---: | :---: | :---: |

| `describe` | 查 | 查看某个资源的详细信息 |
| :---: | :---: | :---: |

| `logs` | 查 | 查看某个 pod 的日志 |
| :---: | :---: | :---: |

| `create` | 增 | 新建资源 |
| :---: | :---: | :---: |

| `explain` | 查 | 查看某个资源的配置项 |
| :---: | :---: | :---: |

| `delete` | 删 | 删除某个资源 |
| :---: | :---: | :---: |

| `edit` | 改 | 修改某个资源的配置项 |
| :---: | :---: | :---: |

| `apply` | 改 | 应用某个资源的配置项 |
| :---: | :---: | :---: |



## 3.对外提供服务
k8s对外提供服务通常有`post-forward` 、`NodePort` 、`Ingress`几种方式


### 3.1 port-forward
port-forward适用于测试某个资源是否可用
`kubectl port-forward <资源类型>/<资源名> <本机端口>:<资源端口>`
![port-forward1.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1607675899436-0495205f-ceb4-4497-931a-ffdc034bc6a0.png#align=left&display=inline&height=367&margin=%5Bobject%20Object%5D&name=port-forward1.png&originHeight=367&originWidth=1193&size=81825&status=done&style=none&width=1193)
访问：
![port-forward2.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1607675911713-f948514d-cc07-4469-863e-ffe09ec45833.png#align=left&display=inline&height=58&margin=%5Bobject%20Object%5D&name=port-forward2.png&originHeight=58&originWidth=790&size=13271&status=done&style=none&width=790)
### 3.2 NodePort
NodePort可以将服务转发到所有k8s节点的指定端口上，其本质上也是一个`Service` 资源，通过指定`nodePort`来将服务映射到节点的端口上，配置如下：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30123
  selector:
    app: kubia
```
创建后，就可以通过`curl 节点ip:端口号` 来访问了，节点ip可以通过`kubectl describe node <节点名>`来查看


### 3.3 Ingress
使用NodePort已经可以满足暴露服务的需求，但是如果有多个服务，虽然可以通过nginx服务器来实现转发，但是如果服务有变更，就需要再次手动配置nginx，而k8s提供了另一种方式：Ingress

Ingress由2部分构成，服务转发服务的`ingress-controller`和`Ingress`资源，Ingress-controller根据Ingress资源提供的配置来转发流量， k8s本身没有自带Ingress-controller，这里我们使用官方推荐的`nginx-ingress-controller`
![ingress1.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1607676497846-8e24895e-6dd8-4337-b79c-15685f658724.png#align=left&display=inline&height=605&margin=%5Bobject%20Object%5D&name=ingress1.png&originHeight=605&originWidth=515&size=29492&status=done&style=stroke&width=515)

- 安装nginx-ingress-controller

官方配置文件： [https://github.com/kubernetes/ingress-nginx/blob/nginx-0.30.0/deploy/static/mandatory.yaml](https://github.com/kubernetes/ingress-nginx/blob/nginx-0.30.0/deploy/static/mandatory.yaml)
注意，官方的配置文件中，指定的`image`镜像源无法用国内网络访问，需要替换为国内镜像源。另外，**`Ingress-controller`本身只是一个pod,它并不直接对外暴露端口**，因此还需要新建一个NodePort将其暴露出去，完整配置文件如下：
[ingress-nginx.yaml](/ingress-nginx-controller.yml)


使用`kubectl create -f ingress-nginx.yaml`之后，就可以查看创建成功的Controller
![controller1.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1607677258423-54901efa-ebb0-4140-a13f-e80586a03b28.png#align=left&display=inline&height=84&margin=%5Bobject%20Object%5D&name=controller1.png&originHeight=84&originWidth=1069&size=23181&status=done&style=none&width=1069)

- 创建ingress资源

（注意，在这一步之前需要创建一个`service`及其`pod`，具体可参考1.6小节中的步骤）
新建`kubia-ingress.yaml`如下：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
    # 将服务映射到该域名
  - host: kubia.example.com
    http:
      paths:
        # 通过 /kubia 路径就可以访问该服务
      - path: /kubia
        # 该服务后端 svc 的名称及端口号
        backend:
          serviceName: kubia
          servicePort: 80
```

- 配置域名映射

在`ingress-nginx.yaml`中，我们已经将`nginx-ingress-controller`这个pod以30010端口暴露到主机上，接下来需要配置域名映射
`sudo vim /etc/hosts`
加入`172.17.0.3(节点ip) kubia.example.com`


接下来就可以在主机上通过`curl http://kubia.example.com:30010/kubia` 来访问服务了
![kubia.png](https://cdn.nlark.com/yuque/0/2020/png/480333/1607678407869-d644cd7a-dafb-4f40-a51d-52d2b15059b6.png#align=left&display=inline&height=212&margin=%5Bobject%20Object%5D&name=kubia.png&originHeight=212&originWidth=976&size=59226&status=done&style=none&width=976)


## 4.参考
[https://www.jianshu.com/p/8d60ce1587e1](https://www.jianshu.com/p/8d60ce1587e1)
[https://www.jianshu.com/p/116ce601a60f](https://www.jianshu.com/p/116ce601a60f)
