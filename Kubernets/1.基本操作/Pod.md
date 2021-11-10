# Pod

## 基本操作

通常不需要手动建立Pod.

### 描述文件

下面是一个Pod基本的yaml描述文件:

```yaml
[user1@server6 ~]$ vi kubia-pod.yaml
apiVersion: v1
kind: Pod
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

在Pod定义中指定的端口完全是为了可读性,实际并没有作用.如果容器通过绑定到地址0.0.0.0的端口接收连接,那么即使端口没写在pod spec中,其他pod依然能连接到该端口.

### 创建Pod

可以使用kubectl create命令从YAML文件创建Pod:

```sh
[user1@server6 ~]$ kubectl create -f kubia-pod.yaml 
pod/alice-pod created
[user1@server6 ~]$ kubectl get pod
NAME        READY   STATUS    RESTARTS   AGE
alice-pod   1/1     Running   0          12s
```

除了YAML格式也支持从JSON文件创建资源.

### 查看Pod详情

查看详细信息也可以输出成为JSON格式:

```sh
[user1@server6 ~]$ kubectl get po alice-pod -o json
{
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
        "creationTimestamp": "2021-11-02T23:37:42Z",
        "name": "alice-pod",
        "namespace": "default",
        "resourceVersion": "6027",
        "uid": "de10a52f-d380-4341-9c11-2bd516cf0a02"
    },
```

### 查看Pod日志

使用kubectl logs命令查看日志不需要像docker logs命令一样,到容器运行的主机上查询:

```sh
[user1@server6 ~]$ kubectl logs alice-pod 
Runing...
```

如果Pod中包含多个容器,则必须通过-c参数指定容器名称:

```sh
[root@localhost flannel]# kubectl logs httpd-app-5bc589d9f7-ggmwq -c httpd-app
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.244.1.5. Set the 'ServerName' directive globally to suppress this message
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.244.1.5. Set the 'ServerName' directive globally to suppress this message
[Thu Jul 18 16:28:44.242092 2019] [mpm_event:notice] [pid 1:tid 139744563483776] AH00489: Apache/2.4.39 (Unix) configured -- resuming normal operations
[Thu Jul 18 16:28:44.242331 2019] [core:notice] [pid 1:tid 139744563483776] AH00094: Command line: 'httpd -D FOREGROUND'
```

每天或每次日志文件达到10MB大小时,容器日志都会自动轮替.同时当Pod被删除时,日志也会被删除.

### 执行命令

使用和Docker命令类似的格式kubectl exec -it来进入容器,虽然可行但会提示此命令已废弃:

```sh
[root@server4-master ~]# kubectl exec -it kubiaex-f4wkw /bin/bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
```

原因是当要在容器运行的命令中带空格时,kubectl会将空格后面的内容作为kubectl的参数解析.正确用法是在容器命令前用两个减号--来分隔:

```sh
[root@server4-master ~]# kubectl exec -it kubiaex-f4wkw -- bash
```

### 转发Pod端口

除了使用service的方式,还能通过端口转发来连接Pod.命令是`kubectl port-forward pod名 本地端口:pod端口`.

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

### 删除Pod

使用kubectl delete命令来删除Pod,可以用空格分隔要删除的多个Pod:

```sh
[user1@server6 ~]$ kubectl delete pod alice-pod alice-pod-v1
pod "alice-pod" deleted
pod "alice-pod-v1" deleted
```

在删除pod的过程中,实际在请求终止该pod中的所有容器.Kubernetes向进程发送一个SIGTERM信号默认等待30秒使其正常关闭.超过30秒则通过SIGKILL终止进程.

通过-l参数来通过标签选择Pod并删除:

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



## 标签注解

标签是可以附加到资源的任意键值对,可以通过标签选择器来选择具有确切标签的资源,使得能一次操作所有相同标签资源.只要标签的key在资源内是唯一的,一个资源便可以拥有多个标签,并且可以随时修改与添加.

### 指定标签

创建一个新的kubia-label.yaml文件,增加labels栏:

```yaml
[user1@server6 ~]$ vi kubia-label.yaml
apiVersion: v1
kind: Pod
metadata:
  name: alice-pod-v1
  labels:
    creation_method: manual
    env: prod
