# Kubelet

## 工作内容

Kubelet负责所有运行在工作节点上内容的组件.

其工作内容是:

- 在API服务器中创建一个Node资源来注册Kubelet所在节点.
- 持续监控API服务器是否把该节点分配给pod.

- 告知节点上的容器运行时拉取镜像,启动容器.
- 持续监控运行的容器,向API服务器报告它们的状态,事件和资源消耗.

- 在pod被从API服务器删除时,Kubelet终止容器并通知服务器pod已被终止.



## 运行静态pod

 Kubelet还可以基于本地指定目录下的pod清单来运行pod,主要用于运行容器化版本的控制面板组件.

静态pod总由kubelet创建在kubelet所在的Node上运行,由kubelet管理.不能通过API Server进行管理,不能与控制器进行关联,kubelet也无法对它们进行健康检查.

启动静态pod需要设置kubelet的启动参数--config,指定需要监控的配置文件所在目录,并根据yaml进行创建操作.例如配置--config=/etc/kubelet.d/,然后重启kubelet服务.在目录下写入static-web.yaml文件:

```sh
[root@server4-master ~]# vi static-web.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  labels:
    name: static-web
  namespace: default
spec:
  containers:
  - name: static-web
    image: nginx
    ports:
    - name: web
      containerPort: 80
```

等待一段时间可以看到pod被创建出来.假如在Master节点删除pod会使其变成Pending状态且不会被删除.

还能通过设置kubelet的启动参数--manifest-url,kubelet将会定期从指定URL地址下载定义文件创建pod.

