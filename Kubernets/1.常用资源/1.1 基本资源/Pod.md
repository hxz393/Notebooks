# Pod

## Pod定义

pod是一组并置的容器,代表了k8s中的基本构建模块.也可以理解为一个应用程序的单一运行实例,由共享资源且关系紧密的一个或多个应用容器组成,并为它们提供相同的环境.这些进程就好像全部运行于一个容器中一样,同时又保持着隔离性.

在实际中并不会单独部署容器,更多是针对一组pod的容器进行部署的操作.当一个pod包含多个容器时,这些容器总是运行于同一个工作节点上,一个pod绝不会跨越多个工作节点.

pod拥有以下特点:

- 同pod中容器部分隔离

  Kubernetes通过配置docker来让一个pod内的所有容器共享相同的Linux命名空间,它们共享相同的主机名和网络接口,容器之间能通过IPC进行通信.其他的MNT,USR和PID名称空间可以分别独立.

- 同pod中容器共享网络命名空间

  在pod中的容器处于同一个网络命名空间中,意味着共享pod的IP地址和端口.所以需要注意容器绑定的端口号不能相同,否则会导致端口冲突.同一pod中的容器可以直接通过回环口(loopback)来通信.

- 平坦的pod间网络

  K8s集群中所有的pod都在同一个共享网络地址空间中,pod之间可以直接通过pod的IP地址来实现互相访问.不管实际节点的网络拓扑结构如何,都不需要NAT网络地址转换,就像在局域网中一样.



## 组织Pod

虽然pod看上去像一个独立机器,但不应该将多个应用填充到一个pod中.每个pod只应包含紧密相关的组件或进程,保持pod的轻量级,这样可以尽可能多创建pod:

- 将多层应用分散到多个pod

  一个典型的应用由前端和后端数据库组成,如果将两者放入同一个容器,那么它们作为整体只能在一个节点运行,无法利用另外节点的计算资源.把他们拆分后可以在不同节点分别布置前端和后端应用,提高基础构架的利用率.

- 基于扩缩容考虑

  pod是扩缩容的基本单位,通常前端和后端组件具有不同的扩缩容需求,后端数据库比无状态的前端更难扩展,因此需要分开部署到单独pod中.

- 使用Sidecar容器

  将多个容器添加到单个pod的主要原因是:应用可能由一个主进程和多个辅助进程组成.此时辅助程序所在的容器被称为Sidecar容器,通常用来做日志收集,数据处理,储存与通信等.

- pod使用多个容器

  如果多个容器作为整体不可分离,必须一起进行扩缩容时,才考虑装入同一个pod中.



## 生命周期

pod对象有5种状态:

- Pending

  API Server创建了pod资源对象并已存入etcd中,但它尚未被调度完成,或仍处于下载镜像中.

- Running

  pod已被调度到某节点,所有容器都已被Kubelet创建完成.

- Succeeded

  pod中所有容器都已成功终止且不会被重启.

- Failed

  所有容器都已终止,至少有一个容器终止失败,返回了非0的退出状态.

- Unknown

  API Server无法正常获取pod对象的状态信息,通常是由于节点失联造成.

  

## Pause容器

每个pod都有一个特殊的被成为根容器的Pause容器,Pause容器对应的镜像属于Kubernetes平台的一部分,剩余是用户业务容器.

启动一个pod资源后,查看pod运行中的节点docker进程:

```sh
[root@server5-node1 ~]# docker ps
CONTAINER ID   IMAGE                                                                 COMMAND                  CREATED              STATUS              PORTS     NAMES
e9d77706b60a   luksa/kubia-pet-peers                                                 "node app.js"            About a minute ago   Up About a minute             k8s_kubia_kubia-2_default_49130122-540c-4c8b-98aa-23c84e9bfc9a_0
fe1980dc3fb7   registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.2   "/pause"                 About a minute ago   Up About a minute             k8s_POD_kubia-2_default_49130122-540c-4c8b-98aa-23c84e9bfc9a_0
```

附加Pause容器会先于应用容器创建,它不做任何事情,仅仅运行一个pause命令.

由于pod内的容器共享同一个网络和Linux命名空间,引入业务无关且不易死亡的Pause容器作为pod根容器,以它的状态代表整个容器组的状态,能让我们更简单地判断pod运行状态.它的声明周期和pod绑定,如果基础pod在这期间被关闭,Kubelet会重新创建它以及pod的所有容器.

另一层面如果pod中有多个容器,通过Pause容器挂接外部卷,共享Pause容器IP,简化了业务容器之间的通信问题.



## 描述文件

pod或其他资源通常是通过向REST API提供JSON或YAML描述文件来创建.可以通过kubectl get命令加入-o yaml参数来查看YAML定义:

```sh
[user1@server6 ~]$ kubectl get pod kubia -o yaml
apiVersion: v1
kind: pod
metadata:
  creationTimestamp: "2021-11-02T21:50:57Z"
  labels:
    run: kubia
  name: kubia
```

pod定义文件中的结构几乎在所有K8s资源中都可以找到:

- apiVersion: K8s API版本.当前具体资源使用API版本可以通过kubectl api-resources命令查到.
- kind: 资源类型.
- metadata: 元数据,包括名称,命名空间,标签和容器其他信息.
- spec: 包括pod内容的实际说明,例如容器和卷.
- status: 包含运行中pod的当前信息,例如容器状态和内部IP信息.