spec:
  containers:
  - image: assassing/kubia
    name: assassing
    ports:
    - containerPort: 8080
      protocol: TCP
```

用新的配置文件创建Pod:

```sh
[user1@server6 ~]$ kubectl create -f kubia-label.yaml 
pod/alice-pod-v1 created
```

一般常用标签有: 版本标签(release:stable),环境标签(environment:dev),构架标签(tier:frontend),分区标签(partition:HK),质量管控标签(track:daily).

### 查看标签

使用--show-labels参数来查看标签:

```sh
[user1@server6 ~]$ kubectl get pods --show-labels
NAME           READY   STATUS    RESTARTS   AGE   LABELS
alice-pod      1/1     Running   0          32m   <none>
alice-pod-v1   1/1     Running   0          53s   creation_method=manual,env=prod
```

可以用-L显示指定标签而不是显示所有标签:

```sh
[user1@server6 ~]$ kubectl get pods -L env,creation_method
NAME           READY   STATUS    RESTARTS   AGE     ENV    CREATION_METHOD
alice-pod      1/1     Running   0          34m            
alice-pod-v1   1/1     Running   0          2m55s   prod   manual
```

### 修改标签

为现有pod添加标签使用kubectl label命令:

```sh
[user1@server6 ~]$ kubectl label pod alice-pod env=test
pod/alice-pod labeled
[user1@server6 ~]$ kubectl get pods -L env
NAME           READY   STATUS    RESTARTS   AGE     ENV
alice-pod      1/1     Running   0          36m     test
alice-pod-v1   1/1     Running   0          4m29s   prod
```

要修改已存在的标签,需要使用--overwrite参数:

```sh
[user1@server6 ~]$ kubectl label pod alice-pod env=prod --overwrite 
pod/alice-pod labeled
[user1@server6 ~]$ kubectl get pods -L env
NAME           READY   STATUS    RESTARTS   AGE     ENV
alice-pod      1/1     Running   0          37m     prod
alice-pod-v1   1/1     Running   0          5m47s   prod
```

一个标签可以附加于多个对象.

### 删除标签

例如删除alice-pod-v1上的creation_method标签:

```sh
[user1@server6 ~]$ kubectl label pod alice-pod-v1 creation_method-
pod/alice-pod-v1 labeled
```

### 标签选择器

标签选择器允许选择标记有特定标签的pod子集并进行操作.可以使用的条件如下:

- 包含(或不包含)使用特定键的标签.

  例如选择所有不存在env键名标签的资源: `!env`

- 包含具有特定键和值的标签.

  同时选择两个标签,标签之间用逗号分开.例如: `app=pc,rel=beta`

  选择env标签,且值为prod或dev的pod: `env in (prod,dev)`

- 包含具有特定键的标签,但值与指定的不同.

  选择带有env标签,并且值不等于prod的pod: `env!=prod `

  选择env标签,且值不为prod或dev的pod: `env notin (prod,dev) `

例如列出标签env=prod的pod:

```sh
[user1@server6 ~]$ kubectl get po -l env=prod
NAME           READY   STATUS    RESTARTS   AGE
alice-pod      1/1     Running   0          53m
alice-pod-v1   1/1     Running   0          21m
```

使用!做选择时,必须将条件用单引号圈引,否则会被bash解释.例如列出标签key不存在env的pod:

```sh
[user1@server6 ~]$ kubectl get po -l '!env'
NAME           READY   STATUS    RESTARTS   AGE
alice-pod      1/1     Running   0          54m
alice-pod-v1   1/1     Running   0          23m
```

### 使用标签约束调度

标签可以附加到任何Kubernetes对象上,包括节点.因此可以在添加新节点时,用标签对节点进行分类.例如一组美国服务器的节点,可以添加标签location=US:

```sh
[root@localhost ~]# kubectl label node k8s-node1 location=US
node/k8s-node1 labeled
[root@localhost ~]# kubectl get nodes -l location=US
NAME        STATUS   ROLES    AGE   VERSION
k8s-node1   Ready    <none>   85m   v1.15.0
```

如果需要部署Pod到美国服务器,在YAML文件中添加一个节点选择器nodeSelector:

```sh
[root@localhost ~]# vi kubia-US.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: alice-pod-us
spec:
  nodeSelector:
    location: "US"
  containers:
  - image: assassing/kubia
    name: assassing
