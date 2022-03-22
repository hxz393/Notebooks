# QoS等级

## 定义

假如节点中两个pod已经完全消耗了系统资源,当pod申请更多资源时势必要有一个pod被终止.可以通过QoS等级决定优先级低的pod做退让.优先级从低到高有三种(BestEffort, Burstable, Guaranteed).

QoS等级来源于pod所包含容器的资源requests和limits的配置.



## BestEffort等级

分配给没有设置任何requests和limits的pod,它们可能分配不到任何CPU资源,同时在需要为其他pod释放内存时,这些容器会被第一批杀死.

不过因为没有配置内存limits,在资源充足时可以分配任意多内存.



## Guaranteed等级

分配给那些所有资源request和limits相等的pod.由以下几个条件决定:

- CPU和内存都要设置requests和limits.
- 每个容器都需要设置资源量.
- requests和limits必须相等.

可以简单地只指定limits字段,那么默认requests与limits相同.不过这些pod容器无法消耗高于limit的资源.



## Burstable等级

其余所有pod都是这个等级,包括requests和limits不同的单容器pod,只定义了requests或limits的pod,多个容器没有完全指定requests或limits的pod.

可以通过describe命令查看pod中QoS Class字段来获得QoS等级:

```sh
[root@server4-master ~]# kubectl describe po pod-drop
QoS Class:                   BestEffort
```



## 处理相同QoS等级容器

在QoS等级相同情况下,每个运行中的进程都有一个OOM(OutOfMemory)分数的值,当需要释放内存时,分数最高的进程将被杀死.

OOM分数由进程已消耗内存百分比和容器内存申请量固定的OOM分数调节因子计算.

例如pod A使用了1G内存,pod B使用了2G内存,但pod A的限制是2G,pod B的限制是5G,则依据内存占比pod A大于pod B会被优先终止.