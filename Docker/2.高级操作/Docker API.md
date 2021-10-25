# Docker API

## RESTful

Docker API是一套基于RESTful(Rspresentational State Transfer, 表述性状态转移)设计的HTTP,用于操作Docker服务的接口.它实现在Docker服务程序中,也由Docker服务程序向外提供.

RESTful设计中有几个比较关键的概念:

- Resource(资源)

  RESTful接口操作或获取的对象就是资源,在Docker里就是容器,镜像,数据卷等实体.

- Representation(表述)

  因为资源形式各式各样,所以在RESTful设计中要求采用可读性的格式去展示资源.在Docker API中大都以JSON文本的形式表述资源.

- State Transfer(状态转移)

  对于资源的修改可以理解为资源状态发生的变化,也就是状态转移.在RESTful设计中主张用HTTP中的方法确定资源的操作方式,例如用GET获取,POST新增,PUT修改,DELETE删除等.



## Docker API

为了区分不同程序提供的接口,Docker API进行了划分.例如操作Docker镜像容器等模块接口称为Docker Remote API,操作管理Docker Registry远程仓库的服务接口称为Docker Registry API,操作Docker Cloud云服务的接口称为Docker Cloud API.

### Docker Remote API

Docker Remote API由Docker服务程序提供,是Docker API最重要部分,它能控制Docker服务及其中镜像,容器,网络等功能的运行.

默认配置中,Docker Remote API通过Socket监听来自本地的连接,监听地址位于unix:///var/run/docker.sock.可以通过curl发送一个简单的请求:

```sh
[root@server4 ~]# curl --unix-socket /var/run/docker.sock http://localhost/info
{"ID":"7YVL:YG7U:BI3Q:BOVZ:XRWD:7BQO:6H3R:2BUW:OCQO:7JKZ:2OWC:UL2Q","Containers":5,"ContainersRunning":4,"ContainersPaused":0,"ContainersStopped":1,"Images":22,"Driver":"
```

### Docker Registry API

Docker Registry API可以管理与使用远程镜像仓库Docker Registry.主要提供以下功能:

- 镜像信息操作

  镜像信息指镜像ID,仓库名,标签,启动命令等能全名描述镜像的内容.无论想要推送还是拉取镜像前,都要进行镜像信息的推送和拉取.

- 镜像验证

  可以通过Docker Registry API所给出的镜像信息中相关字段进行校验,保证镜像数据完整没有被修改.

- 镜像推送

  Docker Registry API提供了分块方式将镜像数据推送到Docker Registry服务器,有利于提高传输速度.镜像服务器收到镜像数据后先进行数据组装,对于缺失部分可以补充上传.

- 镜像拉取

  拉取镜像时同样通过分块方式拉取,再到本地进行组装,提高镜像传输效率.

- 镜像层控制

  当推送一个镜像层到仓库时,Docker Registry API会通过传输的镜像散列值,判断镜像层是否已经存在仓库中,有的话不必传输.

