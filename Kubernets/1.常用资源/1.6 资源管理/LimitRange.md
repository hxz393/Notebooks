# LimitRange

## 介绍

可以通过创建一个LimitRange资源来给命名空间中的容器配置最大和最小限额,同时给没有指定限制的容器设置默认值.

LimitRange资源被LimitRanger准入控制插件控制,当API服务器接收带有pod描述信息的POST请求时,会检查设置的限制值是否在范围内,来决定是否创建资源,避免无法调度成功.

LimitRange作用于同一命名空间中每个独立pod,容器或其他类型对象.它不会限制命名空间中所有pod可用资源的总量,总量是通过ResourceQuuota对象指定.



## 创建LimitRange资源

下面配置展示了一个LimitRange资源的完整定义:

```sh
[root@server4-master ~]# vi LR-example.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: example
spec:
  limits:
  - type: Pod      ##指定了资源类型为Pod
    min:           ##Pod中所有容器的CPU和内存请求量之和的最小值
      cpu: 50m
      memory: 5Mi
    max:           ##Pod中所有容器的CPU和内存请求量之和的最大值
      cpu: 1
      memory: 1Gi
  - type: Container ##指定了资源类型为Container
    defaultRequest: ##容器没有指定请求量时设置的默认值
      cpu: 100m
      memory: 10Mi
    default:        ##容器没有指定限制量时设置的默认值
      cpu: 200m
      memory: 100Mi
    min:            ##容器requests和limits的最小值
      cpu: 50m
      memory: 5Mi
    max:            ##容器requests和limits的最大值
      cpu: 1
      memory: 1Gi
    maxLimitRequestRatio: ##每种资源requests与limits的最大比值
      cpu: 4
      memory: 10
  - type: PersistentVolumeClaim  ##指定请求PVC储存容量的最小最大值
    min:
      storage: 1Gi
    max:
      storage: 10Gi
[root@server4-master ~]# kubectl create -f LR-example.yaml
limitrange/example created
```

maxLimitRequestRatio中设置CPU为4表示容器的CPU limits不能超过requests的4倍,内存同理.

上面新建的LimitRange对象包含了对所有资源的限制,也可以将其分割为多个对象.多个LimitRange对象的限制会在校验pod或PVC合法性时进行合并.

对LimitRange进行修改,已经存在的pod和PVC不会受影响.



## 测试限制值

尝试创建一个CPU申请量大于LimitRange允许值的pod:

```sh
[root@server4-master ~]# vi pod-2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-2
spec:
  containers:
  - image: busybox
    command: ["dd", "if=/dev/zero", "of=/dev/null"]
    name: main
    resources:
      limits:
        cpu: 2
[root@server4-master ~]# kubectl create -f pod-2.yaml 
Error from server (Forbidden): error when creating "pod-2.yaml": pods "pod-2" is forbidden: [maximum cpu usage per Pod is 1, but limit is 2, maximum cpu usage per Container is 1, but limit is 2]
```

服务器返回建立pod被拒绝的原因,每个pod和容器的最大CPU请求量限制为1核,但为容器申请2核.



## 应用限制默认值

测试资源没有指定requests和limits情况下分配到的默认值:

```sh
[root@server4-master ~]# kubectl create -f pod-drop.yaml
pod/pod-drop created
[root@server4-master ~]# kubectl describe po pod-drop
    Limits:
      cpu:     200m
      memory:  100Mi
    Requests:
      cpu:        100m
      memory:     10Mi
```

容器的requests和Limits与LimitRange对象中设置的一致.

可以在另一个命名空间指定不同的LimitRange,来适应不同的使用场景.

