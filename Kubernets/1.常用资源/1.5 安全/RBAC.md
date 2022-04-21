# RBAC

## 定义

RBAC授权插件将用户角色作为决定用户能否执行操作的关键因素.主体和一个或多个角色相关联,每个角色被允许在特定的资源上执行特定的动词.

RBAC授权规则是通过四种资源进行配置,可以分为两组:

- Role(角色)和ClusterRole(集群角色),它们指定了在资源上可以执行哪些动词.

- RoleBinding(角色绑定)和ClusterRoleBinding(集群角色绑定),它们将上述角色绑到特定用户,组或SA上.

角色和集群角色与绑定之间的区别在于,角色绑定命名空间资源,集群角色绑定集群级别资源.

RBAC除了可对全部资源类型应用安全权限,还可以应用于特定的资源实例.

 

## HTTP请求动作

通过客户端发送HTTP请求到RESTful API服务器特定URL可以执行相应的动作.其映射关系如下:

| HTTP方法  | 单一资源的动词     | 集合的动词       |
| --------- | ------------------ | ---------------- |
| GET, HEAD | get(以及watch监听) | list(以及watch)  |
| POST      | create             | -                |
| PUT       | update             | -                |
| PATCH     | patch              | -                |
| DELETE    | delete             | deletecollection |

 动词use用于PodSecurityPolicy资源.



## 测试RBAC

通过一个运行kubectl-proxy的镜像,测试在容器中直接请求API服务器:

```sh
[root@server4-master ~]# kubectl create ns foo
namespace/foo created
[root@server4-master ~]# kubectl create ns bar
namespace/bar created
[root@server4-master ~]# kubectl run test --image=luksa/kubectl-proxy -n foo
pod/test created
[root@server4-master ~]# kubectl run test --image=luksa/kubectl-proxy -n bar
pod/test created
[root@server4-master ~]# kubectl exec -it test -n foo -- curl localhost:8001/api/v1/namespaces/foo/services
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "services is forbidden: User \"system:serviceaccount:foo:default\" cannot list resource \"services\" in API group \"\" in the namespace \"foo\"",
  "reason": "Forbidden",
  "details": {
    "kind": "services"
  },
  "code": 403
}
```

响应表示SA不允许列出foo命名空间中的服务,RBAC插件阻挡了请求.



## 创建角色

Role资源定义在指定命名空间内,对指定资源的可用操作.下面配置允许用户获取并列出foo命名空间中的服务:

```sh
[root@server4-master ~]# vi role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: foo
  name: service-reader
rules:
- apiGroups: [""]
  verbs: ["get", "list"]
  resources: ["services"]
```

定义参数说明如下:

- metadata.namespace: 指定了Role所在命名空间.如果没有定义则使用当前命名空间.
- metadata.name: 指定Role的名字.
- rules.apiGroups: 定义apiGroup名,因为service是核心资源,所以没有apiGroup名.可以定义多种规则.
- rules.verbs: 定义获取独立的Service(通过名字),并能使用get列出所有允许的服务.
- rules.resources: 定义这条规则和服务有关,必须使用复数的名字.
- rules.resourceNames: 这里没定义.用来指定服务实例的名称来限制对服务实例的访问.

在foo命名空间中创建这个Role资源:

```sh
[root@server4-master ~]# kubectl create -f role.yaml -n foo
role.rbac.authorization.k8s.io/service-reader created
```

也能通过create role命令来直接创建:

```sh
[root@server4-master ~]# kubectl create role service-reader --verb=get --verb=list --resource=services -n bar
role.rbac.authorization.k8s.io/service-reader created
```

这两个角色允许在pod中分别列出各自命名空间中的服务.



## 绑定角色

创建角色后需要将角色绑定一个到主体,它可以是用户,组或者SA:

```sh
[root@server4-master ~]# kubectl create rolebinding test -n foo --role=service-reader --serviceaccount=foo:default
rolebinding.rbac.authorization.k8s.io/test created
```