[root@localhost ~]# kubectl get pod --all-namespaces -o wide
NAMESPACE  NAME         READY  STATUS RESTARTS AGE IP          NODE       NOMINATED NODE
default    alice-pod-us 1/1    Running 0       32s 10.244.1.6  k8s-node1  <none>      
default    alice-pod-v1 1/1    Running 0       26m 10.244.2.5  k8s-node2  <none>      
```

### 添加注解

注解只是为了保存标识信息而存在,不能像标签一样分组,但可以容纳更多信息,主要用于工具使用.

使用kubectl annotate命令来添加注解:

```sh
[user1@server6 ~]$ kubectl annotate pod alice-pod build="20160602"
pod/alice-pod annotated
[user1@server6 ~]$ kubectl describe pod alice-pod
Labels:       env=prod
Annotations:  build: 20160602
```



## 命名空间

K8s的命名空间(Namespace)和用于互相隔离进程的Linux命名空间不一样,其不提供对正在运行对象的任何隔离,只是简单地为对象名称提供一个作用域,这样在不同命名空间允许多次使用相同的资源名称.

命名空间为资源名称提供了一个作用域.可以通过命名空间将资源分配为生产,开发,测试环境.资源名称只需要在命名空间内保持唯一即可.虽然大多数类型的资源都与命名空间相关,但仍有一些比如全局且未被约束于单一命名空间的节点资源与它无关.

### 查看命名空间

默认情况下在default命名空间中进行操作.可使用kubectl get namespace或kubectl get ns来查看所有命名空间:

```sh
[user1@server6 ~]$ kubectl get namespace
NAME                   STATUS   AGE
default                Active   4h12m
kube-node-lease        Active   4h12m
kube-public            Active   4h12m
kube-system            Active   4h12m
kubernetes-dashboard   Active   4h11m
```

查询属于特定namespace的pod:

```sh
[user1@server6 ~]$ kubectl get po -n kubernetes-dashboard
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-758548f849-2b7xn   1/1     Running   0          4h12m
kubernetes-dashboard-586fb768c4-kkhkm        1/1     Running   0          4h12m
```

### 创建命名空间

可以通过YAML文件创建命名空间.命名空间名字不能包含点号:

```sh
[user1@server6 ~]$ vi my-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-namespace
[user1@server6 ~]$ kubectl create -f my-namespace.yaml 
namespace/my-namespace created
```

也能直接通过create namesapce命令创建命名空间:

```sh
[user1@server6 ~]$ kubectl create namespace my-ns
namespace/my-ns created
```

### 使用命名空间

可以在创建pod的YAML文件中向metadata字段添加namespace: 

```sh
[user1@server6 ~]$ vi kubia-ns.yaml
apiVersion: v1
kind: Pod
metadata:
  name: alice-ns
  namespace: my-namespace
spec:
  containers:
  - image: assassing/kubia
    name: assassing
```

也可以在kubectl create命令创建资源时加上-n参数指定命名空间:

```sh
[user1@server6 ~]$ kubectl create -f kubia-ns.yaml -n my-namespace 
pod/alice-ns created
```

### 查询命名空间

要查看命名空间下的Pod,通过在查询命令后加上-n选项选择命名空间:

```sh
[user1@server6 ~]$ kubectl get pod -n my-namespace
NAME       READY   STATUS    RESTARTS   AGE
alice-ns   1/1     Running   0          45s
```

可以使用--all-namespaces参数来查看所有命名空间下的Pod:

```sh
[user1@server6 ~]$ kubectl get pod --all-namespaces
```

### 删除命名空间

可以直接删除整个命名空间,命名空间下的Pod将会随着删除:

```sh
[user1@server6 ~]$ kubectl delete ns my-namespace
namespace "my-namespace" deleted
```



## 存活探针

K8s可以通过存活探针(Liveness Probe)检查容器是否还在运行.可以为Pod中每个容器单独指定存活探针,如果探测失败,K8s将定期执行探针并重新启动容器.

K8s有三种探测容器的机制:

- ExecAction
  在容器中执行一个命令,根据返回的状态码进行诊断,状态码为0表示成功,否则不健康.

- TCPSocketAction
  通过与容器的某TCP端口尝试建立连接进行诊断,端口能打开表示正常.

- HTTPGetAction
  通过向容器IP地址某个端口的指定路径发起HTTP GET请求进行诊断,响应码为2xx或3xx时即为成功.

存活探针由各节点上Kubelet负责执行.

### HTTP探针

Http探针用于探测Web服务器是否提供请求很有用.这里使用一个在第五个请求后返回500状态码的程序,创建Pod用的yaml文件如下:

```sh
[root@server4-master ~]# vi kubia-live.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-liveness
spec:
  containers:
  - image: luksa/kubia-unhealthy
    name: kubia
    livenessProbe:
      httpGet:
        path: /
        port: 8080
