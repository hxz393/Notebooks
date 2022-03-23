# ResourceQuota

## 介绍配额

可以通过ResourceQuota对象来限制命名空间中可用资源总量.

ResourceQuota的接纳控制插件会检查将要创建的pod是否会引起总资源量超出限制.如果超出限制,请求会被拒绝.

资源配额除了限制命名空间中pod和PVC储存最多可以使用的资源总量.同时也可以限制用户允许在该命名空间中创建pod, PVC以及其他API对象的数量.



## 创建配额

创建一个ResourceQuota资源来限制命名空间中所有pod允许使用的CPU和内存总量:

```sh
[root@server4-master ~]# vi quota-1.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: cpu-and-mem
spec:
  hard:
    requests.cpu: 440m
    requests.memory: 200Mi
    limits.cpu: 2
    limits.memory: 500Mi
[root@server4-master ~]# kubectl create -f quota-1.yaml 
resourcequota/cpu-and-mem created
```

创建后命名空间内所有pod合计可申请的CPU数量为440m,限制总量2核,可申请内存大小440MB,限制内存总量500MB.



## 测试配额

新建一个pod测试:

```sh
[root@server4-master ~]# kubectl create -f pod-readonly.yaml 
Error from server (Forbidden): error when creating "pod-readonly.yaml": pods "pod-readonly" is forbidden: failed quota: cpu-and-mem: must specify limits.cpu,limits.memory,requests.cpu,requests.memory
```

API服务器提示pod必须设置limits和requests属性才能建立.设置limits后试着建立:

```sh
[root@server4-master ~]# kubectl create -f pod-requests2.yaml 
Error from server (Forbidden): error when creating "pod-requests2.yaml": pods "requests-pod2" is forbidden: exceeded quota: cpu-and-mem, requested: requests.memory=400Mi, used: requests.memory=10Mi, limited: requests.memory=200Mi
```

这次提示配额不足.查看命名空间default下的配额使用量和限制:

```sh
[root@server4-master ~]# kubectl describe quota -n default
Name:            cpu-and-mem
Namespace:       default
Resource         Used   Hard
--------         ----   ----
limits.cpu       200m   2
limits.memory    100Mi  500Mi
requests.cpu     100m   440m
requests.memory  10Mi   200Mi
```



## 储存配额

ResourceQuota对象同样可以限制命名空间中最多可以声明的持久化储存总量:

```sh
[root@server4-master ~]# vi quota-2.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage
spec:
  hard:
    requests.storage: 500Gi
    ssd.storageclass.storage.k8s.io/requests.storage: 300Gi
    standard.storageclass.storage.k8s.io/requests.storage: 1Ti
[root@server4-master ~]# kubectl create -f quota-2.yaml 
resourcequota/storage created
```

上面设置中,对命名空间内所有可申请的PVC总量限制为500GB,ssd名字的SC总量限制为300GB,standard名字的标准SC总量限制为1TB.



## 限制对象数量

可以限制单个命名空间中pod, RC, SRV等对象的个数:

```sh
[root@server4-master ~]# vi quota-3.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: objects
spec:
  hard:
    pods: 10
    services: 5
    services.nodeports: 2                   
[root@server4-master ~]# kubectl create -f quota-3.yaml 
resourcequota/objects created
```

上面声明限制了命名空间中可运行最多10个pod,5个Service.其中最多两个NodePort类型的Service.



## 分配配额

在ResourceQuota通过sepc.scopes可以限制配额作用范围,有四种范围:

- BestEffort和NotBestEffort范围决定配额作用于QoS的最低优先级与其他优先级;

- Terminating和NotTerminating作用于配置了activeDeadlineSeconds或没配置的pod.

BestEffort只允许限制pod个数,其他范围可以限制所有资源.

```sh
[root@server4-master ~]# vi quota-4.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: besteffort-pods
spec:
  scopes:
  - BestEffort
  - NotTerminating
  hard:
    pods: 10
[root@server4-master ~]# kubectl create -f quota-4.yaml 
resourcequota/besteffort-pods created
```

这个quota只会应用于最低优先级,以及没有设置有效期的pod上,这样的pod最多只允许存在10个.

