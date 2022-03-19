# ReplicaSet

下面简称RS.RS和RC都是依赖pod标签选择器来控制.

## 定义

RS的行为和用法与RC几乎完全相同.和RC相比,RS的选择器还允许反向选择,或通过标签key来选择pod.例如选择所有带env标签的pod(env=*).

通常不会直接创建RS对象,而是通过创建Deployment资源时自动创建.Deployment通过RS来管理pod的多个副本.



## 创建RS

建立yaml配置文件并发布.和RC配置不同的是apiVersion版本, kind类型和标签选择器样式:

```sh
[root@server4-master ~]# vi kubia-rs.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
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
        - name: http
          containerPort: 80
[root@server4-master ~]# kubectl create -f kubia-rs.yaml
replicaset.apps/kubia created
```



## 标签选择

标签选择器和pod中使用的一样,不过写法有些不同.例如通过matchLabels匹配单个标签:

```sh
[root@server4-master ~]# vi kubia-rs.yaml
spec:
  selector:
    matchLabels:
      app: kubia
```

例如通过matchExpressions表达式同时选择两个不同app值的标签:

```sh
[root@server4-master ~]# vi kubia-rs.yaml
spec:
  selector:
    matchExpressions:
      - key: app
        operator: In
        values:
          - kubia
          - kubia1
```

operator还可以使用其他运算符: NotIn(不在列表中),Exists(匹配Key),DoesNotExist(反向匹配Key)

如果指定了多个表达式,所有表达式匹配结果都必须为true才能使选择器与pod匹配.

如果同时使用matchLabels和matchExpressions,则所有标签都必须匹配.