可以通过kubectl explain pods命令看到关于yaml中各属性的说明.需要了解详细可以直接查看属性,比如kubectl explain pod.status:

```sh
[user1@server6 ~]$ kubectl explain pod.status

FIELDS:
   conditions   <[]Object>
     Current service state of pod. More info:
     https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#pod-conditions

   containerStatuses    <[]Object>
     The list has one entry per container in the manifest. Each entry is
     currently the output of `docker inspect`. More info:
     https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#pod-and-container-status
```

下面是一个pod基本的yaml描述文件:

```yaml
[user1@server6 ~]$ vi kubia-pod.yaml
apiVersion: v1
kind: pod
metadata:
  name: alice-pod
spec:
  containers:
  - image: assassing/kubia
    name: assassing
    ports:
    - containerPort: 8080
      protocol: TCP
```

在pod定义中指定的端口完全是为了可读性,实际并没有作用.如果容器通过绑定到地址0.0.0.0的端口接收连接,那么即使端口没写在pod spec中,其他pod依然能连接到该端口.



## 创建Pod

可以使用kubectl create命令从YAML文件创建包括pod在内的任何资源:

```sh
[user1@server6 ~]$ kubectl create -f kubia-pod.yaml 
pod/alice-pod created
[user1@server6 ~]$ kubectl get pod
NAME        READY   STATUS    RESTARTS   AGE
alice-pod   1/1     Running   0          12s
```

除了YAML格式也支持从JSON文件创建资源.



## 查看Pod详情

查看详细信息也可以输出成为JSON格式:

```sh
[user1@server6 ~]$ kubectl get po alice-pod -o json
{
    "apiVersion": "v1",
    "kind": "pod",
    "metadata": {
        "creationTimestamp": "2021-11-02T23:37:42Z",
        "name": "alice-pod",
        "namespace": "default",
        "resourceVersion": "6027",
        "uid": "de10a52f-d380-4341-9c11-2bd516cf0a02"
    },
```



## 查看Pod日志

使用kubectl logs命令查看日志不需要像docker logs命令一样,到容器运行的主机上查询:

```sh
[user1@server6 ~]$ kubectl logs alice-pod 
Runing...
```

如果pod中包含多个容器,则必须通过-c参数指定容器名称:

```sh
[root@localhost flannel]# kubectl logs httpd-app-5bc589d9f7-ggmwq -c httpd-app
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.244.1.5. Set the 'ServerName' directive globally to suppress this message
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.244.1.5. Set the 'ServerName' directive globally to suppress this message
[Thu Jul 18 16:28:44.242092 2019] [mpm_event:notice] [pid 1:tid 139744563483776] AH00489: Apache/2.4.39 (Unix) configured -- resuming normal operations
[Thu Jul 18 16:28:44.242331 2019] [core:notice] [pid 1:tid 139744563483776] AH00094: Command line: 'httpd -D FOREGROUND'
```

每天或每次日志文件达到10MB大小时,容器日志都会自动轮替.同时当pod被删除时,日志也会被删除.



## 执行命令

使用和Docker命令类似的格式kubectl exec -it来进入容器,虽然可行但会提示此命令已废弃:

```sh
[root@server4-master ~]# kubectl exec -it kubiaex-f4wkw /bin/bash
kubectl exec [pod] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [pod] -- [COMMAND] instead.
```

原因是当要在容器运行的命令中带空格时,kubectl会将空格后面的内容作为kubectl的参数解析.正确用法是在容器命令前用两个减号--来分隔:

```sh
[root@server4-master ~]# kubectl exec -it kubiaex-f4wkw -- bash
```



## 转发Pod端口

除了使用service的方式,还能通过端口转发来连接pod.命令是`kubectl port-forward pod名 本地端口:pod端口`.

例如转发alice-pod的8080端口到本地8888:

```sh
[user1@server6 ~]$ kubectl port-forward alice-pod 8888:8080
Forwarding from 127.0.0.1:8888 -> 8080
Forwarding from [::1]:8888 -> 8080
Handling connection for 8888
[root@server6 ~]# curl 127.0.0.1:8888
Hostname: alice-pod
```

这个方法可以用来调试应用.



## 删除Pod

使用kubectl delete命令来删除pod,可以用空格分隔要删除的多个pod:

```sh
[user1@server6 ~]$ kubectl delete pod alice-pod alice-pod-v1
pod "alice-pod" deleted
pod "alice-pod-v1" deleted
```

在删除pod的过程中,实际在请求终止该pod中的所有容器.Kubernetes向进程发送一个SIGTERM信号默认等待30秒使其正常关闭.超过30秒则通过SIGKILL终止进程.

通过-l参数来通过标签选择pod并删除:

```sh
[root@localhost ~]# kubectl delete pod -l run=assassing
pod "assassing-5c54d7b988-76kdz" deleted
pod "assassing-5c54d7b988-hbxtp" deleted
```

删除当前命名空间内所有pod,但保留命名空间:

```sh
[root@localhost ~]# kubectl delete pod --all
```

删除当前命名空间内资源,但保留命名空间:

```sh
[user1@server6 ~]$ kubectl delete all --all
service "kubernetes" deleted
service "kubia" deleted
service "kubia-http" deleted
```