上面在foo命名空间建立的RoleBinding资源名为test,它将service-reader角色绑定到命名空间foo中的default SA上.如果需要绑定到用户使用--user参数,如果绑定到组用--group参数.

RoleBinding只能指定一个角色,但角色绑定的主体可以多个.现在测试效果:

```sh
[root@server4-master ~]# kubectl exec -it test -n foo -- curl localhost:8001/api/v1/namespaces/foo/services
{
  "kind": "ServiceList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "1666771"
  },
  "items": []
}
```

在foo命名空间中的pod已经成功获取到foo命名空间中Service的信息,不过现在没有运行Service所以items项为空.

可以修改foo命名空间中的RoleBinding,添加角色绑定到bar命名空间的默认SA:

```sh
[root@server4-master ~]# kubectl edit rolebinding test -n foo
subjects:
- kind: ServiceAccount
  name: default
  namespace: foo
- kind: ServiceAccount
  name: default
  namespace: bar
```

现在可以在bar命名空间的pod中也能列出foo命名空间中的服务:

```sh
[root@server4-master ~]# kubectl exec -it test -n bar -- curl localhost:8001/api/v1/namespaces/foo/services
{
  "kind": "ServiceList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "1667251"
  },
  "items": []
}
```



## 绑定集群角色

当只有Role时,需要引用所有命名空间的资源必须一个个建立角色和角色绑定,而且有些资源不在命名空间中(Node,PV,NS等),常规角色不能对这些资源授权,因此引入ClusterRole.

ClusterRole和ClusterRoleBinding都是集群级资源,允许访问没有命名空间的资源和非资源型URL,或则作为单个命名空间内部绑定的公共角色,从而避免必须在每个命名空间中重新定义相同的角色.

创建一个能列出集群中PV的集群角色pv-reader:

```sh
[root@server4-master ~]# kubectl create clusterrole pv-reader --verb=get,list --resource=persistentvolumes
clusterrole.rbac.authorization.k8s.io/pv-reader created
```

foo命名空间的pod上尝试获取PV列表会提示没权限.用clusterrolebinding来绑定集群角色pv-reader到foo命名空间默认SA:

```sh
[root@server4-master ~]# kubectl create clusterrolebinding pv-test --clusterrole=pv-reader --serviceaccount=foo:default
clusterrolebinding.rbac.authorization.k8s.io/pv-test created
```

这次foo命名空间的pod便能成功列出PV资源列表了:

```sh
[root@server4-master ~]# kubectl exec -it test -n foo -- curl localhost:8001/api/v1/persistentvolumes
{
  "kind": "PersistentVolumeList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "1671344"
  },
  "items": [
    {
      "metadata": {
        "name": "mongodb-pv",
```



## 访问非资源型URL

要访问API服务器对外暴露非资源型的URL也需要显式地授权,否则API服务器会拒绝客户端请求.通常可以通过system:discovery这个集群角色和相同命名地集群角色绑定来完成.可以查看一下角色信息:

```sh
[root@server4-master ~]# kubectl get clusterrole system:discovery -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2021-11-03T05:36:25Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:discovery
  resourceVersion: "85"
  uid: e1ff33b5-dc26-422e-8082-5def977e2e2a
rules:
- nonResourceURLs:
  - /api
  - /api/*
  - /apis
  - /apis/*
  - /healthz
  - /livez
  - /openapi
  - /openapi/*
  - /readyz
  - /version
  - /version/
  verbs:
  - get
```

其中rules中定义的是nonResourceURLs而不是资源字段,方法只有GET请求可用.再查看角色绑定的详细信息:

```sh
[root@server4-master ~]# kubectl get clusterrolebinding system:discovery -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2021-11-03T05:36:25Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:discovery
  resourceVersion: "149"
  uid: 98c8ddc8-e6eb-4b9f-8af2-c37e4804c0b3
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:discovery
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:authenticated
```

它绑定了两个组system:authenticated和system:authenticated,这使得所有人都可以访问ClusterRole中的URL.

组位于身份认证插件的域中,API服务器接收到一个请求时,会调用身份认证插件获取用户属组,之后授权中使用这些组信息.

