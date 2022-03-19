# Secret资源

## 定义和用途

Secret的结构和使用方法与ConfigMap相同,也是键值对的映射,主要用来保存敏感信息,比如账号和证书.

k8s通过将Secret分发到pod所需节点来保障安全性,Secret只会储存在节点的内存中,这样就无法被窃取.



## 默认令牌

K8s安装完毕后会生产一个以default-token开头,默认挂载至所有容器的Secret:

```sh
[root@server4-master ~]# kubectl get secrets 
NAME                  TYPE                                  DATA   AGE
default-token-vqjsw   kubernetes.io/service-account-token   3      132d
[root@server4-master ~]# kubectl describe secrets default-token-vqjsw
Name:         default-token-vqjsw
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: default
              kubernetes.io/service-account.uid: ae0b90b8-1323-42a2-990b-a18d4bfb7da8

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1099 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IlZDd
```

默认Secret包含三个条目: ca.crt, namespace和token,包含从pod内部安全访问API服务器所需的全部信息.

可以在po信息的mount栏看到secret卷被挂载的位置:

```sh
[root@server4-master ~]# kubectl describe po dnsutils 
Containers:
  dnsutils:
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-5ghsj (ro)
```

默认挂载行为可以通过设置pod定义中的automountServiceAccountToken字段为false来关闭.



## 创建Secret

用命令行来建立一个名为ngx-https的Secret来保存Nginx所需的私钥和证书:

```sh
[root@server4-master ~]# kubectl create secret generic ngx-https --from-file=https.cert --from-file=https.key --from-file=foo
```

Secret条目的内容都会被以Base64格式编码,而ConfigMap直接以纯文本展示.所以Secret甚至可以用来储存最大1MB的二进制数据.

通过stringData字段设置条目的纯文本值,不会被base64编码:

```sh
[root@server4-master ~]# vi secret-test.yaml
apiVersion: v1
kind: Secret
stringData:
  foo: ymfyc
data:
  https.cert: Ls-3lcc
  https.key: Lsv9elx
[root@server4-master ~]# kubectl create -f secret-test.yaml 
```

注意stringData字段是只写的.



## 使用Secret

通过secret卷将Secret暴露给容器之后,Secret条目的值会被解码为真实形式(纯文本或二进制)写入对应的文件.通过环境变量暴露Secret条目亦是如此.在这两种情况下,应用程序均无须主动解码,可直接读取文件内容或查找环境变量:

```sh
spec:
  containers:
	volumeMounts:
	- name: certs
	  mountPath: /etc/nginx/certs/
	  readOnly: true
  volumes:
    - name: certs
	  secret:
	    secretame: ngx-https
```

与configMap卷相同,secret卷同样支持通过defaultModes属性指定卷中文件的默认权限.

Secret条目也可以暴露为环境变量:

```sh
spec:
  containers:
    env:
    - name: INTERVAL
      valueFrom:
	    secretKeyRef:
		  name: ngx-https
		  key: foo
```

可以创建一个包含Docker镜像仓库证书的Secret,类型为docker-registry,名叫dockerhubsecret.并在拉取镜像时引用:

```sh
[root@k8s-master 2]# kubectl create secret docker-registry dockerhubsecret \
 --docker-username=myusername --docker-password=mypasswd \
 --docker-email=my@email.com
[root@k8s-master 2]# vi dockerhub.yaml
apiVersion: v1
kind: Pod
metadata:
  name: privae-pod
spec:
  imagePullSecrets:
  - name: dockerhubsecret
  containers:
  - image: assassing/av:v1
    name: main
```