[root@server4-master ~]# kubectl create -f kubia-live.yaml 
```

上面定义了一个httpGet方式的存活探针,会定义检查Pod的8080端口以确定容器是否健康.

启动Pod后观察Pod的状态:

```sh
[root@server4-master ~]# watch kubectl get po
NAME             READY   STATUS    RESTARTS      AGE       
kubia-liveness   1/1     Running   2 (12h ago)   4m8s  
```

大约在过了两分钟后存活探针检测到返回码500即会重启容器,并给重启计数加一,如此循环.

查看下Pod日志.由于kubectl logs只会打印当前容器的日志,想看前一个容器日志需要使用--previous选项:

```sh
[root@server4-master ~]# kubectl logs kubia-liveness --previous
Kubia server starting...
Received request from ::ffff:192.168.2.206
Received request from ::ffff:192.168.2.206
Received request from ::ffff:192.168.2.206
Received request from ::ffff:192.168.2.206
```

日志显示的请求访问都来自存活探针.再看以下Pod的描述:

```sh
[root@server4-master ~]# kubectl describe po kubia-liveness
    Last State:     Terminated
      Reason:       Error
      Exit Code:    137
      Started:      Wed, 03 Nov 2021 14:38:46 +0800
      Finished:     Wed, 03 Nov 2021 14:40:33 +0800
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Pulled     12h                kubelet            Successfully pulled image "luksa/kubia-unhealthy" in 40.786546605s
  Normal   Pulled     12h                kubelet            Successfully pulled image "luksa/kubia-unhealthy" in 3.188020575s
  Normal   Created    12h (x3 over 12h)  kubelet            Created container kubia
  Normal   Pulled     12h                kubelet            Successfully pulled image "luksa/kubia-unhealthy" in 3.281767877s
  Normal   Started    12h (x3 over 12h)  kubelet            Started container kubia
  Warning  Unhealthy  12h (x9 over 12h)  kubelet            Liveness probe failed: HTTP probe failed with statuscode: 500
```

可以清楚地看到上次退出代码为137=128+9,表示进程由外部信号终止,9代表SIGKILL强行终止信号.当容器被强行终止时,会创建一个全新的容器而不是重启原先的容器.

### Exec探针

只有一个可用属性command,用于指定要执行的命令:

```sh
        livenessProbe:
          exec:
            command: ["test", "-e", "/tmp/healthy"]
```

### TCP探针

相比来说比HTTP的探测更高效和节约资源,但精度略低.毕竟连接建立成功并不意味页面资源可用:

```sh
        livenessProbe:
          tcpSocket:
            port: 443
```

### 附加属性

可以给存活探针增加附加属性,可以设置五个属性:

- initialDelaySeconds: 表示容器启动后等待多少秒开始探测,默认为0s.
- timeoutSeconds: 表示响应时间超过多少秒为失败,默认为1s.
- periodSeconds: 表示探针探测周期时间秒,默认10s.
- successThreshold: 表示探测成功几次后,将Pod状态恢复成正常,默认1次.
- failureThreshold: 表示连续探测失败几次后重启容器,默认3次.

其中最常用的是设置初始化等待时间,将默认启动后即开始探测改为15秒:

```sh
[root@localhost ~]# vi kubia-http-liveness-probe.yaml 
    livenessProbe:
      httpGet:
        path: /
        port: 8080
      initialDelaySeconds: 15
```

