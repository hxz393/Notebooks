# PodSecurityPolicy

下面简称PSP.PSP将在k8s 1.25版本中移除,后续可能使用Kyverno或OPA(Open Policy Agent).

PSP默认为关闭状态,需要修改API服务器配置/etc/kubernetes/manifests/kube-apiserver.yaml中,添加 `- --enable-admission-plugins=NodeRestriction,PodSecurityPolicy` 来启用PSP控制器

## 定义

PSP是一种集群级别(无命名空间)资源,它定义了用户能否在pod中使用各种安全相关的特性.维护PSP配置工作由集成在API服务器中的PodSecurityPolicy准入控制插件完成.

当向API服务器发送建立pod资源时,PSP准入控制插件会将这个pod与已经配置的PSP进行校验.如果符合安全策略通过则存入etcd,否则会立即被拒绝.这个插件也会根据安全策略中配置的默认值对pod进行修改.

 PSP资源主要定义以下规则:

- 是否允许pod使用宿主节点的PID,IPC,网络命名空间.

- pod允许绑定的宿主节点端口.

- 容器运行时允许使用的用户ID.

- 是否允许拥有特权模式容器的pod.

- 允许添加哪些内核功能,默认添加哪些内核功能,总是禁用哪些内核功能.

- 允许容器使用哪些SELinux选项.

- 容器是否允许使用可写的根文件系统.

- 允许容器在哪些文件系统组下运行.

- 允许pod使用哪些类型的储存卷.

PSP存在的主要问题是:

- 授权模型存在缺陷.

- 功能易开难关.

- API接口缺乏一致性及扩展性,如MustRunAsNonRoot,AllowPrivilegeEscalation此类配置.

- 无法处理动态注入的side-car(如knative).

- 在CI/CD场景难以落地.



## 建立PSP

下面定义一个PSP样例,作用是阻止pod使用宿主节点命名空间,不允许特权模式,定义能绑定节点的端口:

```sh
[root@server4-master ~]# vi psp-default.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: default
spec:
  hostIPC: false
  hostPID: false
  hostNetwork: false
  hostPorts:
  - min: 10000
    max: 11000
  - min: 13000
    max: 14000
  privileged: false
  readOnlyRootFilesystem: true
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  volumes:
  - '*'
[root@server4-master ~]# kubectl create -f psp-default.yaml 
Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
podsecuritypolicy.policy/default created
```

创建成功后,限制立即生效.



## 用户和组限制

当在runAsUser或类似字段使用RunAsAny规则时,对容器运行时可使用的用户和用户组没有任何限制,如果规则改为MustRunAs可以限制容器能使用和用户和用户组

例如限制运行用户ID的范围在2-10或20-30范围内,限制组ID在2-30之间:

```sh
  runAsUser:
    rule: MustRunAs
    ranges:
    - min: 2
      max: 10
    - min: 20
      max: 30
  fsGroup:
    rule: MustRunAs
SYS_MODULE    ranges:
    - min: 2
      max: 30
```

所有超过ID限制范围的创建请求都会被拒绝.修改策略对已经存在的pod无效,PSP资源仅作用于创建和新升级pod.

假如镜像指定了运行用户ID,PSP也使用了限制运行用户策略,最终只要镜像运行的用户ID在PSP限定范围内,实际运行用户ID以PSP限制为准.



## 配置内核功能

在PSP中分别配置spec下的allowedCapabilities, defaultAddCapabilities, requiredDropCapabilities三个字段来实现允许容器添加内核功能,为容器自动添加功能,禁止容器使用的内容功能用途.

例如配置容器允许添加SYS_TIME功能,自动添加CHOWN功能,禁止SYS_ADMIN和SYS_MODULE功能:

```sh
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: default
spec:
  allowedCapabilities:
  - SYS_TIME
  defaultAddCapabilities:
  - CHOWN
  requiredDropCapabilities:
  - SYS_ADMIN
  - SYS_MODULE
```

没有配置的内核功能,默认禁止添加到pod中.

如果pod配置文件和PSP中都定义了同样的字段,以PSP为准.试图在pod定义加入禁止的功能会被API服务器拒绝创建pod.



## 限制储存卷类型

使用"*"代表所有类型,也可以手动指定类型:

```sh
  volumes:
  - emptyDir
  - configMap
```

其他没有指定的储存卷类型不可以被使用.

如果有多个PSP,pod实际可以使用的储存卷类型是所有PSP中volume列表的并集.



## 对用户分配PSP

针对不同用户或组分配不同的PSP策略是通过RBAC机制实现的.实现方式是:先创建需要的PSP资源,然后创建ClusterRole资源并通过名称将它们指向不同的策略,以此使PSP资源中的策略对不同用户或组生效.通过ClusterRoleBinding资源将特定的用户或组绑定到ClusterRole上,当PSP访问控制插件需要决定是否接纳一个Pod时,它只会考虑创建pod的用户可以访问到的PSP中的策略.

先创建一个允许privileged权限的PSP:

```sh
[root@server4-master ~]# vi psp-privileged.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: privileged
spec:
  privileged: true
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  volumes:
  - '*'
[root@server4-master ~]# kubectl create -f psp-privileged.yaml
Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
podsecuritypolicy.policy/privileged created
[root@server4-master ~]# kubectl get psp
NAME                      PRIV    CAPS   SELINUX    RUNASUSER          FSGROUP     SUPGROUP    READONLYROOTFS   VOLUMES
default                   false          RunAsAny   RunAsAny           RunAsAny    RunAsAny    true             *
privileged                true           RunAsAny   RunAsAny           RunAsAny    RunAsAny    false            *
```

创建两个ClusterRole,一个使用default策略,一个使用privileged策略.需要使用动词use:

```sh
[root@server4-master ~]# kubectl create clusterrole psp-default --verb=use --resource=podsecuritypolicies --resource-name=default
clusterrole.rbac.authorization.k8s.io/psp-default created
[root@server4-master ~]# kubectl create clusterrole psp-privileged --verb=use --resource=podsecuritypolicies --resource-name=privileged
clusterrole.rbac.authorization.k8s.io/psp-privileged created
```

将默认权限绑定到认证账户组,允许privileged权限的PSP绑定到用户Bob:

```sh
[root@server4-master ~]# kubectl create clusterrolebinding psp-all-users --clusterrole=psp-default --group=system:authenticated
clusterrolebinding.rbac.authorization.k8s.io/psp-all-users created
[root@server4-master ~]# kubectl create clusterrolebinding psp-bob --clusterrole=psp-privileged --user=bob
clusterrolebinding.rbac.authorization.k8s.io/psp-bob created
```

使用kubectl config创建两个新用户:

```sh
[root@server4-master ~]# kubectl config set-credentials alice --username=alice --password=password
User "alice" set.
[root@server4-master ~]# kubectl config set-credentials bob --username=bob --password=password
User "bob" set.
```

使用不同用户创建带特权选项的Pod:

```sh
[root@server4-master ~]# kubectl --user alice create -f pod-settime.yaml 
Error from server (Forbidden): unknown
```

API服务器不允许alice创建特权pod,用bob可以创建成功.