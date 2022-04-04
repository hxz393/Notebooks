# DaemonSet

下面简称DS.DS和RS相比多了一个nodeSelector选择器,DS依赖节点标签选择器来控制.

## 定义

使用DS来设置在集群中每个节点固定运行一个pod,通常用来部署系统服务,比如用来收集日志或者做资源监控.

也可以通过nodeSelector选择器来选择一组节点作为部署节点,而不是默认的所有节点.

如果被选择器选择的节点被设置为不可调度,DS依然会绕过调度器将pod部署到这种节点上.



## 创建DS

首先给一个节点打上标签disk='ssd':

```sh
[root@server4-master ~]# kubectl label node server5-node1 disk='ssd'
node/server5-node1 labeled
```

然后建立YAML配置文件并启动:

```sh
[root@server4-master ~]# vi kubia-ds.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds-ssd
spec:
  selector:
    matchExpressions:
      - key: app
        operator: In
        values:
          - kubia-ssd
  template:
    metadata:
      labels:
        app: kubia-ssd
    spec:
      nodeSelector:
        disk: ssd
      containers:
      - name: kubia
        image: luksa/kubia
[root@server4-master ~]# kubectl create -f kubia-ds.yaml 
daemonset.apps/ds-ssd created
```

DS还可以配置更新机制,相关配置定义在spec.update-Strategy字段,方式为RollingUpdate(滚动更新)或OnDelete(删除再更新).回滚操作同样支持.



## 验证

将server6-node2也打上ssd标签后再查看信息:

```sh
[root@server4-master ~]# kubectl label node server6-node2 disk='ssd'
[root@server4-master ~]# kubectl get pod -o wide
NAME           READY   STATUS    RESTARTS   AGE     IP               NODE     
ds-ssd-drb2z   1/1     Running   0          4m19s   10.244.191.199   server5-node1  
ds-ssd-f6cwh   1/1     Running   0          23s     10.244.244.199   server6-node2  
```

去除节点标签后运行在其上的pod会立即删除:

```sh
[root@server4-master ~]# kubectl label node server6-node2 disk-
[root@server4-master ~]# kubectl label node server5-node1 disk-
[root@server4-master ~]# kubectl get ds
NAME     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
ds-ssd   0         0         0       0            0           disk=ssd        7m52s
```

