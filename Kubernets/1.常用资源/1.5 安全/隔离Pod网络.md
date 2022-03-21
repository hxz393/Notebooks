# 隔离Pod网络

## 定义

有时需要通过限制pod与pod之间的通信,来确保pod之间的网络安全.

是否可以进行这些配置取决于集群中使用的CNI容器网络插件,如果插件支持,可以通过NetworkPolicy资源配置网络隔离.

一个NetworkPolicy会应用在匹配它的标签选择器的pod上,指明这些允许访问这些pod的源地址,或这些pod可以访问的目标地址.这些分别由ingress和egress规则指定.这两种规则都可以匹配由标签选择器选出的pod,或者一个命名空间中的所有pod,或者通过无类别域间路由(CIDR, Classless Inter-Domain Routing)指定IP地址段.



## 同命名空间网络隔离

默认情况下,某一命名空间中的pod可以被任意来源访问.

可以在特定命名空间创建一个NetworkPolicy,阻止任何客户端访问该命名空间的pod:

```sh
[root@k8s-master 5]# vi np-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector:
```

空的标签选择器匹配命名空间中所有的pod.



## 同命名空间部分访问

也可以指定在同命名空间中允许互相访问的pod,没定义的pod不许与互联:

```sh
[root@k8s-master 5]# vi np-postgres.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-netpolicy
spec:
  podSelector:
    matchlabels:
      app: database
  ingress:
  - from:
    - podSelector:
      matchLabels:
        app: webserver
    ports:
    - port: 5432
```

上面配置了允许标签为app=webserver的pod访问app=database的pod的5432端口.



## 不同命名空间网络隔离

假设有多个命名空间分属于不同用户,在NS中有标签tenant:manning来标记属于哪个用户,只允许同一标签的pod,也就是同一用户的NS中pod可以互相访问,不符合标签的用户禁止访问这个pod:

```sh
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: np-shopping
spec:
  podSelector:
    matchlabels:
      app: shopping
  ingress:
  - from:
    - namespaceSelector:
      matchLabels:
        tenant: manning
    ports:
    - port: 80
```

如果需要别的用户访问可以创建一个新的NP资源,或者在上面配置增加一条入向规则.

通常在多租户K8s集群中,租户不能为他们的命名空间添加标签,否则可以规避基于namespaceSelector的入向规则.



## 使用CIDR隔离网络

可以通过CIDR表示法指定一个IP段,例如允许129.168.1.0/24段的客户端访问shopping的pod:

```sh
  ingress:
  - from:
    - ipBlock:
        cidr: 192.168.1.0/24
```



## 限制Pod对外访问

也可以通过egress来设置出向规则,限制pod的对外流量访问流量:

```sh
spec:
  podSelector:
    matchlabels:
      app: shopping
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
```

这个策略应用于包含app=shopping标签的pod,它们只能与有同样标签的pod通信.除此之外不能访问任何地址,不管是pod还是IP,无论集群内还是集群外.

