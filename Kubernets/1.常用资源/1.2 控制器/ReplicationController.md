# ReplicationController

下面简称RC.可以通过kubectl api-resources命令查看所有缩写.

## 定义

ReplicationController用来确保pod始终保持运行状态,它主要通过pod标签用来控制副本数量.RC是用于复制和重新调度节点的最初组件,后引入ReplicaSet来代替它,不推荐再使用.

RC主要有三个部分组成:

- label selector(标签选择器): 用于确定RC作用域中的pod.
- replica count(副本个数): 指定应运行的pod数量.
- pod template(pod模板): 用于创建新的pod副本.

RC的配置可以随时修改:

- 修改副本数量会立即生效.
- 更改标签选择器和pod模板对运行中pod没有影响,只会使现有的pod脱离RC的控制范围.
- 修改模板仅影响由此RC创建的新pod.
- 如果更改一个pod的标签,那么它不再归RC管理,RC会自动启动一个新的pod来代替它.



## 创建RC

创建一个最简化运行3个pod副本的RC配置如下:

```sh
[root@server4-master ~]# vi kubia-rc.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
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
[root@server4-master ~]# kubectl create -f kubia-rc.yaml 
replicationcontroller/kubia created
```

标签选择器需要与pod的labels标签匹配,否则RC会一直启动新容器.假如不指定选择器selector,RC会根据pod模板中的标签自动配置.

运行后K8s会创建一个名为kubia的新RC,并且始终运行3个实例.没有足够pod时会根据模板创建新pod.下面手动删除一个pod,RC会立即重建一个新的容器:

```sh
[root@server4-master ~]# kubectl delete po kubia-wtlrp
pod "kubia-wtlrp" deleted
[root@server4-master ~]# kubectl get po
NAME          READY   STATUS        RESTARTS   AGE
kubia-2vpxd   1/1     Running       0          105s
kubia-9fqql   1/1     Running       0          105s
kubia-slltr   1/1     Running       0          22s
kubia-wtlrp   1/1     Terminating   0          105s
```

如果有节点故障,那么RC会在新的节点上启动节点故障上的pod,之后节点故障恢复也不会再迁移回去.



## 查看RC

查询当前运行的所有RC:

```sh
[root@server4-master ~]# kubectl get rc
NAME    DESIRED   CURRENT   READY   AGE
kubia   3         3         3       9m37s
```

查询具体RC的信息同样使用kubectl describe命令:

```sh
[root@server4-master ~]# kubectl describe rc kubia 
Name:         kubia
Namespace:    default
Selector:     app=kubia
Labels:       app=kubia
Annotations:  <none>
Replicas:     3 current / 3 desired
P Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
```



## 修改模板

可以通过kubectl edit rc命令来修改RC的pod模板,例如修改模板中的标签选择器和副本数:

```sh
[root@server4-master ~]# kubectl edit rc kubia
spec:
  replicas: 2
  selector:
    app: kubia1
  template:
    metadata:
      labels:
        app: kubia1
"/tmp/kubectl-edit-2608748675.yaml" 46L, 1157C written
replicationcontroller/kubia edited
```

查看pod和Label发现共有5个pod,新旧标签一同存在:

```sh
[root@server4-master ~]# kubectl get po --show-labels 
NAME          READY   STATUS    RESTARTS   AGE   LABELS
kubia-2c44j   1/1     Running   0          56s   app=kubia1
kubia-2vpxd   1/1     Running   0          40m   app=kubia
kubia-9fqql   1/1     Running   0          40m   app=kubia
kubia-ggk2j   1/1     Running   0          56s   app=kubia1
kubia-slltr   1/1     Running   0          38m   app=kubia
```



## 水平缩放

使用scale命令能修改RC配置中spec.replicas字段的数值,达到扩缩容效果:

```sh
[root@server4-master ~]# kubectl scale rc kubia --replicas=1
replicationcontroller/kubia scaled
[root@server4-master ~]# kubectl get pod --show-labels
NAME          READY   STATUS        RESTARTS   AGE     LABELS
kubia-2c44j   1/1     Running       0          5m18s   app=kubia1
kubia-ggk2j   1/1     Terminating   0          5m18s   app=kubia1
```



## 删除RC

当使用delete删除RC时,pod也会被删除.可以指定--cascade=orphan选项来删除RC同时保持pod运行:

```sh
[root@server4-master ~]# kubectl delete rc kubia --cascade=orphan
replicationcontroller "kubia" deleted
```

之后可以通过标签选择器创建新的RC或RS将它们再次管理起来.

