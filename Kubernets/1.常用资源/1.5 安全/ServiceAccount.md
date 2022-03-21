# ServiceAccount

下面简称SA.

## 介绍

每个pod都与一个SA相关联,SA代表了运行在pod中应用程序的身份证明.位于容器内的token文件持有SA的认证token,应用程序使用这个token连接API服务器时,身份认证插件会对SA进行身份认证,并将SA的用户名传回API服务器内部.SA用户名的格式如下: `system:serviceaccount:<namespace>:<service account name>`.

API服务器将这个用户名传给已配置好的授权插件,来决定应用程序所尝试执行的操作是否被允许执行.

SA只不过是一种运行在pod中的应用程序和API服务器身份认证的一种方式,应用程序通过在请求中传递SA token来实现这点.



## 查看SA

SA资源作用在单独的命名空间,每个命名空间会自动创建一个默认的SA.可以通过get命令查看:

```sh
[root@server4-master ~]# kubectl get sa
NAME      SECRETS   AGE
default   1         137d
```

在pod的配置文件中,可以使用指定账户名称的方式将一个SA赋值给一个pod.如果不指定,pod会使用这个命名空间中的默认SA.

SA可以给多个pod关联使用,来控制每个pod可以访问的资源.但是pod不能使用跨命名空间的SA.



## 创建SA

可以使用create命令来创建SA:

```sh
[root@server4-master ~]# kubectl create sa foo
serviceaccount/foo created
[root@server4-master ~]# kubectl describe sa foo
Name:                foo
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   foo-token-68qgz
Tokens:              foo-token-68qgz
Events:              <none>
```

显示SA详细信息中,主要有三个参数:

- Image pull secrets

  镜像拉取秘钥(Image pull secrets)不仅给pod拉取镜像时使用,这个字段的秘钥会自动添加到所有使用这个SA的pod中,以此达到批量添加的操作.

- Mountable secrets

  默认情况下pod可以挂载任何需要的秘钥,通过Mountable secrets配置可以让pod只允许挂载其中的secrets.

  需要配合注解kubernetes.io/enforce-mountable-secrets="true"来限制.

- Tokens

  pod认证时使用的token.

可以进一步查看自定义的token密钥内容:

```sh
[root@server4-master ~]# kubectl describe secrets foo-token-68qgz

Data
====
ca.crt:     1099 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiI
```

其中包括了和默认SA相同的条目:ca证书, 命名空间和token.SA中使用的身份认证token是JWT(JSON Web Token).



## 配置SA

通过配置pod模板中sepc.serviceAccountName字段来分配SA.pod创建之后不能被修改:

```sh
[root@server4-master ~]# vi curl-sa.yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-custom-sa
spec:
  serviceAccountName: foo
  containers:
  - name: main
    image: curlimages/curl
    command: ["sleep", "99999"]
  - name: ambassador
    image: luksa/kubectl-proxy:1.6.2
[root@server4-master ~]# kubectl create -f curl-sa.yaml
pod/curl-custom-sa created
```

查看容器内的token:

```sh
[root@server4-master ~]# kubectl exec -it curl-custom-sa -c main -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
eyJhbGciOiJSUzI1NiIsImtpZCI6IlZDdEV4ZVNxWmNVVlNMS2lqRU5hM2NNVzBRSE54bHZsSzZucERQY3lzblEifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjL
```

对比发现和foo-token-llg77完全相同.测试通过使用自定义SA的token和API服务器进行通信:

```sh
[root@server4-master ~]# kubectl exec -it curl-custom-sa -c main -- curl localhost:8001/api/v1/pods/
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:default:foo\" cannot list resource \"pods\" in API group \"\" at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
}
```

结果提示新建SA没有列出pod列表的权限.SA需要和RBAC授权插件配合使用,否则SA只起到提供镜像拉取秘钥的功能.