在本地和pod内试着访问/api路径来测试:

```sh
[root@server4-master ~]# kubectl exec -it test -n foo -- curl localhost:8001/api -k
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.2.204:6443"
    }
  ]
}
```



## 集群角色配合角色绑定

ClusterRole不是必须和ClusterRoleBinding捆绑使用,也能和RoleBinding捆绑.先查看一个叫view的默认ClusterRole:

```sh
[root@server4-master ~]# kubectl get clusterrole view -o yaml
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.authorization.k8s.io/aggregate-to-view: "true"
```

可以看到这个角色有很多规则,有一些是存在命名空间的资源比如CM,PVC等,有部分是无命名空间资源,它能定义的权限完全取决于绑定方式:

- 如果创建ClusterRoleBinding来绑定view,在绑定中列出的主体可以在所有命名空间中查看指定资源.
- 如果创建RoleBinding来绑定view,那么只能查看RoleBinding命名空间中的资源.

下面先使用ClusterRoleBinding把ClusterRole view绑定到命名空间foo的默认SA上:

```sh
[root@server4-master ~]# kubectl create clusterrolebinding view-test --clusterrole=view --serviceaccount=foo:default
clusterrolebinding.rbac.authorization.k8s.io/view-test created
[root@server4-master ~]# kubectl exec -it test -n foo -- curl localhost:8001/api/v1/namespaces/bar/services
{
  "kind": "ServiceList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "1754935"
  },
  "items": []
}
```

现在foo命名空间中的pod可以获取集群中所有其他命名空间的Service.

删除集群绑定后换成角色绑定:

```sh
[root@server4-master ~]# kubectl delete clusterrolebinding view-test
clusterrolebinding.rbac.authorization.k8s.io "view-test" deleted
[root@server4-master ~]# kubectl create rolebinding view-test --clusterrole=view --serviceaccount=foo:default -n foo
rolebinding.rbac.authorization.k8s.io/view-test created
[root@server4-master ~]# kubectl exec -it test -n foo -- curl localhost:8001/api/v1/namespaces/bar/services
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "services is forbidden: User \"system:serviceaccount:foo:default\" cannot list resource \"services\" in API group \"\" in the namespace \"bar\"",
  "reason": "Forbidden",
  "details": {
    "kind": "services"
  },
  "code": 403
}
```

这次返回403拒绝列出,命名空间foo中的pod只能列出当前命名空间的Service列表了.



## 角色和绑定的组合

下面表格简单列举访问需求和角色与绑定类型之间的搭配:

| **访问的资源**                                         | **角色类型** | **绑定类型** |
| ------------------------------------------------------ | ------------ | ------------ |
| 集群级别资源(Nodes, PV...)                             | 集群         | 集群         |
| 非资源型URL(/api, /healthz...)                         | 集群         | 集群         |
| 在任何命名空间中的资源                                 | 集群         | 集群         |
| 在具体命名空间中的资源(可在多个命名空间中重用这个角色) | 集群         | 角色         |
| 在具体命名空间中的资源(角色需在每个命名空间中都定义)   | 角色         | 角色         |



## 自带集群角色

K8s提供了一组默认的CR和CRB,每次API服务器启动都会更新它们,这保证了误删或k8s版本更新后,所有的默认角色和绑定都会被重新创建:

- view角色

  允许读取一个命名空间的大多数资源,除了Role, RoleBinding和Secret.

- edit角色

  允许修改一个命名空间中的资源,同时允许读取和修改Secret,但不允许查看或修改Role和RoleBinding.

- admin角色

  可以读取和修改命名空间中的任何资源,除了ResourceQuota和命名空间资源本身.

  API服务器只允许用户在已经拥有一个角色中列出的所有权限的情况下,创建和更新这个角色.

- cluster-admin角色

  拥有集群完全控制权,包括命名空间的ResourceQuota和命名空间资源本身.

- 其他角色

  默认角色以system:为前缀,这些角色用于各种组件中.
