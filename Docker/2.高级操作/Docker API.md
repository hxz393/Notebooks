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

采用Docker API通信,可以免去Docker客户端安装,同时效率更高更自由.

为了区分不同程序提供的接口,Docker API进行了划分.例如操作Docker镜像容器等模块接口称为Docker Remote API,操作管理Docker Registry远程仓库的服务接口称为Docker Registry API,操作Docker Cloud云服务的接口称为Docker Cloud API.



## Docker Remote API

Docker Remote API由Docker服务程序提供,是Docker API最重要部分,它能控制Docker服务及其中镜像,容器,网络等功能的运行.

其中常用操作方法如下:

| 功能         | 方法                         |
| ------------ | ---------------------------- |
| 列出容器     | GET /containers/json         |
| 创建容器     | POST /containers/create      |
| 查看容器信息 | GET /containers/(id)/json    |
| 查看容器进程 | GET /containers/(id)/atop    |
| 查看容器日志 | GET /containers/(id)/logs    |
| 查看文件变更 | GET /containers/(id)/changes |
| 导出容器     | GET /containers/(id)/export  |
| 启动容器     | GET /containers/(id)/start   |
| 停止容器     | GET /containers/(id)/stop    |
| 重启容器     | GET /containers/(id)/restart |
| 杀死容器     | GET /containers/(id)/kill    |
| 附加终端     | GET /containers/(id)/attach  |
| 暂停容器     | GET /containers/(id)/pause   |
| 恢复容器     | GET /containers/(id)/unpause |
| 等待容器停止 | GET /containers/(id)/wait    |
| 删除容器     | DELETE /containers/(id)      |
| 从容器复制   | POST /containers/(id)/copy   |
| 列出镜像     | GET /images/json             |
| 创建镜像     | POST /images/create          |
| 查看镜像信息 | GET /images/(name)/json      |
| 获取镜像历史 | GET /images/(name)/history   |
| 推送镜像     | POST /images/(name)/push     |
| 镜像贴标     | POST /images/(name)/tag      |
| 删除镜像     | DELETE /images/(name)        |
| 搜索镜像     | GET /images/search           |

也可以使用curl -X发送请求来测试:

```sh
[root@server4 ~]# curl -X GET http://192.168.2.234:5999/containers/json?all=1&size=1
[{"Id":"4608034f24faa0fde74fe3a53f89bd98af58bcc98c1d82ff51d8f19a9493ce15","Names":["/heuristic_ishizaka"],"Image":"alpine","ImageID":"sha256:14119a10abf4669e8cdbdff324a9f9605d99697215a0d21c360fe8dfa8471bab","Command":"/bin/sh","Created":1635662347,"Ports":[],"Labels":{},"State":"exited","Status":"Exited (0) 2 seconds ago","HostConfig":{"NetworkMode":"default"},"NetworkSettings":{"Networks":{"bridge":{"IPAMConfig":null,"Links":null,"Aliases":null,"NetworkID":"c441e7f297764fe10f2ded3a955f4c86f263210a56e77b7a19ce36ef300f44c0","EndpointID":"","Gateway":"","IPAddress":"","IPPrefixLen":0,"IPv6Gateway":"","GlobalIPv6Address":"","GlobalIPv6PrefixLen":0,"MacAddress":"","DriverOpts":null}}},"Mounts":[]}]
[root@server4 ~]# curl -X POST http://192.168.2.234:5999/containers/4608034f24faa/start
[root@server4 ~]# curl -X DELETE http://192.168.2.234:5999/containers/4608034?v=1&force=1
[1] 74386
```

详细用法可参考官网: https://docs.docker.com/engine/api/



## Docker Registry API

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



## Docker-py

docker-py为Docker官方提供的Python版本的API接口库,参考文档:https://docker-py.readthedocs.io/en/stable/

### 安装

使用pip install docker来安装:

```powershell
C:\Users\assassing>pip install docker
Successfully installed docker-5.0.3 pywin32-227 websocket-client-1.2.1
C:\Users\assassing>python
Python 3.8.1 (tags/v3.8.1:1b293b6, Dec 18 2019, 23:11:46) [MSC v.1916 64 bit (AMD64)] on win32
Type "help", "copyright", "credits" or "license" for more information.
>>> import docker
```

### 镜像查询

简单查看一下服务端信息:

```python
import docker

cli = docker.DockerClient(base_url='tcp://192.168.2.234:5999')
print('docker info:')
print(cli.info())
print('docker images:')
print(cli.images.list())
```

### 镜像构建

尝试使用dockerfile构建一个简单的镜像:

```python
import docker
from io import BytesIO

dockerfile = '''
#Share
FROM busybox
VOLUME /data
CMD ["/bin/sh"]
'''
f = BytesIO(dockerfile.encode('utf-8'))
cli = docker.DockerClient(base_url='tcp://192.168.2.234:5999')
response = [line for line in cli.images.build(fileobj=f, rm=True, tag='pytest/volume')]
print(response)
```

### 创建容器

启动刚才建立的镜像:

```python
import docker

cli = docker.DockerClient(base_url='tcp://192.168.2.234:5999')
container=cli.containers.run(image='pytest/volume',command='/bin/sleep 60')
print(container)
```

